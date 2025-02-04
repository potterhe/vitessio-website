---
author: 'Rameez Sajwani and Max Englander'
title: "Backup & Restore Performance"
date: 2023-05-08
slug: '2023-05-08-backup-and-restore-performance'
tags:
- Vitess
- backup
- Cluster Management
- restore
description: Performance Improvement in Backup & Restore.
---

The performance of backups and restores is a business requirement for Vitess users and an ongoing concern for Vitess maintainers. For sufficiently large databases, 
if we can't take backups fast enough, we risk missing daily SLAs in a production context. In the event we need to perform an emergency restore, it is paramount that we
can do so as fast as possible.

The performance of backups and restores is driven by a number of factors:

- The size of the data set.
- The CPU and memory on the machine.
- The I/O throughput of the machine and disk.
- The network bandwidth between the machine and backup storage, e.g. S3.
- The efficiency of the compression algorithm.

Over the last few releases, we have been making improvements to our ability to observe these factors, and experimenting with different options to improve performance.

Before we dive into the details, here is a quick overview of Vitess backup & restore design

## Overview

Backup & restore is provided by tablets managed by Vitess as pluggable interfaces to support multiple implementations. These interfaces are provided through
- [Backup Engine](https://github.com/vitessio/vitess/blob/main/go/vt/mysqlctl/backupengine.go)
- [Backup Storage](https://github.com/vitessio/vitess/blob/main/go/vt/mysqlctl/backupstorage/interface.go)

Backup engine is the technology used for generating the backup. Currently Vitess supports following engines:
- [Builtin](https://github.com/vitessio/vitess/blob/main/go/vt/mysqlctl/builtinbackupengine.go): Shutdown an instance and copy all the database files (default)
- [XtraBackup](https://github.com/vitessio/vitess/blob/main/go/vt/mysqlctl/xtrabackupengine.go): An online backup using [Percona's XtraBackup](https://www.percona.com/software/mysql-database/percona-xtrabackup)

Backup storage provides different plugins for persisting these backups. Currently Vitess has plugins for
- [File](https://github.com/vitessio/vitess/tree/main/go/vt/mysqlctl/filebackupstorage) (using a path on shared storage, e.g. an NFS mount)
- [Google Cloud Storage](https://github.com/vitessio/vitess/tree/main/go/vt/mysqlctl/gcsbackupstorage)
- [Amazon S3](https://github.com/vitessio/vitess/tree/main/go/vt/mysqlctl/s3backupstorage)
- [Ceph](https://github.com/vitessio/vitess/tree/main/go/vt/mysqlctl/cephbackupstorage)
- [Azure](https://github.com/vitessio/vitess/tree/main/go/vt/mysqlctl/azblobbackupstorage)

## Compression engines and benchmarks

Prior to [Vitess v15](https://github.com/vitessio/vitess/releases/tag/v15.0.0#support-for-additional-compressors-and-decompressors-during-backup-&-restore), backups were compressed and decompressed exclusively using the built-in `pargzip` engine, which generates gzip compatible files.

In v15, Vitess contributor [Renan Rangel](https://github.com/rvrangel) [added the ability to](https://github.com/vitessio/vitess/pull/10558):

 * specify alternate built-in compression engines
 * specify an external compression engine
 * control compression level and backup file extension

With these changes, backup/restore now supports many more options for compression and decompression.

The built-in supported compression engines are:

- pargzip (default)
- pgzip
- lz4
- zstd

Having the ability to use an external library allows you to plug-in your own compressor with an external binary, which is a more flexible setup for users. Users may have different requirements influencing which choices make more sense in their environment (e.g., optimizing for size, cpu, or memory). For some, it might be advisable to move to zstd compression, as it compresses much faster (or with less CPU), supports multithreaded compression out of the box, and its single thread decompression is easily 4x as fast as our current gzip/zlib library. But for others, using parallel compression and multiple threads could consume CPU in a way that adversely affects other parts of their environments.

Renan's contributions were based on experiments which yielded a 30% performance improvement using an external compressor rather than the built-in Vitess compression engines. We reproduced Renan's findings and made them part of the codebase in the [form of benchmarks](https://github.com/vitessio/vitess/pull/11994).

## I/O buffering

Vitess uses an I/O buffer of 2MiB when writing to disks in restores. However, until v17, there was no way to control the size of this buffer, nor to enable I/O buffering when reading data from disks in backups.

In v17, it's possible to control the size of I/O buffers used while reading and writing data to disk during backups and restores. (Currently this is limited to the built-in backup engine.)

Depending on the environment where vtbackup or other backup/restore programs are run, tuning these settings could impact I/O throughput.

### Detailed backup stats

[In v17, new detailed stats](https://vitess.io/docs/17.0/reference/backup-and-restore/metrics/) are available in vtbackup and other programs that participate in backups and restores. These stats can be used to identify the major bottlenecks in the backup and restore process, and to get visibility into how configuration or code changes impact the performance of individual factors.

Below is a human-friendly sample of these stats obtained from the backup phase of a vtbackup in an environment with:

- Roughly ~40GiB of database data.
- Vitess' built-in zstd compression engine.
- An AWS EC2 r5a.xlarge instance.
- An AWS EBS volume with 250 MiB/s of provisioned throughput.


| Phase  | Component    | Implementation | Operation        | Data (GiB) | Time (seconds) |
|--------|--------------|----------------|------------------|------------|----------------|
| Backup | BackupEngine | Builtin        | source:read      | 38.03      | 177.63         |
| Backup | BackupEngine | Builtin        | compressor:write | 38.03      | 43.042         |


In this configuration, `source:read` represents reading data from disk, and `compressor:write` is compressing that data. We can see that reading from disk is a bigger bottleneck.

Compare the stats sample above with the one below taken in the same environment with a 2MiB I/O read buffer:

- `--builtinbackup-file-read-buffer-size=2097152`


| Phase  | Component    | Implementation | Operation        | Data (GiB) | Time (seconds)                                               |
|--------|--------------|----------------|------------------|------------|--------------------------------------------------------------|
| Backup | BackupEngine | Builtin        | source:read      | 38.03      | <span style="background-color: MediumSeaGreen">138.00</span> |
| Backup | BackupEngine | Builtin        | compressor:write | 38.03      | <span style="background-color: OrangeRed">86.108</span>      |

In this experiment, it looks like we read from disk faster, but compression became a bottleneck. Let's see what happens next when we use an external compression engine:

- `--compression-engine=external`
- `--external-compressor=zstd -c -1 -T4`


| Phase  | Component    | Implementation | Operation        | Data (GiB) | Time (seconds)                                               |
|--------|--------------|----------------|------------------|------------|--------------------------------------------------------------|
| Backup | BackupEngine | Builtin        | source:read      | 38.03      | <span style="background-color: MediumSeaGreen">148.64</span> |
| Backup | BackupEngine | Builtin        | compressor:write | 38.03      | <span style="background-color: MediumSeaGreen">28.108</span> |


From the original settings, that's:

- ~35% improvement to compression performance.
- ~16% improvement to disk I/O performance.
- A ~20% improvement to net backup performance.

One observation here is that with the original configuration, we were getting ~220 MiB/s of I/O throughput. With the final configuration, we got ~260 MiB/s, which is closer to the 250 MiB/s we had provisioned in the benchmark environment. In order to make further improvements to disk I/O throughput in this environment, we would need to experiment with different hardware configurations.

## Learn more

You can learn more about backup and restore in Vitess from the [docs](https://vitess.io/docs/).
