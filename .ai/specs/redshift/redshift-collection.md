# Amazon Redshift Data Collection Specification

Status: **Design — not yet implemented**
Target stack version: v3.14.8 (`data-collection/utils/version.json`)
Verified against: boto3 1.43.102

## Overview

Org-wide collection of Amazon Redshift configuration and operational data to power a single
pane of glass covering:

1. **Data share relationships** — producer/consumer topology across accounts and regions
2. **Zero-ETL integrations** — inbound source→Redshift pipelines and their sync status
3. **Maintenance tracks and patch levels** — current version/revision vs. available targets
4. **Resiliency posture** — Multi-AZ, encryption, snapshot retention, cross-region copy, logging
5. **Advisor Impact** — Redshift Advisor recommendations ranked by impact

Both **provisioned clusters** and **Redshift Serverless** (namespaces/workgroups) are in scope
for v1.

### Out of scope for v1

Items below are recorded so they are not lost. Each states **why** it is excluded and **what
would have to change** to bring it in, so a later phase can pick it up without re-deriving the
analysis.

**Per-query performance data** (slow queries, execution plans, `SYS_QUERY_HISTORY`,
`STL_ALERT_EVENT_LOG`).

*Why excluded — a boundary of this framework, not a design preference.* The Redshift control
plane exposes no per-query API. Reading it requires the Redshift Data API (`redshift-data`)
executing SQL against system tables, which needs in-database credentials
(`SecretArn`/`DbUser`), network reachability to each cluster, per-database iteration, and an
async three-call polling pattern (`ExecuteStatement` → `DescribeStatement` →
`GetStatementResult`). Every collector in this repo authenticates with IAM alone and calls
`describe_*`/`list_*` over public endpoints. In-database authentication and VPC reachability
sit outside that security model, so this is not a gap in the spec — it is a capability the
current collection model cannot express.

*What would have to change to add it.* A secrets-distribution mechanism (per-cluster
`SecretArn` discovery plus linked-account read access), VPC-attached Lambdas or
Redshift-managed VPC endpoints for reachability, an async execution pattern no existing module
uses, and a security review of running arbitrary SQL in customer databases. That is a separate
module with its own topology and opt-in, not an extension of this one — note that
`MODULE_GUIDELINES.md` §2 would justify a `-<facet>` suffix in that case, precisely because
the collection topology differs. **Advisor Impact (§5) is the intentional stand-in**: it
surfaces query-performance findings through an IAM-only control-plane call, which is why it
is in v1 while the underlying query data is not.

**Cross-region snapshot copy *verification*.** v1 answers whether cross-region copy is
configured, to which destination, and with what retention — but not whether copies are
actually succeeding. No per-cluster last-successful-copy timestamp exists on the cluster
object. Confirming copies land requires `describe_cluster_snapshots` in the **destination**
region, filtered to cross-region-copied snapshots, which is the API excluded from v1 for being
unbounded (see Constraints §3). Deferred by decision, not oversight; revisit as a scoped
re-add with a `StartTime` window if copy-failure detection becomes a requirement.

## Module Placement

Placement follows `data-collection/MODULE_GUIDELINES.md`. The guidelines forbid putting
everything in one Redshift module — classification is by **data kind** and **collection
topology**, not by consuming dashboard.

**Duplicate-collection check (MODULE_GUIDELINES §1, "Before adding any collection").** No
existing module collects any Redshift data: `grep -i redshift data-collection/deploy/*.yaml`
returns nothing. Redshift is absent from `module-pricing` (no `pricing_redshift_data`), from
`module-reference` (no Redshift engine/version catalog), and from the `module-inventory`
`AwsObjects` fan-out. So none of this is derivable from existing tables, and every table below
is new collection rather than a view over something already present.

| Data | Module | Guideline |
|---|---|---|
| `describe_clusters`, `list_namespaces`, `list_workgroups` | `module-inventory` (new `AwsObjects` entries) | §1 — `describe_*` of live resources per linked account; "Don't create a new module" |
| Data shares, zero-ETL, DB revisions, snapshot schedules/copy grants, logging status, Advisor recommendations, usage limits, reserved nodes | **new `module-redshift`** | §1 service-specific; §2 one generic module per service |
| `describe_cluster_tracks`, `describe_cluster_versions`, serverless `list_tracks` | `module-reference` | §1 — catalogs not tied to a customer resource; data-collection account only |
| Patch drift, share topology, resiliency scoring, Advisor rollups | Athena `NamedQuery` views | §1 — "Collect raw data once; compute in Athena" |

**No `-<facet>` suffix.** §2 permits a suffix only when collection topology differs. All
service-specific Redshift data is plain LINKED, so it consolidates into one `module-redshift`.
Do not create `module-redshift-datashares`, `module-redshift-maintenance`, etc.

**Known trade-off:** the cluster anchor table lives in `module-inventory`, so views join
across two modules. Accepted deliberately — the inventory fan-out already provides the
per-account/per-region loop, tag enrichment, and S3 upload uniformly.

## Naming

Per §3, one token at every touch-point:

| Touch-point | Value |
|---|---|
| Template file | `data-collection/deploy/module-redshift.yaml` |
| `CFDataName` default | `redshift` |
| Deploy-stack parameter | `IncludeRedshiftModule` |
| Deploy-stack condition | `DeployRedshiftModule` |
| Nested stack logical id | `RedshiftModule` |
| Lambda role | `${ResourcePrefix}redshift-LambdaRole` |
| Crawler | `${ResourcePrefix}redshift-Crawler` |

`module-rds-usage` is the counter-example to avoid (three different tokens for one module).

### Descriptions

Per §4 — name the service and data category, never a specific API. Avoid `health`, `metrics`,
`usage`, `utilization`.

- Module template: `Description: Retrieves Redshift operational and configuration data across the AWS organization`
- Deploy-stack parameter: `Description: Collects Redshift operational details (e.g. data shares, integrations, Advisor recommendations) from your accounts`

## API → Table Mapping

### module-inventory additions

`AwsObjects` gains three entries. Provisioned clusters use the generic `paginated_scan`;
serverless needs the separate `redshift-serverless` client.

| `AwsObjects` entry | `path` | API | `obj_name` | Table |
|---|---|---|---|---|
| `RedshiftClusters` | `redshift-clusters` | `redshift:describe_clusters` | `Clusters[*]` | `inventory_redshift_clusters_data` |
| `RedshiftServerlessNamespaces` | `redshift-serverless-namespaces` | `redshift-serverless:list_namespaces` | `namespaces[*]` | `inventory_redshift_serverless_namespaces_data` |
| `RedshiftServerlessWorkgroups` | `redshift-serverless-workgroups` | `redshift-serverless:list_workgroups` | `workgroups[*]` | `inventory_redshift_serverless_workgroups_data` |

### module-redshift tables

All LINKED, all `redshift`/`redshift-serverless` read-only calls.

| Table | API | Paginates | Notes |
|---|---|---|---|
| `redshift_datashares_data` | `describe_data_shares` | yes | No required args — returns all shares in account |
| `redshift_datashares_producer_data` | `describe_data_shares_for_producer` | yes | Producer-side view |
| `redshift_datashares_consumer_data` | `describe_data_shares_for_consumer` | yes | Consumer-side view |
| `redshift_inbound_integrations_data` | `describe_inbound_integrations` | yes | **Primary zero-ETL table** (source→Redshift) |
| `redshift_integrations_data` | `describe_integrations` | yes | Outbound/managed view; adds `IntegrationName`, `KMSKeyId`, `Tags` |
| `redshift_cluster_db_revisions_data` | `describe_cluster_db_revisions` | yes | `ClusterIdentifier` optional → one call per region |
| `redshift_snapshot_schedules_data` | `describe_snapshot_schedules` | yes | `ScheduleDefinitions`, `NextInvocations`, `AssociatedClusterCount` |
| `redshift_snapshot_copy_grants_data` | `describe_snapshot_copy_grants` | yes | Cross-region copy KMS grants |
| `redshift_logging_status_data` | `describe_logging_status` | **no** | **N+1** — requires `ClusterIdentifier` |
| `redshift_recommendations_data` | `list_recommendations` | yes | **Advisor Impact.** `ClusterIdentifier`/`NamespaceArn` both optional → one call per region |
| `redshift_usage_limits_data` | `describe_usage_limits` | yes | `FeatureType`, `LimitType`, `Amount`, `BreachAction` |
| `redshift_reserved_nodes_data` | `describe_reserved_nodes` | yes | Commitment coverage |
| `redshift_endpoint_access_data` | `describe_endpoint_access` | yes | Cross-VPC/cross-account endpoints |
| `redshift_serverless_recovery_points_data` | `redshift-serverless:list_recovery_points` | yes | Serverless resiliency |
| `redshift_serverless_snapshot_copy_configs_data` | `redshift-serverless:list_snapshot_copy_configurations` | yes | Serverless cross-region copy |

Deliberately **excluded**: `describe_cluster_snapshots` (unbounded — see Constraints),
`describe_cluster_parameters` (per-parameter-group N+1, high cardinality, low dashboard value),
`describe_events` (90-day rolling window, better served by `module-health-events`).

### module-reference additions

Region-scoped catalogs, collected in the data-collection account only.

| Table | API | Purpose |
|---|---|---|
| `redshift_cluster_tracks` | `describe_cluster_tracks` | `MaintenanceTrackName`, `DatabaseVersion`, `UpdateTargets[]` — the patch-drift denominator |
| `redshift_cluster_versions` | `describe_cluster_versions` | Version → parameter-group-family catalog |
| `redshift_serverless_tracks` | `redshift-serverless:list_tracks` | `trackName`, `workgroupVersion`, `updateTargets[]` |

## Field Coverage by Capability

Field names below are the API's; Glue columns are lowercased by convention.

### Data share relationships

`DataShare`: `DataShareArn`, `ProducerArn`, `AllowPubliclyAccessibleConsumers`, `ManagedBy`,
`DataShareType` (enum: `INTERNAL`), `DataShareAssociations[]`.

Graph edges live in `DataShareAssociations[]`:
`ConsumerIdentifier`, `Status`, `ConsumerRegion`, `CreatedDate`, `StatusChangeDate`,
`ProducerAllowedWrites`, `ConsumerAcceptedWrites`.

Status enums differ by perspective:
- `DataShareStatus`: `ACTIVE`, `PENDING_AUTHORIZATION`, `AUTHORIZED`, `DEAUTHORIZED`, `REJECTED`, `AVAILABLE`
- producer: `ACTIVE`, `AUTHORIZED`, `PENDING_AUTHORIZATION`, `DEAUTHORIZED`, `REJECTED`
- consumer: `ACTIVE`, `AVAILABLE`

Because collection runs in every linked account, each share is observed twice (producer and
consumer side). That redundancy is intentional: it lets the topology view reconcile a true
org-wide edge list and flag one-sided or orphaned shares.

`ConsumerIdentifier` is sometimes an account ID and sometimes a namespace ARN — normalize in
the view.

### Zero-ETL integrations

`InboundIntegration`: `IntegrationArn`, `SourceArn`, `TargetArn`, `Status`, `Errors`, `CreateTime`.
`Integration` adds: `IntegrationName`, `Description`, `KMSKeyId`, `AdditionalEncryptionContext`, `Tags`.

`ZeroETLIntegrationStatus` enum: `creating`, `active`, `modifying`, `failed`, `deleting`,
`syncing`, `needs_attention`. Note these are **lowercase** — do not compare case-sensitively
against uppercased values.

### Maintenance tracks and patch levels

Current state, provisioned (`Cluster`): `MaintenanceTrackName`, `ClusterVersion`,
`ClusterRevisionNumber`, `PreferredMaintenanceWindow`, `NextMaintenanceWindowStartTime`,
`DeferredMaintenanceWindows[]`, `PendingModifiedValues`, `PendingActions`, `AllowVersionUpgrade`.

`PendingModifiedValues` members include `MaintenanceTrackName`, `ClusterVersion`, `NodeType`,
`NumberOfNodes`, `ClusterType`, `EncryptionType` — i.e. pending track *switches* are visible here.

Current state, serverless (`Workgroup`): `trackName`, `pendingTrackName`, `patchVersion`,
`workgroupVersion`.

Available targets: `MaintenanceTrack{MaintenanceTrackName, DatabaseVersion, UpdateTargets[]}`
from `module-reference`; `ClusterDbRevision{ClusterIdentifier, CurrentDatabaseRevision,
DatabaseRevisionReleaseDate, RevisionTargets[]}`.

Drift is **derived** — a view joining cluster revision against the track catalog. Not a collector.

### Resiliency

From `Cluster`: `MultiAZ`, `MultiAZSecondary`, `AvailabilityZoneRelocationStatus`, `Encrypted`,
`KmsKeyId`, `PubliclyAccessible`, `EnhancedVpcRouting`, `AutomatedSnapshotRetentionPeriod`,
`ManualSnapshotRetentionPeriod`, `ClusterSnapshotCopyStatus{DestinationRegion, RetentionPeriod,
ManualSnapshotRetentionPeriod, SnapshotCopyGrantName}`, `SnapshotScheduleIdentifier`,
`SnapshotScheduleState`, `ExpectedNextSnapshotScheduleTime`, `ClusterAvailabilityStatus`.

Plus `redshift_snapshot_schedules_data`, `redshift_logging_status_data`, and the serverless
recovery-point / snapshot-copy tables.

**`MultiAZ` is a string, not a boolean.** The `Cluster` member is typed `string` with values
`"Enabled"` / `"Disabled"`, unlike the genuinely boolean `Encrypted`, `PubliclyAccessible`,
`EnhancedVpcRouting` and `AllowVersionUpgrade`. The Glue column must be `string`, and every
resiliency measure has to compare against the literal (`multiaz = 'Enabled'`) rather than
treating the column as truthy — a bare `WHERE multiaz` will not parse in Athena, and
`multiaz IS NOT NULL` would count `"Disabled"` clusters as Multi-AZ. Same caution applies to `SnapshotScheduleState` and
`LakehouseRegistrationStatus`, which are also enum strings.

#### Cross-region snapshot copy — detection semantics

This one does not behave like a boolean and is easy to get wrong in three separate ways.

**Absent, not `false`.** `ClusterSnapshotCopyStatus` is documented as *"Returns the destination
region and retention period that are configured for cross-region snapshot copy."* There is no
disabled variant of the struct — and `Cluster` has **no required members**, so when copy is off
the key is simply missing from the response JSON. Detection is therefore "is `DestinationRegion`
populated," and a copy-disabled cluster contributes **no** value to count.

Consequence for the dashboard: any "% of clusters with cross-region copy" measure must take its
denominator from `inventory_redshift_clusters_data` and treat absent as disabled. Counting rows
that *have* the struct yields only the compliant clusters and silently reports 100%.

**Stays a typed struct.** `coding-standards.md` §1.8 prefers raw-JSON `string` columns for
variable nested fields, and Constraints §5 applies that rule to most of this table — but
`ClusterSnapshotCopyStatus` is the carve-out §1.8 reserves for "stable shapes on a pre-created
table": four scalar members, no nested list, unchanged across API versions. Type it as
`struct<destinationregion:string,retentionperiod:bigint,manualsnapshotretentionperiod:int,snapshotcopygrantname:string>`
and query `clustersnapshotcopystatus.destinationregion IS NOT NULL`. Note `RetentionPeriod` is a
`long` while `ManualSnapshotRetentionPeriod` is an `integer`.

Two reasons to keep it typed rather than take the `string` default. It is the only nested field
any view in this spec reads, so it is the one place native struct access earns its keep; and
typing it removes the `'null'` hazard by construction — a copy-disabled cluster omits the key
entirely, which reads as SQL `NULL` with no dependence on the collector guarding against
`to_json(None)`. A `string` column would also work, but only while that guard holds.

**Serverless has no such field.** Cross-region copy for serverless is an entirely separate
top-level API, `redshift-serverless:list_snapshot_copy_configurations`, returning
`SnapshotCopyConfiguration{namespaceName, destinationRegion, destinationKmsKeyId,
snapshotRetentionPeriod, snapshotCopyConfigurationArn, snapshotCopyConfigurationId}`.
`namespaceName` is *optional* on the request, so this is one call per region, not per namespace.
Serverless also exposes a destination KMS key, which provisioned expresses indirectly through
`SnapshotCopyGrantName` → `redshift_snapshot_copy_grants_data`.

### Advisor Impact

`Recommendation`: `Id`, `ClusterIdentifier`, `NamespaceArn`, `CreatedAt`, `RecommendationType`,
`Title`, `Description`, `Observation`, `ImpactRanking`, `RecommendedActions[]`, `ReferenceLinks[]`.

- `ImpactRanking` enum: `HIGH`, `MEDIUM`, `LOW`. Documented as the scale of impact to
  "the performance **and cost**" of the cluster — a blended scale. Label the dashboard
  **"Advisor Impact"**, not "Query Performance".
- `RecommendationType` is **free-form, not an enum**. Safe to `GROUP BY` for a distribution
  chart; never hardcode a value list, or new Advisor types silently drop from totals.
- `RecommendedActions[]` = `{Text, Database, Command, Type}` where `Type` ∈ `SQL`/`CLI` —
  often literal executable SQL, valuable in drill-down.
- `ReferenceLinks[]` = `{Text, Link}`.
- `NamespaceArn` is populated for serverless, so this one table covers both compute models.

## Athena Views

### Governing principle: absence is a value

**Every view in this spec must be driven from the resource table, never from the finding
table.** This is the single most important rule here, and it is the root cause of three
separately-discovered bugs documented below (View 1's Advisor join, View 5's cross-region copy
struct, View 5's serverless copy join). State it once and apply it everywhere.

The Redshift APIs consistently express "feature not configured" and "nothing wrong here" as
**missing data** rather than as an explicit negative value:

| Question | How "no" is encoded | Naive query result |
|---|---|---|
| Does this cluster have Advisor findings? | no rows in `redshift_recommendations_data` | clean clusters vanish from the pane |
| Is cross-region snapshot copy on? | `ClusterSnapshotCopyStatus` key absent from JSON | only copy-enabled clusters counted → 100% compliant |
| Does this serverless namespace copy snapshots? | no row in `..._snapshot_copy_configs_data` | non-copying namespaces vanish |
| Is a snapshot schedule attached? | `SnapshotScheduleIdentifier` absent | unscheduled clusters vanish |
| Is audit logging on? | absent/`false` in `redshift_logging_status_data` | unlogged clusters under-reported |

All five failures are **silent and directionally flattering** — they make the estate look
healthier than it is, which is the worst possible failure mode for a compliance pane. Nothing
errors; totals simply undercount, and a "% compliant" measure trends toward 100% precisely
because non-compliant resources are the ones missing.

Three rules follow:

1. **`FROM` the inventory table, `LEFT JOIN` the finding.** The resource table is the
   denominator and defines the row set. Findings, recommendations, schedules, and copy
   configurations are all optional enrichments joined onto it — never the driving table.
2. **`coalesce` every aggregate to an explicit negative.** `coalesce(max(rank), 0)` →
   `'NONE'`; `destinationregion IS NOT NULL` → `false`. Emit a real value for "no", so it can
   be counted, filtered, and charted like any other.
3. **Never let an absent value become the string `'null'`.** That string is truthy under
   `IS NOT NULL` and inverts the measure. Two ways to satisfy this, per Constraints §5: type
   the column as a struct (closed scalar shapes), or keep it a raw-JSON `string` and never
   serialize a missing value — leave the key absent or `null` rather than writing `to_json(None)`.
   Both land as SQL `NULL`. What is *not* acceptable is flattening unconditionally.

**Distinguish "no" from "not applicable."** These are different and must not collapse. A
provisioned cluster without cross-region copy is non-compliant (`false`). A serverless
workgroup has no `MultiAZ` concept at all (`NULL` → render "N/A"). Coalescing the second to
`false` would mark every serverless workgroup as failing a check that does not apply to it.

**Validation check for every view:** the distinct resource count must equal
`inventory_redshift_clusters_data` + serverless workgroups for the same `collection_date`. If a
view returns fewer rows than the estate has resources, it is dropping the exact resources the
pane exists to surface.

### View 1 — `redshift_advisor_impact_by_cluster`

Cluster grain: one row per cluster with counts by impact level and a worst-rank column.

```sql
WITH latest AS (
  SELECT max(collection_date) AS cd
  FROM optimization_data.redshift_recommendations_data
),
scored AS (
  SELECT r.accountid, r.region, r.clusteridentifier, r.namespacearn,
         r.recommendationtype, r.impactranking,
         CASE r.impactranking
           WHEN 'HIGH' THEN 3 WHEN 'MEDIUM' THEN 2 WHEN 'LOW' THEN 1 ELSE 0
         END AS impact_rank
  FROM optimization_data.redshift_recommendations_data r
  CROSS JOIN latest l
  WHERE r.collection_date = l.cd
)
SELECT c.accountid, c.region, c.clusteridentifier, c.nodetype,
       c.maintenancetrackname, c.clusterversion,
       coalesce(max(s.impact_rank), 0) AS worst_impact_rank,
       CASE coalesce(max(s.impact_rank), 0)
         WHEN 3 THEN 'HIGH' WHEN 2 THEN 'MEDIUM' WHEN 1 THEN 'LOW' ELSE 'NONE'
       END AS worst_advisor_impact,
       count(s.clusteridentifier)                  AS rec_count,
       count_if(s.impactranking = 'HIGH')          AS high_count,
       count_if(s.impactranking = 'MEDIUM')        AS medium_count,
       count_if(s.impactranking = 'LOW')           AS low_count
FROM optimization_data.inventory_redshift_clusters_data c
LEFT JOIN scored s
  ON  c.clusteridentifier = s.clusteridentifier
  AND c.accountid         = s.accountid
  AND c.region            = s.region
GROUP BY 1,2,3,4,5,6
```

Three non-obvious requirements, each of which fails **silently**:

1. **Integer ordinal for `ImpactRanking`.** As a string it sorts `HIGH` < `LOW` < `MEDIUM`
   alphabetically, so `MAX(impactranking)` wrongly returns `MEDIUM` as the worst level. The
   `impact_rank` integer fixes both `MAX()` and QuickSight sort order — expose it as its own
   column so the visual sorts on it rather than the label.
2. **`LEFT JOIN` from the cluster table.** Clusters with zero recommendations have no rows in
   the recommendations table; an inner join drops them and cluster counts won't reconcile
   against inventory. "No findings" is a result the pane should show.
3. **Latest partition before aggregating.** Advisor recommendations are ephemeral — they appear
   and disappear as conditions change. Without the `latest` CTE, resolved items linger across
   all date partitions forever. Same pattern as the `latest_snapshot` CTE in
   `module-inventory.yaml`.

### View 2 — `redshift_advisor_impact_by_account`

`GROUP BY accountid` over View 1 for the account-level tile row: cluster count, clusters with
any HIGH, and worst impact per account.

### View 3 — `redshift_patch_compliance`

Joins `inventory_redshift_clusters_data` + `redshift_cluster_db_revisions_data` against
`redshift_cluster_tracks`, plus serverless workgroups against `redshift_serverless_tracks`.
Outputs current vs. latest available revision, a drift flag, `DatabaseRevisionReleaseDate` age
in days, and pending track switches from `PendingModifiedValues`.

`pendingmodifiedvalues` is a raw-JSON `string` column (Constraints §5), so read the pending
track with `json_extract_scalar(c.pendingmodifiedvalues, '$.MaintenanceTrackName')` rather than
struct access. It returns SQL `NULL` both when no modification is pending and when the pending
change does not involve the track, which is the wanted behaviour — a cluster with nothing
pending is not "pending an unknown track".

### View 4 — `redshift_datashare_topology`

Union of producer- and consumer-side observations, unnesting `DataShareAssociations[]` into a
normalized edge list: producer account/region → consumer account/region, status, cross-region
and cross-account flags, write-enablement. Flags one-sided shares (observed by one party only).

### View 5 — `redshift_resiliency_posture`

Per-cluster resiliency scorecard: Multi-AZ, encryption, public accessibility, automated
snapshot retention, cross-region copy configured, snapshot schedule attached, audit logging
enabled.

**This view is a UNION of two differently-shaped sources, not a single table with a shared
column.** Cross-region copy in particular arrives one way for provisioned clusters (a struct
field on the cluster row) and another for serverless (a join to a separate table):

```sql
-- provisioned: struct field present on the cluster row
SELECT c.accountid, c.region,
       c.clusteridentifier                          AS resource_name,
       'provisioned'                                AS compute_model,
       c.multiaz = 'Enabled'                        AS multiaz_enabled,
       c.encrypted, c.publiclyaccessible,
       c.automatedsnapshotretentionperiod           AS snapshot_retention_days,
       c.clustersnapshotcopystatus.destinationregion  IS NOT NULL
                                                    AS xregion_copy_enabled,
       c.clustersnapshotcopystatus.destinationregion AS xregion_copy_destination,
       c.clustersnapshotcopystatus.retentionperiod   AS xregion_copy_retention_days
FROM optimization_data.inventory_redshift_clusters_data c

UNION ALL

-- serverless: separate API, LEFT JOIN so namespaces without copy still appear
SELECT w.accountid, w.region,
       w.workgroupname                              AS resource_name,
       'serverless'                                 AS compute_model,
       CAST(NULL AS boolean)                        AS multiaz_enabled,
       CAST(NULL AS boolean)                        AS encrypted,
       w.publiclyaccessible,
       CAST(NULL AS integer)                        AS snapshot_retention_days,
       scc.destinationregion IS NOT NULL            AS xregion_copy_enabled,
       scc.destinationregion                        AS xregion_copy_destination,
       scc.snapshotretentionperiod                  AS xregion_copy_retention_days
FROM optimization_data.inventory_redshift_serverless_workgroups_data w
LEFT JOIN optimization_data.redshift_serverless_snapshot_copy_configs_data scc
  ON  w.namespacename = scc.namespacename
  AND w.accountid     = scc.accountid
  AND w.region        = scc.region
```

Both halves must be driven **from** the resource table (plain `SELECT` for provisioned, `LEFT
JOIN` for serverless) so that resources *without* cross-region copy still produce a row with
`xregion_copy_enabled = false`. See the detection semantics under Resiliency above — this is the
same absent-means-disabled trap as the Advisor `LEFT JOIN` requirement in View 1.

Several provisioned-only fields (`MultiAZ`, `Encrypted`, `AutomatedSnapshotRetentionPeriod`) have
no serverless equivalent and are `NULL` on that side; the dashboard must render those as "N/A"
rather than "not configured", or serverless workgroups will read as non-compliant. Serverless
backup posture is instead evidenced by `redshift_serverless_recovery_points_data`. The serverless
`NULL`s are explicitly `CAST` because a bare `NULL` in a `UNION ALL` branch leaves Athena to infer
the column type from the other branch — which silently works today and breaks if the provisioned
branch is ever reordered or a third branch is added.

The provisioned branch converts `multiaz` to a boolean **in the view**, not in the collector: the
Glue column stays the raw API string (see Resiliency above), and the view exposes
`multiaz_enabled` so the three-state distinction survives — `true` (Multi-AZ), `false`
(`"Disabled"`), `NULL` (serverless, not applicable). Comparing against the literal is what keeps
`"Disabled"` out of the compliant bucket.

Both halves need the latest-`collection_date` filter before use, per View 1.

## Constraints and Implementation Notes

Verified against boto3 1.43.102.

1. **Serverless is a separate client, but needs no bespoke scan function.**
   `redshift-serverless` uses camelCase members and `nextToken` pagination, distinct from
   `redshift`'s PascalCase and `Marker`. The default `obj_name` convention
   (`function_name.split('_')[-1].capitalize() + '[*]'`) does not produce correct paths for it,
   but `paginated_scan` accepts `obj_name` explicitly, and boto3 registers paginators for
   `list_namespaces`, `list_workgroups`, `list_recovery_points`,
   `list_snapshot_copy_configurations` and `list_tracks` — so `nextToken` is handled
   transparently. A plain `partial(paginated_scan, service='redshift-serverless',
   obj_name='namespaces[*]')` is sufficient.

2. **`describe_logging_status` is the only true N+1.** Requires `ClusterIdentifier` and does
   not paginate — needs a bespoke function following the `opensearch_domains_scan` /
   `eks_clusters_scan` pattern in `module-inventory.yaml`. Everything else in the table above
   paginates and takes no required arguments, so it is one call per account+region.

3. **`describe_cluster_snapshots` is excluded from v1.** Unbounded result set; if added later
   it must filter `SnapshotType` and a `StartTime` window or it will exceed the 300s Lambda
   timeout and inflate S3/Glue cost.

4. **boto3 layer required.** `describe_integrations`, serverless `list_tracks`, and newer
   `Cluster` fields (`LakehouseRegistrationStatus`, `CatalogArn`,
   `ExtraComputeForAutomaticOptimization`, `MultiAZ`) postdate the Lambda runtime's bundled
   boto3. Use the conditional `Layers` pattern from `module-health-events.yaml:143` against
   `Boto3LayerVersion` (`deploy-data-collection.yaml:1059`).

5. **Nested structures: `string` by default, typed struct only for closed scalar shapes.**
   Per `coding-standards.md` §1.8, a raw-JSON `string` column never drifts and survives API
   schema changes; typed structs are reserved for stable shapes on a pre-created table. The
   dividing line that matters in practice is **whether the API type contains a nested list**,
   because that is what varies and what a hand-written struct silently truncates.

   `string`, because the API type carries a nested list (member that a struct type drops
   shown in parentheses):

   | Field | Nested member |
   |---|---|
   | `PendingModifiedValues` | 11 members, varies by pending operation |
   | `MultiAZSecondary` | `SecondaryClusterInfo.ClusterNodes` |
   | `Endpoint` (provisioned and serverless) | `Endpoint.VpcEndpoints` / `vpcEndpoints` |
   | `ClusterParameterGroups` | `ClusterParameterGroupStatus.ClusterParameterStatusList` |

   `PendingModifiedValues` is also already a `string` column on the RDS and ElastiCache tables
   in `module-inventory.yaml`, so typing it here would be locally inconsistent.

   Typed struct, being closed shapes with only scalar members: `ClusterSnapshotCopyStatus`
   (see Resiliency for why this one earns it), `DeferredMaintenanceWindows`, `IamRoles`,
   `VpcSecurityGroups`, `PricePerformanceTarget`. `Tags` must stay
   `array<struct<Key:string,Value:string>>` regardless — the handler's tag-enrichment loop
   iterates `obj["Tags"]` and breaks on a string.

   For `module-redshift`'s own tables the same test applies to `DataShareAssociations`,
   `RecommendedActions`, `ReferenceLinks`, `UpdateTargets` and `RevisionTargets`.

   **No collector change is needed to emit a `string` column.** The SerDe returns the raw JSON
   text for an object mapped to a `string` column — which is how the existing RDS and
   ElastiCache `pendingmodifiedvalues` columns work, both collected with plain
   `paginated_scan` and no serialization step. The `to_json()` flattening of Lambda
   `Environment` in the handler solves a different problem: env-var keys are arbitrary, so the
   crawler would otherwise invent unbounded columns. Fixed-key objects do not need it. Note
   that if a value *is* serialized, it must be guarded (`obj.get(k) is not None`) — `to_json(None)`
   produces the literal `'null'`, the hazard in the governing principles above.

6. **Scrub `PendingModifiedValues.MasterUserPassword` before the write.** The API can report a
   pending admin password there, and `paths` collects the object whole, so it would otherwise
   reach S3 and Glue. Drop the key in the handler alongside the existing `Environment` special
   case. This is independent of the `string`-vs-struct choice: a typed struct that merely omits
   the member still leaves the value in the JSONL on S3 — only the Athena projection hides it.

7. **Schedule.** Module default is `rate(14 days)`. Patch/maintenance and Advisor panes need
   **daily** or `NextMaintenanceWindowStartTime` goes stale past the event and Advisor findings
   lag reality.

8. **Timestamp serialization.** Reuse the existing `to_json` helper (ISO-8601 via
   `default=lambda x: x.isoformat() ...`). Many Redshift fields are timestamps:
   `ClusterCreateTime`, `CreatedAt`, `CreateTime`, `DatabaseRevisionReleaseDate`,
   `NextMaintenanceWindowStartTime`, `ExpectedNextSnapshotScheduleTime`,
   `customDomainCertificateExpiryTime`.

9. **Regional availability.** Redshift Serverless is not in every region where provisioned
   Redshift exists. Per-region calls must tolerate endpoint/`AccessDenied` errors without
   failing the account — the existing broad `except` in `paginated_scan` covers this, but
   avoid a hardcoded region allowlist like `WORKSPACES_REGIONS` unless measurably needed.

## IAM

Added to `deploy-in-linked-account.yaml` as `RedshiftPolicy`, conditional on
`IncludeRedshiftModulePolicy`:

```yaml
Action:
  - "redshift:DescribeClusters"
  - "redshift:DescribeClusterDbRevisions"
  - "redshift:DescribeDataShares"
  - "redshift:DescribeDataSharesForConsumer"
  - "redshift:DescribeDataSharesForProducer"
  - "redshift:DescribeInboundIntegrations"
  - "redshift:DescribeIntegrations"
  - "redshift:DescribeLoggingStatus"
  - "redshift:DescribeSnapshotSchedules"
  - "redshift:DescribeSnapshotCopyGrants"
  - "redshift:DescribeUsageLimits"
  - "redshift:DescribeReservedNodes"
  - "redshift:DescribeEndpointAccess"
  - "redshift:ListRecommendations"
  - "redshift-serverless:ListNamespaces"
  - "redshift-serverless:ListWorkgroups"
  - "redshift-serverless:ListRecoveryPoints"
  - "redshift-serverless:ListSnapshotCopyConfigurations"
Resource: "*"   # these actions do not support resource-level permissions
```

Needs the `cfn_nag` W12 suppression used by the sibling policies. All actions are read-only;
no `GetClusterCredentials` (that would be required only if the Data API phase is ever added).

## Wiring Checklist

- [ ] `data-collection/deploy/module-redshift.yaml` — new template
- [ ] `data-collection/deploy/module-inventory.yaml` — 3 `AwsObjects` entries, `ServicesMap`
      entries with Glue table definitions, `sub_modules` registrations, serverless scan function
- [ ] `data-collection/deploy/module-reference.yaml` — 3 catalog tables + collectors
- [ ] `data-collection/deploy/deploy-data-collection.yaml` — `IncludeRedshiftModule` parameter,
      `ParameterGroups`/`ParameterLabels` entries, `DeployRedshiftModule` condition, nested
      `RedshiftModule` stack (model on the `InventoryCollectorModule` block, lines 1184–1209)
- [ ] `data-collection/deploy/deploy-in-linked-account.yaml` — `IncludeRedshiftModule`
      parameter, `IncludeRedshiftModulePolicy` condition, `RedshiftPolicy`, and the
      `${ResourcePrefix}redshift-LambdaRole` ARN added to the `LambdaRole` trust list (line ~159)
- [ ] **No** change to `deploy-in-management-account.yaml` — LINKED-only. Org-wide data-share
      reconciliation works from linked accounts alone, since both endpoints of any share are
      themselves linked accounts.
- [ ] `test/utils.py` — register new state machine ARNs (see lines 386–399)
- [ ] `data-collection/README.md` — module table row
- [ ] `data-collection/CHANGELOG.md` — entry

## Downstream

This repo produces Glue tables and Athena views only. The QuickSight dashboard consuming them
is a separate deliverable in the CID dashboards / `cid-cmd` project.
