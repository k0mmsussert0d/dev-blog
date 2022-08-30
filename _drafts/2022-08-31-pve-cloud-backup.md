---
layout: post
title: "Proxmox cloud backup"
date: 2022-08-31 00:00:00
categories:
 - blog
tags:
 - linux
 - sysop
 - proxmox
---

If you are running [Proxmox Virtual Environment \[0\]][0] in your infra, chances are that you utilize the [Backup and Restore functionality\[1\][1]. Like many other PVE features, its web GUI is devoid of a large number of controls for managing options described in the documentation.

Within this short entry I want to demonstrate setup I've came up with for **synchronizing Proxmox VMs/CTs to Backblaze B2 cloud backup** using hook scripts feature.

<!--break -->

## Prerequisites

Before proceeding, you should be in control of a [Backblaze B2 account\[2\]][2] with a configured bucket to upload dumps to. Alternatively, you should be fine using any other remote storage provided [supported by rclone\[3\]][3]. My choice of Backblaze is justified by the affordable pricing model they employ.

## Backup job hook scripts

[Hook script\[4\]][4], in terms of PVE's backups functionality, is an optional script set per-job called at various phases of the backup process. An example Perl script provided with `pve-manager` documentation enumerates the phases script might be called at:

```
job-start
job-end
job-abort
backup-start
backup-end
backup-abort
log-end
pre-stop
pre-restart
post-restart
```

Current phase is passed to the script as the parameter.

## rclone

We're on the right track with integrating custom processes into PVE backup jobs. Let's focus on preparing the process itself before *hooking it up* to the job.

Assuming that you've already created a Backblaze bucket, you should be in possession of API key and secret key.

Install `rclone` on the machine running PVE instance and run `rclone config`. Use the interactive prompt to configure the access to the bucket. Try running commands like `rclone ls <bucket-name>:` and `rclone copy <sample-file> <bucket-name>:` to verify if it works correctly. `rclone` should be capable of both reading and writing files.

## Preparing sync script

## Links
~~~
[0]: https://www.proxmox.com/en/proxmox-ve
[1]: https://pve.proxmox.com/wiki/Backup_and_Restore
[2]: https://www.backblaze.com/b2/cloud-storage.html
[3]: https://rclone.org/#providers
[4]: https://pve.proxmox.com/wiki/Backup_and_Restore#_hook_scripts
~~~