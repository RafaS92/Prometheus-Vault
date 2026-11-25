# Amazon EC2 Storage

## 1. Storage Options at a Glance

| Feature | Amazon EBS | EC2 instance store | Amazon EFS |
| --- | --- | --- | --- |
| Storage and connection | Network-attached block storage | Host-local block storage | Shared network file storage (NFS) |
| Data after reboot | Preserved | Preserved | Preserved independently of EC2 |
| Data after stop or hibernation | Preserved | Lost | Preserved independently of EC2 |
| Data after termination | Depends on the volume's Delete on termination setting | Lost | Preserved independently of EC2 |
| Access from multiple instances | One instance by default, with Multi-Attach on supported volumes | Attached to one instance for its lifetime | Concurrent NFS access through mount targets |
| Automatic capacity scaling | No | No | Yes |
| Typical use | Operating systems, databases, and persistent application data | Caches, buffers, and temporary processing | Shared web content and application files |

### Storage Performance Terms

- **Capacity** is the amount of data the storage can hold. EBS volume sizes use GiB or TiB.
- **IOPS** measures how many input/output operations complete per second.
- **Throughput** measures how much data transfers per second, commonly in MiB/s.
- **Latency** measures how long one I/O operation takes.

For a transaction-heavy database, compare random IOPS and throughput against the workload's requirements.

## 2. Amazon EBS (Elastic Block Store)

An **EBS volume** is a persistent block device that EC2 accesses over the network. For example, an EC2 instance can run a Node.js application while an attached EBS volume stores the operating system, dependencies, application files, logs, and persistent data.

An EBS volume belongs to one Availability Zone (AZ). You can detach it and reattach it to another compatible instance in that AZ. To move the data to another AZ, [restore a snapshot there](#restoring-a-snapshot). [EBS volume attachment](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-attaching-volume.html)

EBS charges for provisioned storage capacity and, depending on the volume type, for provisioned I/O performance.

### Delete on Termination

The per-volume **Delete on termination** setting (`DeleteOnTermination`) controls whether EC2 deletes an EBS volume when its instance terminates. Volumes retained after termination continue to incur charges.

| Volume attachment | Default on termination |
| --- | --- |
| Root volume attached at launch | Delete |
| Data volume attached at launch through the EC2 console | Preserve |
| Data volume attached at launch through the AWS CLI | Delete |
| Root or data volume attached after launch | Preserve |

The AMI's block device mapping can supply the launch setting, so check the actual value. To retain a root volume after termination, set `DeleteOnTermination` to `false` at launch through the EC2 console or AWS CLI. For an existing instance, use the AWS CLI to change the root volume's setting. [AWS documentation on EBS volume retention](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/preserving-volumes-on-termination.html)

### EBS Volume Types

SSD volumes support EC2 boot volumes and suit workloads with frequent random I/O. HDD volumes serve as data volumes and perform best with large, sequential I/O.

| Type | Capacity | Maximum IOPS | When to use it |
| --- | --- | --- | --- |
| General Purpose SSD (`gp3`) | 1 GiB–64 TiB | 80,000 | General-purpose application storage, boot volumes, development and testing, virtual desktops, transactional workloads, and medium-sized databases. |
| General Purpose SSD (`gp2`) | 1 GiB–16 TiB | 16,000 | Similar general-purpose workloads, with baseline performance that scales with volume size. |
| Provisioned IOPS SSD (`io1`) | 4 GiB–16 TiB | 64,000 | I/O-intensive databases and other applications needing sustained provisioned IOPS. |
| Provisioned IOPS SSD (`io2` Block Express) | 4 GiB–64 TiB | 256,000 | Critical databases and transaction-intensive applications that need sustained IOPS and predictable sub-millisecond latency. |
| Throughput Optimized HDD (`st1`) | 125 GiB–16 TiB | 500 | Frequently accessed sequential data, such as big data, data warehouses, and log processing. |
| Cold HDD (`sc1`) | 125 GiB–16 TiB | 250 | Infrequently accessed data with modest performance requirements and a priority on low storage cost. |

- **`gp3`** is the default when you create a volume in the EC2 console. It includes 3,000 IOPS and 125 MiB/s of baseline throughput. You can provision additional IOPS and throughput separately from capacity, within size and IOPS-to-throughput limits. Throughput tops out at 2,000 MiB/s. [Creating EBS volumes](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-volume.html) · [General Purpose SSD performance](https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose.html)
- **`gp2`** has a baseline of 3 IOPS per GiB, with a floor of 100 IOPS. Volumes below 1 TiB with a baseline below 3,000 IOPS can burst to 3,000 IOPS while burst credits last.
- **`io1`** lets you set IOPS explicitly, up to 50 IOPS per GiB. Supported Nitro-based instances can reach 64,000 IOPS. Other supported instances can reach 32,000 IOPS.
- **`io2` Block Express** supports up to 1,000 IOPS per GiB and is designed for average latency under 500 microseconds with 16 KiB I/O.
- **`st1`** supports up to 500 MiB/s, and **`sc1`** supports up to 250 MiB/s. The table's HDD IOPS limits assume 1 MiB I/O operations.

Actual performance depends on the volume configuration, I/O size, and EC2 instance limits. [AWS EBS volume type specifications](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)

### EBS Multi-Attach

**Multi-Attach** gives up to 16 compatible Nitro-based EC2 instances in the same AZ read and write access to one enabled `io1` or `io2` volume. Linux supports both volume types. Windows supports `io2`.

For example, three EC2 instances in `us-east-1a` can share an `io2` volume in a high-availability Linux cluster. Teradata also uses `io2` Multi-Attach for database failover. [Teradata Multi-Attach case study](https://aws.amazon.com/solutions/case-studies/teradata-ebs-multi-attach-case-study/)

The application must coordinate concurrent writes to prevent data corruption. If multiple instances mount the same file system, use a cluster-aware file system. Standard ext4 and XFS are designed for mounting on a single host at a time. Use a separate volume for booting. [AWS Multi-Attach requirements](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes-multi.html)

## 3. EBS Snapshots

An **EBS snapshot** is a point-in-time backup of a volume for recovery or migration. The first snapshot captures all written blocks. Later snapshots in the standard tier store changed blocks, and each can restore the volume's data as it existed at that point in time. AWS stores snapshots in service-managed Amazon S3 buckets and exposes them through EC2.

You can take a snapshot while a volume is attached. To capture an application-consistent backup, flush buffered writes and pause application writes as needed. [AWS documentation on creating snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-snapshot.html)

### Restoring a Snapshot

A regional snapshot can create a volume in any AZ in its Region. To restore in another Region, first copy the snapshot to that Region. For example, to recover data in a second AZ, follow these steps. [AWS documentation on EBS snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html)

1. Create a snapshot of the EBS volume in `us-east-1a`.
2. Create a new volume from the completed snapshot in `us-east-1b`.
3. Attach the new volume to an EC2 instance in `us-east-1b`.

### Snapshot Archive, Recycle Bin, and Fast Snapshot Restore

- **Snapshot Archive** is for rarely accessed snapshots kept long term. Archiving converts an incremental snapshot to a full snapshot and can lower snapshot storage costs by up to 75% for suitable retention patterns. The minimum archive billing period is 90 days. Restore an archived snapshot to the standard tier before using it. This can take up to 72 hours. [AWS Snapshot Archive](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-archive.html) · [Archive restore limits](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-archive-considerations.html)
- **Recycle Bin** protects against accidental deletion by retaining snapshots that match a retention rule. The rule can retain a snapshot for 1–365 days, during which it can be recovered. [AWS Recycle Bin retention rules](https://docs.aws.amazon.com/ebs/latest/userguide/recycle-bin-create-rule.html)
- **Fast Snapshot Restore (FSR)** creates a fully initialized volume, avoiding the first-read latency that occurs when blocks are loaded on demand. Enable it for the snapshot in the target AZ. AWS charges for each snapshot and AZ pair while FSR is enabled. [AWS Fast Snapshot Restore](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-fast-snapshot-restore.html)

### EBS-Backed AMIs

An EBS-backed [Amazon Machine Image (AMI)](03-EC2.md#amazon-machine-images) references EBS snapshots for its disk data. Its **block device mapping** specifies the root volume and any additional volumes to attach. At launch, EC2 creates new EBS volumes from the referenced snapshots and attaches them to the new instance. [Creating an EBS-backed AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html)

## 4. EC2 Instance Store

**Instance store** provides ephemeral block storage on disks physically attached to the EC2 host. On supported instance types, it offers low latency and high disk I/O for data that can be recreated, such as caches, buffers, scratch files, and intermediate processing results.

Instance store volumes attach only at launch and stay tied to that instance. A host or disk failure can cause data loss. Because EBS snapshots only back up EBS volumes, copy or replicate any instance store data you need to keep to persistent storage. [AWS documentation on instance store persistence](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-store-lifetime.html)

For local disk capacity and performance, the **I series** offers high-performance SSDs, the **D series** offers dense storage, and **H1** is an example of high-throughput local HDD storage. [Storage-optimized instance specifications](https://docs.aws.amazon.com/ec2/latest/instancetypes/so.html)

## 5. Amazon EFS (Elastic File System)

**Amazon EFS** is a managed, POSIX-compatible network file system that multiple EC2 instances can mount concurrently. It supports NFSv4.0 and NFSv4.1 and is commonly used with Linux workloads. Capacity grows and shrinks as files are added and removed. EFS can scale to petabytes and support thousands of concurrent client connections. Aggregate throughput depends on the configuration and service quotas. [AWS documentation on EFS mounts](https://docs.aws.amazon.com/efs/latest/ug/how-it-works.html) · [EFS quotas](https://docs.aws.amazon.com/efs/latest/ug/limits.html)

EFS is useful for web serving, content management systems, and shared application directories. For example, if two WordPress instances mount the same EFS file system, an image uploaded through one instance is available to the other through the shared directory.

### Availability and Access Controls

- **Regional EFS** stores data redundantly across multiple AZs in a Region. It provides the availability and resilience typically needed for production and supports a mount target in each AZ.
- **EFS One Zone** stores data in a single AZ and uses a single mount target there. It costs less and suits development, testing, or workloads that can tolerate AZ-level data loss. Clients in another AZ can reach the mount target, with cross-AZ data transfer charges. [AWS EFS availability and mount targets](https://docs.aws.amazon.com/efs/latest/ug/how-it-works.html)

A mount target provides a network endpoint for the file system. On TCP port 2049, the EC2 security group must allow outbound NFS traffic to the mount target, and the mount target's security group must allow inbound NFS traffic from the clients. [EFS network access](https://docs.aws.amazon.com/efs/latest/ug/network-access.html)

EFS supports encryption at rest with AWS KMS, TLS for encryption in transit, and POSIX ownership and permissions for file access. [EFS features](https://docs.aws.amazon.com/efs/latest/ug/features.html)

### Performance and Throughput Modes

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

### Storage Classes and Lifecycle Management

EFS storage charges are based on the amount of data in each storage class. Throughput and data access charges depend on the selected modes and classes.

| Storage class | Use case and cost |
| --- | --- |
| EFS Standard | Frequently accessed files for active applications that need low latency. |
| EFS Infrequent Access (IA) | Occasionally accessed data with lower storage costs and data access charges. |
| EFS Archive | Data accessed a few times a year or less, with lower storage costs and data access charges. Requires Regional EFS with Elastic throughput. |

**Lifecycle management** applies policies to move files between classes. For example, a new file starts in Standard. A policy can move it to Infrequent Access after 60 days without access to reduce storage costs. Policies can also move eligible files to Archive and return files to Standard on first access. [AWS EFS storage classes](https://docs.aws.amazon.com/efs/latest/ug/features.html#storage-classes) · [EFS lifecycle policies](https://docs.aws.amazon.com/efs/latest/ug/enable-lifecycle-management.html)
