# AWS EC2 Storage

| Feature | Amazon EBS | Amazon EFS | EC2 instance store |
| --- | --- | --- | --- |
| Storage and connection | Network-attached block storage | Shared network file storage (NFS) | Host-local block storage |
| Data after an EC2 stop | Preserved | Preserved independently of EC2 | Lost |
| Access from multiple instances | One instance by default, with Multi-Attach on supported volumes | Concurrent file access across AZs | Attached to one instance for its lifetime |
| Automatic capacity scaling | No | Yes | No |
| Typical use | Operating systems, databases, and persistent application data | Shared web content and application files | Caches, buffers, and temporary processing |

## Amazon EBS (Elastic Block Store)

An **EBS volume** is a persistent virtual disk that EC2 accesses over the network. For example, EC2 runs a Node.js application while its EBS volume stores the operating system, dependencies, application files, logs, and persistent data.

An EBS volume belongs to one Availability Zone (AZ). A volume in `us-east-1a` can be detached and reattached to another compatible instance in `us-east-1a`. To use its data in `us-east-1b`, create a new volume there from a snapshot.

EBS charges for provisioned storage capacity and, depending on the volume type, for provisioned I/O performance.

### Delete on termination

The per-volume **Delete on termination** setting (`DeleteOnTermination`) controls whether EC2 deletes an EBS volume when its instance terminates. A stop preserves attached volumes. Volumes retained after termination continue to incur charges.

| Volume attachment | Default on termination |
| --- | --- |
| Root volume attached at launch | Delete |
| Data volume attached at launch through the EC2 console | Preserve |
| Data volume attached at launch through the AWS CLI | Delete |
| Root or data volume attached after launch | Preserve |

The AMI's block device mapping can supply the launch setting, so check the actual value. To retain a root volume after termination, set `DeleteOnTermination` to `false` at launch through the EC2 console or AWS CLI. For an existing instance, use the AWS CLI to change the root volume's setting. [AWS documentation on EBS volume retention](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/preserving-volumes-on-termination.html)

### EBS volume types

**Capacity** is the volume size in GiB or TiB. **IOPS** measures the number of input/output operations per second. **Throughput** measures how much data moves per second, typically in MiB/s. **Latency** is the time a single I/O operation takes.

The four SSD types support EC2 boot volumes. Both HDD types are for data volumes and perform best with large, sequential I/O.

| Type | Capacity | Maximum IOPS | When to use it |
| --- | --- | --- | --- |
| General Purpose SSD (`gp3`) | 1 GiB–64 TiB | 80,000 | Common default for application storage, boot volumes, development, and testing because it balances cost and configurable performance. Also suited to virtual desktops, transactional workloads, and medium-sized databases. |
| General Purpose SSD (`gp2`) | 1 GiB–16 TiB | 16,000 | Similar general-purpose workloads, with baseline performance that scales with volume size. |
| Provisioned IOPS SSD (`io1`) | 4 GiB–16 TiB | 64,000 | I/O-intensive databases and other applications needing sustained provisioned IOPS. |
| Provisioned IOPS SSD (`io2` Block Express) | 4 GiB–64 TiB | 256,000 | Critical databases and transaction-intensive applications that need sustained IOPS and predictable sub-millisecond latency. |
| Throughput Optimized HDD (`st1`) | 125 GiB–16 TiB | 500 | Frequently accessed sequential data, such as big data, data warehouses, and log processing. |
| Cold HDD (`sc1`) | 125 GiB–16 TiB | 250 | Infrequently accessed data with modest performance requirements and a priority on low storage cost. |

- **`gp3`** includes 3,000 IOPS and 125 MiB/s of baseline throughput. You can provision additional IOPS and throughput separately from capacity, within size and IOPS-to-throughput limits. Throughput tops out at 2,000 MiB/s.
- **`gp2`** has a baseline of 3 IOPS per GiB, with a floor of 100 IOPS. Volumes below 1 TiB with a baseline below 3,000 IOPS can burst to 3,000 IOPS while burst credits last.
- **`io1`** lets you set IOPS explicitly, up to 50 IOPS per GiB. Supported Nitro-based instances can reach 64,000 IOPS. Other supported instances can reach 32,000 IOPS.
- **`io2` Block Express** supports up to 1,000 IOPS per GiB and is designed for average latency under 500 microseconds with 16 KiB I/O.
- **`st1`** supports up to 500 MiB/s, and **`sc1`** supports up to 250 MiB/s. The table's HDD IOPS limits assume 1 MiB I/O operations.

Actual performance depends on the volume configuration, I/O size, and EC2 instance limits. [AWS EBS volume type specifications](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)

### EBS Multi-Attach

**Multi-Attach** gives up to 16 compatible Nitro-based EC2 instances in the same AZ read and write access to one enabled `io1` or `io2` volume. Linux supports both volume types. Windows supports `io2`.

For example, three EC2 instances in `us-east-1a` can share an `io2` volume for a high-availability Linux cluster or a clustered database such as Teradata. The application must coordinate concurrent writes to prevent data corruption. If multiple instances mount the same file system, use a cluster-aware file system. Standard ext4 and XFS are designed for mounting on a single host at a time. [AWS Multi-Attach requirements](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes-multi.html)

## EBS snapshots

An **EBS snapshot** is a point-in-time backup of a volume for recovery or migration. The first snapshot captures all written blocks. Later snapshots in the standard tier store changed blocks, and each can restore the volume's data as it existed at that point in time. AWS stores snapshots in service-managed Amazon S3 buckets and exposes them through EC2.

You can take a snapshot while a volume is attached. To capture an application-consistent backup, flush buffered writes and pause application writes as needed. [AWS documentation on creating snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-snapshot.html)

### Restore in another Availability Zone

1. Create a snapshot of the EBS volume in `us-east-1a`.
2. Create a new volume from the completed snapshot in `us-east-1b`.
3. Attach the new volume to an EC2 instance in `us-east-1b`.

A snapshot can create a volume in any AZ in its Region. To restore in another Region, copy the snapshot to that Region first. [AWS documentation on EBS snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html)

### Snapshot Archive, Recycle Bin, and Fast Snapshot Restore

- **Snapshot Archive** is for rarely accessed snapshots kept long term. Archiving converts an incremental snapshot to a full snapshot and can lower snapshot storage costs by up to 75% for suitable retention patterns. The minimum archive billing period is 90 days. Restore an archived snapshot to the standard tier before using it. This can take up to 72 hours. [AWS Snapshot Archive](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-archive.html) · [Archive restore limits](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-archive-considerations.html)
- **Recycle Bin** protects against accidental deletion by retaining snapshots that match a retention rule. The rule can retain a snapshot for 1–365 days, during which it can be recovered. [AWS Recycle Bin retention rules](https://docs.aws.amazon.com/ebs/latest/userguide/recycle-bin-create-rule.html)
- **Fast Snapshot Restore (FSR)** creates a fully initialized volume, avoiding the first-read latency that occurs when blocks are loaded on demand. Enable it for the snapshot in the target AZ. AWS charges for each snapshot and AZ pair while FSR is enabled. [AWS Fast Snapshot Restore](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-fast-snapshot-restore.html)

## Amazon Machine Image (AMI)

An **AMI** is a reusable template for launching EC2 instances. It supplies the operating system and can include installed applications, dependencies, system settings, and monitoring tools. Its **block device mapping** specifies the volumes to attach at launch.

An EBS-backed AMI references EBS snapshots for its disk data. When you launch an instance, the AMI supplies the launch configuration and EC2 uses the snapshots to create the EBS volumes. AMIs are Region-specific and can be copied to another Region. [AWS documentation on AMIs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html)

| AMI source | Description |
| --- | --- |
| AWS-provided | Public images maintained by AWS |
| Custom | Images you create and maintain |
| AWS Marketplace | Third-party images that may carry a software charge |

For example, create a custom AMI with Node.js, dependencies, and monitoring tools already installed. Launching from that image speeds up deployment, keeps the configuration consistent, and removes the need to repeat the setup on each instance.

## EC2 instance store

**Instance store** provides ephemeral block storage on disks physically attached to the EC2 host. On supported instance types, it offers low latency and high disk I/O for data that can be recreated, such as caches, buffers, scratch files, and intermediate processing results.

Data survives a reboot. Stopping, hibernating, or terminating the instance erases the data, and a host or disk failure can also cause data loss. Instance store volumes are attached at launch and stay tied to that instance. They have no direct EBS snapshot support. If you need to retain the data, copy or replicate it to persistent storage. [AWS documentation on instance store persistence](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-store-lifetime.html)

## Amazon EFS (Elastic File System)

**Amazon EFS** is a managed, POSIX-compatible network file system that multiple EC2 instances can mount concurrently. It supports NFSv4.0 and NFSv4.1 and is commonly used with Linux workloads. Capacity grows and shrinks as files are added and removed. EFS can scale to petabytes and support thousands of concurrent client connections. Aggregate throughput depends on the configuration and service quotas. [AWS documentation on EFS mounts](https://docs.aws.amazon.com/efs/latest/ug/how-it-works.html) · [EFS quotas](https://docs.aws.amazon.com/efs/latest/ug/limits.html)

EFS is useful for web serving, content management systems, and shared application directories. For example, if two WordPress instances mount the same EFS file system, an image uploaded through one instance is available to the other through the shared directory.

### Availability and access controls

- **Regional EFS** stores data redundantly across multiple AZs in a Region. It provides the availability and resilience typically needed for production and supports a mount target in each AZ.
- **EFS One Zone** stores data in a single AZ and uses a single mount target there. It costs less and suits development, testing, or workloads that can tolerate AZ-level data loss. Clients in another AZ can reach the mount target, with cross-AZ data transfer charges. [AWS EFS availability and mount targets](https://docs.aws.amazon.com/efs/latest/ug/how-it-works.html)

Security groups control network access to mount targets. EFS supports encryption at rest with AWS KMS, TLS for encryption in transit, and POSIX ownership and permissions for file access.

### Performance and throughput modes

The **performance mode** affects the latency of individual file operations and how the file system handles parallel I/O.

| Performance mode | Characteristics |
| --- | --- |
| General Purpose | Default low-latency mode for workloads such as web servers, content management systems, and shared files. |
| Max I/O | Previous-generation mode for highly parallel workloads such as big data or media processing that can tolerate higher per-operation latency. Applies to Regional file systems using Bursting or Provisioned throughput. AWS recommends General Purpose for all file systems. |

The **throughput mode** determines how read and write capacity is allocated.

| Throughput mode | How it works | When it fits |
| --- | --- | --- |
| Bursting | Baseline throughput scales with data stored in EFS Standard. Burst credits allow temporary increases above baseline. | Workloads with a storage-dependent baseline and short peaks. |
| Provisioned | You set throughput independently of stored capacity. Charges apply to provisioned throughput above the included baseline. | Sustained throughput needs with relatively little stored data. |
| Elastic | EFS adjusts throughput automatically and meters reads and writes. | Unpredictable or frequently changing demand. |

[AWS EFS performance specifications](https://docs.aws.amazon.com/efs/latest/ug/performance.html)

### Storage classes and lifecycle management

EFS storage charges are based on the amount of data in each storage class. Throughput and data access charges depend on the selected modes and classes.

| Storage class | Use case and cost |
| --- | --- |
| EFS Standard | Frequently accessed files for active applications that need low latency. |
| EFS Infrequent Access (IA) | Occasionally accessed data with lower storage costs and data access charges. |
| EFS Archive | Data accessed a few times a year or less, with lower storage costs and data access charges. Requires Regional EFS with Elastic throughput. |

**Lifecycle management** applies policies to move files between classes. For example, a new file starts in Standard. A policy can move it to Infrequent Access after 60 days without access to reduce storage costs. Policies can also move eligible files to Archive and return files to Standard on first access. [AWS EFS storage classes](https://docs.aws.amazon.com/efs/latest/ug/features.html#storage-classes) · [EFS lifecycle policies](https://docs.aws.amazon.com/efs/latest/ug/enable-lifecycle-management.html)
