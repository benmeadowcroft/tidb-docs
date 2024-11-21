---
title: Overview of TiDB Disaster Recovery Solutions
summary: Learn about the disaster recovery solutions provided by TiDB, including disaster recovery based on primary and secondary clusters, disaster recovery based on multiple replicas in a single cluster, and disaster recovery based on backup and restore.
---

# Overview of TiDB Disaster Recovery Solutions

TiDB provides multiple options for delivering Disaster Recovery (DR) for your cluster. Disaster Recovery focuses on responding to larger scale threats to the availability and integrity of your data. Such threats may include natural disasters, technical issues, human error, and even malicious actions.

This document introduces common approaches to delivering a Disaster Recovery solution with TiDB and understanding the trade-offs between the solutions.

## Key concepts

It is important to identify the Recovery Time Objective (RTO) and Recovery Point Objective (RPO) that are acceptable for your system. These objectives will help guide selection of appropriate Disaster Recovery solutions and ensure they align with your overall Service Level Objectives (SLO).

- RPO (Recovery Point Objective): In the event of a disaster occurring, the RPO represents the maximum amount of data loss that the organization can tolerate for this service.

- RTO (Recovery Time Objective): After a disaster has occurred, the RTO represents the maximum amount of time that the organization can tolerate until this service to be fully operational.

![RTO and RPO](/media/dr/rto-rpo.png)

- SLO (Service Level Objectives): In addition to RPO and RTO, the SLO's outline other objectives for the service. These may include objectives around Performance, Availability, and Reliability. As you consider the different approaches to meeting your recovery related objectives, you will also want to ensure that you are satisfying the other objectives for the service.

## Disaster Recovery capabilities provided by TiDB

TiDB includes multiple recovery capabilities to address different recovery needs. These include:

- TIDB distributed cluster design delivers High Availability within a geographic region.
- Cross-cluster replication with TiCDC delivers DR across different geographic regions.
- Flashback delivers rapid recovery from destructive DML/DDL statements.
- Backup and restore (with Point-in-Time-Recovery) delivers fine-grained recovery from cluster failures, or destructive statements.

### TiDB High Availability

TiDB's distributed architecture and use of the [Raft protocol](./best-practices/tidb-best-practices.md#raft) enables TiDB to deliver high availability against failure of individual hosts. To be resilient against the failure of an entire Availaiblity Zone (AZ), a TiDB cluster can also be distributed across multiple AZs within a single geographic region.

- Tutorial: [Multiple Availability Zones in One Region Deployment](./multi-data-centers-in-one-city-deployment.md)
- Solution Detail: [DR Solution Based on Multiple Replicas in a Single Cluster](./dr-multi-replica.md)

### Backup and Restore

TiDB's BR (Backup and Recovery) provides periodic backups of the data within a TiDB cluster. Additionally log backups and PiTR (Point-in-Time-Recovery) allow TiDB to provide fine-grained recovery to any point in time within the retained backups. TiDB backups also support immutable object storage as a backup target for protection against accidental or malicious deletional.

The Backup and Restore capability is critical for delivering recovery options for threats that impact the data itself (e.g. inadvertent `DELETE/TRUNCATE/DROP/UPDATE` operations), and not the availability of the cluster.

- Feature Overview: [Backup and Restore Overview](./br/backup-and-restore-overview.md)
- Usage Overview: [Backup and Restore Overview](./br/br-use-overview.md)
- Feature Guide: [Snapshot Backup and Restore Guide](./br/br-snapshot-guide.md)
- Feature Guide: [Log Backup and PITR Guide](./br/br-pitr-guide.md)
- Solution Guide: [DR Solution Based on BR](./dr-backup-restore.md)

### Cross-Cluster Replication with TiCDC

TiDB's Change Data Capture (TiCDC) can replicate incremental data from one TiDB cluster to another. TiCDC can be configured to leverage object storage for [redo logs](./ticdc/ticdc-sink-to-mysql.md#disaster-recovery) to support eventual consistency in the face of a disaster.

A TiCDC solution for DR requires at least two clusters. If TiCDC is setup to replicate in one direction, the secondary cluster can be used to run read-only queries for less latency-sensitive applications using TiCDC's [Syncpoint feature](./dr-secondary-cluster.md#query-business-data-on-the-secondary-cluster). Multiple clusters can also be configured with [bidirectional replication](./ticdc/ticdc-bidirectional-replication.md) to support a multi-active cluster solution.

- Feature Overview: [TiCDC Overview](./ticdc/ticdc-overview.md)
- Solution Guide: [DR Solution Based on Primary and Secondary Clusters](./dr-secondary-cluster.md)
- Solution Guide: [Perform Bi-Directional Replication Between Clusters](./dr-secondary-cluster.md#perform-bidirectional-replication-between-the-primary-and-secondary-clusters)

### Flashback

TiDB's `FLASHBACK` capability provides the ability to recover from certain destructive DDL or DML statements by restoring a snapshot of the cluster, using data that is more recent than the current [Garbage Collection](./garbage-collection-overview.md) (GC) [safe point](./status-variables.md#tidb_gc_safe_point). `FLASHBACK CLUSTER` restores the cluster to a specific snapshot. `FLASHBACK DATABASE` restores a database and its data deleted by the DROP statement. `FLASHBACK TABLE` restores a tables and its data dropped by the DROP or TRUNCATE operation.

- Reference: [`FLASHBACK CLUSTER`](./sql-statements/sql-statement-flashback-cluster.md)
- Reference: [`FLASHBACK DATABASE`](./sql-statements/sql-statement-flashback-database.md)
- Reference: [`FLASHBACK TABLE`](./sql-statements/sql-statement-flashback-table.md)

## Comparison of available DR options

> **Note:**
>
> - While these tables distinguish between the individual capabilities, they may be combined together to provide greater protection. For example, the BR tool may be configured to provide recovery from destructive DML/DDL, while TiCDC is used to provide faster recovery from a regional disaster.

### Applicable recovery scope

What kinds of disaster impact/scope does each capability guard against?

| Capability | Resilient to Host Failure | …Availability Zone Failure | …Region Failure | …Destructive DML/DDL |
| ------------------ | :-------------: | :-------------: | :-------------: | :-------------: |
| Single AZ deployment  | Yes | No | No | No |
| Multi-AZ deployment | Yes | Yes | No | No |
| 2+ Clusters with TiCDC | Not Applicable | Yes, if clusters are in different AZs | Yes, if clusters are in different geographic regions | No |
| Flashback | Not Applicable | No, requires cluster availability | No, requires cluster availability | Yes |
| Backup and Restore | Not Applicable | Yes, when backup data is stored outside of AZ | Yes, when backup data is stored outside of region | Yes |

### Achievable recovery characteristics

What performance and recovery options does each capability provide?

| Capability | Recovery Granularity |  RPO  |  RTO  | Recovery Point Selection |
| ---------- | :------------: | :---: | :---: | :----------------------: |
| Single AZ deployment | Cluster | Zero Data Loss | Zero Downtime | N/A |
| Multi-AZ deployment | Cluster | Zero Data Loss | Zero Downtime | N/A |
| 2+ Clusters with TiCDC  | Changefeed <br/>(set of Databases/Tables) | Less than 10 seconds | Less than 5 minutes | Latest, or within GC window (with Flashback and syncpoint) |
| Flashback (Cluster)  | Cluster | Up to [GC life time](./system-variables.md#tidb_gc_life_time-new-in-v50) | Seconds | Up to [GC safe point](./status-variables.md#tidb_gc_safe_point) |
| Flashback (Database/Table)  | Database/Table | Up to [GC life time](./system-variables.md#tidb_gc_life_time-new-in-v50) | Seconds | `DROP/TRUNCATION` point only |
| Backup and Restore  | Cluster/Databases/Tables | Less than 5 minutes | Hours (scales with data size) | Any retained point in time |

### Cost considerations

What are the key cost of ownership considerations for each capability?

| Capability | TCO Considerations |
| ---------- | :----------------- |
| Single AZ deployment | <ul><li>Operational complexity recovering from disasters impacting the zone</li></ul> | 
| Multi-AZ deployment | <ul><li>Network traffic costs for cross-AZ traffic</li></ul> | 
| 2+ Clusters with TiCDC (uni-directional)  | <ul><li>Secondary cluster costs</li><li>Network traffic costs for cross-region replication</li></ul> |
| 2+ Clusters with TiCDC (bidirectional) | <ul><li>Application complexity for regional affinity</li><li>Secondary cluster costs</li><li>Network traffic costs for cross-region replication</li></ul> |
| Flashback  | <ul><li>TiKV Storage overhead (particularly if extending [GC life time](./system-variables.md#tidb_gc_life_time-new-in-v50))</li></ul> | 
| Backup and Restore  | <ul><li>Backup storage costs</li><li>Network traffic costs for cross-region replication</li></ul> | 