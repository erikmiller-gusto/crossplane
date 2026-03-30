# Provider Sharding

* Owner: Erik Miller (@erik.miller)
* Reviewers: Crossplane Maintainers
* Status: Draft

## Background

Crossplane currently runs a single controller instance per provider. When a
Provider is installed, the package manager creates one ProviderRevision, which
creates one Deployment with one replica (`runtime.go`, default replicas=1).
That single provider pod is responsible for reconciling every managed resource
of every type the provider supports.

The [smaller providers design][design-smaller-providers] addressed a related
scaling dimension — too many CRDs per provider — by splitting large providers
into service-scoped packages. That design also documented the resource
consumption characteristics of upjet-based providers: ~600MB base memory for
the Crossplane provider process, ~300MB per Terraform provider process, and
roughly one CPU core to simultaneously reconcile 10 managed resources
(design-doc-smaller-providers.md, "Compute Resource Impact" section).

This proposal addresses the orthogonal dimension: the number of *resources*
being reconciled by a single provider instance. As the number of managed
resources grows, a single provider pod must maintain informer caches for all
of them and serialize reconciliation through a single process. Today, the
only scaling mechanism is vertical — increasing CPU and memory limits on the
provider Deployment via `DeploymentRuntimeConfig`. Horizontal scaling (running
multiple provider replicas) is not feasible because all replicas would watch
and attempt to reconcile the same resources, leading to conflicts.

This constraint is particularly relevant for upjet-based providers (such as
provider-aws, provider-azure, and OpenTofu/Terraform-based providers), which
spawn external processes (Terraform CLI and Terraform provider binaries) per
reconciliation, amplifying per-resource resource consumption relative to
native providers (design-doc-smaller-providers.md, "Compute Resource Impact"
section).

### Prior Art

- **[design-doc-smaller-providers.md][design-smaller-providers]**: Broke large
  providers into service-scoped packages to reduce CRD count. Documents
  upjet-based provider resource consumption characteristics. Complementary to
  this proposal.
- **[one-pager-crd-scaling.md][one-pager-crd-scaling]**: Analyzed performance
  issues with large numbers of CRDs, including client-side throttling and API
  server resource consumption. This proposal addresses resource count scaling,
  not CRD count scaling.
- **[one-pager-performance-characteristics-of-providers.md][one-pager-perf]**:
  Proposed tooling for measuring provider CPU utilization, memory utilization,
  and time-to-readiness for managed resources.
- **controller-runtime label selectors**: controller-runtime supports
  `cache.Options.DefaultLabelSelector` and per-GVK `cache.Options.ByObject`
  overrides, enabling filtered informer caches. This is the mechanism we
  build on.

## Goals

1. Enable horizontal scaling of providers by distributing managed resources
   across multiple provider instances using shard-based partitioning.
2. Maintain full backward compatibility — existing deployments with no sharding
   configuration continue to work unchanged.
3. Require minimal changes to Crossplane core. The bulk of the mechanism should
   live in crossplane-runtime, where providers already set up their watches.
4. Support gradual adoption — users can shard some resources while leaving
   others on a default (unsharded) provider instance.

## Non-Goals

1. Automatic shard assignment or load-based rebalancing. The initial
   implementation requires explicit user action to assign resources to shards.
2. Sharding of Crossplane core controllers (composition, claim syncing). Only
   provider controllers are sharded.
3. Cross-shard resource management. A shard provider only reconciles resources
   in its shard. However, cross-shard reference *resolution* (reading another
   shard's resources to resolve a reference) should be supported.

## Proposal

### Overview

We introduce a new label, `crossplane.io/shard`, that partitions managed
resources across provider instances. The mechanism has three parts:

1. **Shard label propagation**: The `crossplane.io/shard` label set on a Claim
   or XR is automatically propagated to all composed resources (managed
   resources), alongside existing labels like `crossplane.io/composite`.

2. **Shard-aware providers**: Providers accept a `--shard-id` flag (or
   `SHARD_ID` environment variable). When set, the provider configures its
   controller-runtime cache to watch only resources matching
   `crossplane.io/shard={shardID}`. Non-sharded resource types (ProviderConfig,
   Secrets) are exempted from this filter.

3. **Multiple provider deployments**: `DeploymentRuntimeConfig` gains a
   `shards` field. The ProviderRevision reconciler creates one Deployment per
   shard (plus a default instance for unlabeled resources).

### Shard Label Propagation

The label `crossplane.io/shard` flows through the standard Crossplane ownership
chain:

```
Claim (user sets label) → XR → Composed Resources (MRs)
```

**Claim → XR**: Already works. The claim syncer (`syncer_csa.go:84`,
`syncer_ssa.go:106`) propagates all non-reserved-k8s labels from the Claim to
the XR via `meta.AddLabels(xr, withoutReservedK8sEntries(cm.GetLabels()))`.
The `crossplane.io/shard` label will flow through this path with no code
changes.

**XR → Composed Resources**: Requires a change. Today, `RenderComposedResourceMetadata`
in `composition_render.go` (lines 105-113) propagates only three specific labels:
`crossplane.io/composite`, `crossplane.io/claim-name`, and
`crossplane.io/claim-namespace`. The shard label must be added to this set:

```go
// In composition_render.go, after the claim label block:
if v := xr.GetLabels()[xcrd.LabelKeyShard]; v != "" {
    metaLabels[xcrd.LabelKeyShard] = v
}
```

**Direct XR usage (no Claim)**: Users can set `crossplane.io/shard` directly on
the XR. The XR → Composed Resource propagation works the same way.

### Shard-Aware Providers

The key change lives in crossplane-runtime, not Crossplane core. When a provider
starts with `--shard-id=X`:

1. The controller manager's **reconciliation cache** is configured with a
   default label selector: `crossplane.io/shard=X`. This determines which
   resources trigger reconcile loops.
2. Per-GVK overrides exempt non-sharded types from the selector:
   - ProviderConfig (all types)
   - ProviderConfigUsage
   - StoreConfig
   - Secrets (used for credentials)
   - ConfigMaps
3. A separate **uncached client** (or unfiltered cache) is available for
   cross-resource reference resolution, allowing the provider to read
   resources in other shards without reconciling them. See
   [Cross-Shard Reference Resolution](#cross-shard-reference-resolution).
4. The managed reconciler itself requires no changes — it reconciles whatever
   the informer cache provides.

```go
// Pseudocode for provider main() setup
if shardID != "" {
    mgr, err := ctrl.NewManager(cfg, ctrl.Options{
        Cache: cache.Options{
            DefaultLabelSelector: labels.SelectorFromSet(labels.Set{
                "crossplane.io/shard": shardID,
            }),
            ByObject: map[client.Object]cache.ByObject{
                &v1alpha1.ProviderConfig{}:      {Label: labels.Everything()},
                &v1alpha1.ProviderConfigUsage{}: {Label: labels.Everything()},
                &corev1.Secret{}:                {Label: labels.Everything()},
            },
        },
    })
}
```

### Default Provider Behavior

When sharding is configured, the default provider instance (no `--shard-id`)
must handle resources that have no `crossplane.io/shard` label. There are two
possible approaches:

**Option A: Default watches only unlabeled resources.** The default provider
uses a `!crossplane.io/shard` label selector (label-does-not-exist). This
ensures each resource is reconciled by exactly one provider instance — either
the default or its assigned shard. No dual reconciliation is possible.

**Option B: Default watches everything.** The default provider applies no label
selector. It reconciles all resources, including those with shard labels. This
means sharded resources are reconciled by both the default and the shard
provider, creating dual reconciliation.

**Recommendation: Option A.** The default provider should use a
`!crossplane.io/shard` selector to avoid dual reconciliation. This is
expressible as a `metav1.LabelSelectorRequirement` with operator
`DoesNotExist`:

```go
// Default provider (no --shard-id) watches only unlabeled resources
if shardingEnabled && shardID == "" {
    mgr, err := ctrl.NewManager(cfg, ctrl.Options{
        Cache: cache.Options{
            DefaultLabelSelector: labels.Parse("!crossplane.io/shard"),
            // ... same ByObject overrides as shard providers
        },
    })
}
```

When sharding is NOT configured (no `spec.shards` in `DeploymentRuntimeConfig`),
the single provider instance applies no label selector at all, preserving
today's behavior exactly.

### Multiple Provider Deployments

`DeploymentRuntimeConfig` gains a `shards` field. When configured, the
ProviderRevision reconciler creates one Deployment per shard, plus a default
Deployment for unlabeled resources:

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: sharded-aws
spec:
  shards:
  - id: shard-a
  - id: shard-b
  - id: shard-c
  deploymentTemplate:
    spec:
      replicas: 1
      # ... other deployment settings shared across all shards
```

The ProviderRevision reconciler's Post hook (`runtime_provider.go`) iterates
over `spec.shards` and creates one Deployment per shard. Each shard Deployment
is named `{provider}-{shard-id}` and receives the `SHARD_ID` environment
variable. If the combined name exceeds the Kubernetes 63-character limit, the
provider name portion is truncated and a short hash suffix is appended to
ensure uniqueness (following the same pattern used by ReplicaSet pod naming).
A default Deployment (no `SHARD_ID`) is always created to handle unlabeled
resources.

When no `spec.shards` field is present (or it is empty), behavior is identical
to today: a single Deployment is created. This ensures backward compatibility.

All shard Deployments share the same:
- ServiceAccount (created by the ProviderRevision reconciler)
- RBAC (ClusterRoleBinding from the RBAC binding controller)
- Webhook Service (stateless validation load-balanced across pods)
- Provider image and configuration from `deploymentTemplate`

### Shard Lifecycle Operations

#### Adding a Shard

1. Add the shard to `DeploymentRuntimeConfig.spec.shards`. The ProviderRevision
   reconciler creates a new Deployment for the shard.
2. The new provider instance starts and watches for resources labeled
   `crossplane.io/shard=new-shard`. Initially it finds none.
3. Label existing Claims/XRs with `crossplane.io/shard=new-shard` to move
   resources to the new shard.
4. The shard label propagates to composed resources. The new shard provider
   begins reconciling them; the old provider (default or another shard) stops
   seeing them in its filtered cache.

#### Removing a Shard

Removing a shard requires draining it first. Resources labeled for a removed
shard will have no provider reconciling them.

1. **Drain**: Relabel all Claims/XRs currently assigned to the shard being
   removed. Change `crossplane.io/shard` to another shard or remove it
   entirely (moving resources to the default provider).
2. **Verify**: Confirm the shard provider's reconcile queue is empty (no
   resources match its selector).
3. **Remove**: Remove the shard from `DeploymentRuntimeConfig.spec.shards`.
   The ProviderRevision reconciler deletes the shard Deployment.

**Safety mechanism**: Before removing a shard Deployment, the reconciler
should check whether any managed resources still carry that shard label and
block removal with a status condition if resources remain. This prevents
accidental orphaning of resources.

#### Shard Rebalancing

Moving a resource between shards involves changing the `crossplane.io/shard`
label on the Claim or XR. The label change propagates to composed resources on
the next reconciliation cycle.

**What happens during the transition**:
- The old shard provider's filtered informer stops tracking the resource
  (it no longer matches the label selector). controller-runtime does not
  enqueue a reconcile event for objects that fall out of a filtered cache —
  the informer simply stops watching them.
- Even if a stale reconcile were somehow triggered, the managed reconciler
  (`crossplane-runtime/pkg/reconciler/managed/reconciler.go`) only deletes
  external resources when `meta.WasDeleted(managed)` returns true (line 1165),
  which requires an explicit deletion timestamp on the Kubernetes object.
  A relabeled object has no deletion timestamp; the deletion path is not
  entered.
- The new shard provider's filtered informer picks up the resource on its
  next list/watch sync and begins reconciling it.

This transition is safe because:
1. The Kubernetes object is never deleted — only relabeled. No deletion
   timestamp is set.
2. The managed reconciler's deletion logic is gated on `meta.WasDeleted()`,
   not on informer cache presence.
3. The external cloud resource is untouched.
4. There is a brief window where neither shard is actively reconciling.
   This window is bounded by the new shard provider's informer resync
   interval or the next watch event on the resource.

### Enabling on Existing Systems

Sharding is fully opt-in and backward compatible:

1. **No changes required for existing deployments**. Without the
   `crossplane.io/shard` label on any resources, and without any shard
   provider deployments, behavior is identical to today.

2. **Gradual adoption path**:
   a. Add `spec.shards` to the `DeploymentRuntimeConfig` referenced by the
      Provider. The reconciler creates shard Deployments alongside the
      default Deployment.
   b. Start labeling new Claims/XRs with `crossplane.io/shard=<shard-id>`.
      New resources go to shard providers.
   c. Optionally migrate existing resources by adding the shard label to their
      Claims/XRs.
   d. The default provider continues handling unlabeled resources indefinitely.

3. **Feature flag**: A feature flag `EnableAlphaSharding` gates the shard
   label propagation in Crossplane core. Providers can independently support
   `--shard-id` regardless of the core feature flag, since the flag only
   affects cache configuration.

### Who Assigns Shard Labels

#### Manual Assignment

Users explicitly set `crossplane.io/shard` on their Claims or XRs. This is
the simplest model and gives operators full control over resource distribution.

Unlabeled resources are NOT rejected. They are handled by the default provider
instance. This preserves backward compatibility and avoids a disruptive
migration for existing systems.

#### Policy-Based Assignment (Future)

A validating/mutating webhook or Crossplane admission policy could auto-assign
shards based on rules:
- Round-robin across configured shards.
- Namespace-based mapping (e.g., all resources in namespace `team-a` go to
  `shard-a`).
- Resource-type-based (e.g., all `rds.aws.upbound.io` resources go to
  `shard-rds`).

This would be implemented as a separate component (webhook or Composition
Function) rather than built into Crossplane core, and is out of scope for
the initial design.

#### Automatic Assignment (Future)

A controller could monitor provider load metrics and automatically assign or
rebalance shards. This is significantly more complex and is also out of scope
for the initial design.

### Observability

Operators need visibility into shard distribution and health to make informed
decisions about adding, removing, or rebalancing shards.

**Resource-level**: The `crossplane.io/shard` label on each managed resource is
queryable via `kubectl get <type> -l crossplane.io/shard=<shard-id>`. This
allows operators to count resources per shard and identify imbalances.

**Provider-level**: Each shard Deployment is a standard Kubernetes Deployment
with its own pod metrics (CPU, memory, restart count). Existing monitoring
(Prometheus scrape annotations are already added by the provider runtime in
`runtime_provider.go`) works per-shard pod without changes.

**ProviderRevision status**: The ProviderRevision status conditions should
report per-shard Deployment health. When a shard Deployment is unhealthy
(e.g., crash-looping), the ProviderRevision should surface this as a status
condition identifying the affected shard.

**Shard count metrics**: Providers could emit a Prometheus gauge
`crossplane_managed_resources_total{shard="<id>"}` indicating how many
resources each shard instance is reconciling. This enables alerting on shard
imbalance.

### Known Challenges and Mitigations

#### Cross-Shard Reference Resolution

A shard provider only **reconciles** (watches and manages the lifecycle of)
resources matching its shard label. However, providers also need to **read**
resources for cross-resource reference resolution — for example, an EKS
Cluster MR in shard A may reference a VPC MR in shard B to resolve its VPC ID.

If the provider's entire informer cache is filtered by shard label, it cannot
find the VPC in shard B. To support cross-shard reference resolution, the
shard label selector must apply only to the **reconciliation watches** (the
informers that trigger reconcile loops), not to all resource lookups.

**Implementation approach**: Reference resolution in crossplane-runtime should
use either:
1. An **uncached client** for reference lookups, bypassing the filtered
   informer cache and reading directly from the API server. This is slightly
   more expensive but simple and correct.
2. A **secondary, unfiltered cache** dedicated to reference resolution. This
   avoids per-lookup API server round-trips but adds memory overhead.

Approach (1) is recommended for the initial implementation due to its
simplicity. Reference resolution is infrequent relative to reconciliation (it
runs once to resolve a value, then the resolved value is cached in the resource
spec). The per-lookup API server cost is acceptable.

Note that all composed resources within a single XR hierarchy naturally share a
shard (the label propagates uniformly from XR to all children). Cross-shard
references only arise when referencing resources owned by a *different* XR.

#### Webhooks and Services

Today, each Provider has one webhook Service. The Service selector uses
`pkg.crossplane.io/revision` and the package type label (`runtime.go:332-349`).
Shard Deployments must carry the same pod labels so that the Service routes
traffic to all shard pods. Since webhook validation/mutation logic is stateless
(it validates the resource spec, not reconciliation state), load balancing
across shard pods is safe. The ProviderRevision reconciler must ensure that
shard Deployment pod templates include these standard labels.

#### RBAC

All shard Deployments share the same ServiceAccount created by the
ProviderRevision reconciler. The existing ClusterRoleBinding grants that
ServiceAccount access to all relevant CRDs. No RBAC changes are needed.

#### Composition Functions

Composition functions run in Crossplane core's XR reconciler, not in the
provider. Functions produce desired resource state; the XR reconciler applies
it. The shard label is added by `RenderComposedResourceMetadata` *after* the
function pipeline runs. Functions do not need to be shard-aware.

#### Garbage Collection

Owner-reference-based Kubernetes GC ties composed resources to the XR, not to
the provider. If a shard provider is temporarily down, the Kubernetes objects
remain and the XR controller may report composed resources as not-ready. This
is identical to today's behavior when a single provider is down. No new GC
concerns arise.

#### Status Aggregation

Each shard provider independently updates the `.status` of the managed
resources it reconciles. The XR controller reads status from all composed
resources (regardless of which shard provider updated them) and aggregates
readiness. No cross-shard coordination is needed.

#### ProviderConfigUsage

Each managed resource creates a `ProviderConfigUsage` to track its reference
to a `ProviderConfig`. These are cluster-scoped and exempted from the shard
label selector (so the ProviderConfig finalizer controller can see all usages).
This means every shard provider will list all ProviderConfigUsages, not just
those for its shard. This is acceptable because ProviderConfigUsage
reconciliation is lightweight (it only maintains a finalizer on the referenced
ProviderConfig). If the volume of ProviderConfigUsages becomes a concern at
extreme scale, a future optimization could add the shard label to
ProviderConfigUsages and have a dedicated, unsharded controller handle the
ProviderConfig finalizer logic.

#### ProviderConfig Sharing

ProviderConfigs are not sharded. All shard providers need access to the same
ProviderConfigs. The per-GVK cache override (`cache.Options.ByObject`) exempts
ProviderConfig types from the shard label selector, making them visible to all
shard instances.

## Implementation Scope

### crossplane core

- Define `LabelKeyShard = "crossplane.io/shard"` in `internal/xcrd/schemas.go`.
- Add shard label propagation in `RenderComposedResourceMetadata`
  (`composition_render.go`).
- Add `EnableAlphaSharding` feature flag in `internal/features/features.go`.
- Extend `DeploymentRuntimeConfig` with `spec.shards` field
  (`apis/pkg/v1beta1/deployment_runtime_config_types.go`).
- Update ProviderRevision reconciler to create N+1 Deployments
  (N shards + 1 default) in `runtime_provider.go`.
- Add shard health status to ProviderRevision status conditions.
- Add safety checks: block shard removal if labeled resources still exist.

### crossplane-runtime

- Add `--shard-id` flag to provider CLI helpers.
- When shard ID is set, configure `cache.Options.DefaultLabelSelector`.
- Add `cache.Options.ByObject` overrides for ProviderConfig, ProviderConfigUsage,
  StoreConfig, Secrets, ConfigMaps.
- Ensure reference resolution uses an uncached client to support cross-shard
  lookups.

### User responsibility

- Configure `DeploymentRuntimeConfig.spec.shards` for providers they want to
  shard.
- Label Claims/XRs with `crossplane.io/shard`.

## Future Work

- Webhook or admission policy for automatic shard assignment.
- Shard load monitoring and rebalancing tooling.
- Per-shard `deploymentTemplate` overrides (e.g., different resource limits
  per shard).

## Alternatives Considered

### Multiple Replicas with Leader Election per Resource

Run N replicas of the same provider and use leader election or consistent
hashing to assign individual resources to specific replicas. This avoids
user-facing shard labels but adds significant complexity:
- Requires a coordination mechanism (etcd, configmap-based election per
  resource, or consistent hashing).
- controller-runtime's built-in leader election is all-or-nothing (one leader
  reconciles everything), not per-resource.
- Consistent hashing requires all replicas to agree on the hash ring, adding
  a distributed systems coordination problem.

We rejected this because label-based sharding is simpler, uses native
Kubernetes mechanisms (label selectors on informer caches), and gives
operators explicit control.

### Namespace-Based Partitioning

Assign resources to namespaces and have each provider instance watch a specific
namespace. This is a common pattern in multi-tenant Kubernetes controllers.

We rejected this because:
- Crossplane managed resources are cluster-scoped, not namespaced.
- Composite resources may be cluster-scoped or namespaced.
- Forcing namespace-based partitioning would require architectural changes to
  resource scoping.

### Virtual Clusters (vcluster)

Run each shard as a separate virtual cluster, each with its own Crossplane
installation. This provides strong isolation but:
- Dramatically increases operational complexity.
- Makes cross-shard visibility and management difficult.
- Is overkill for the stated goal of horizontal provider scaling.

## Critical Files

| File | Change |
|------|--------|
| `internal/xcrd/schemas.go` | Add `LabelKeyShard` constant |
| `internal/controller/apiextensions/composite/composition_render.go` | Propagate shard label to composed resources |
| `internal/features/features.go` | Add `EnableAlphaSharding` feature flag |
| `apis/pkg/v1beta1/deployment_runtime_config_types.go` | Add `Shards` field |
| `internal/controller/pkg/runtime/runtime_provider.go` | Multi-deployment creation per shard |

Changes in **crossplane-runtime** (separate repo):

| File | Change |
|------|--------|
| Provider CLI setup helpers | Add `--shard-id` flag |
| Manager/cache configuration | Apply label selector when shard ID is set |
| ByObject exemptions | Exempt ProviderConfig and related types |

[design-smaller-providers]: design-doc-smaller-providers.md
[one-pager-crd-scaling]: one-pager-crd-scaling.md
[one-pager-perf]: one-pager-performance-characteristics-of-providers.md
