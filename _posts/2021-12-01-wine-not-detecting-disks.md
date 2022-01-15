---
layout: post
title: "Wine not detecting drives in winecfg"
date: 2021-12-02 00:00:00
categories:
 - blog
tags:
 - linux
---

Just wanted to drop this quick fix for anyone who runs into it the same issue as the viable solution is not trivial to find. My two evenings full of efforts prove it.

When I run `winecfg` and try to go into `Drives` tab to make a change in the config, there's an error message saying:
```
Failed to connect to the mount manager, the drive configuration cannot be edited.
```
<!--break-->

Looking at stdout of the process, this log entry can be spotted:
```
0009:err:winecfg:open_mountmgr failed to open mount manager err 2
```

Several topics on WineHQ forums provide no insight into the cause of the problem, besides fact that it predominantly occurs on 64-bit systems running 32-bit Wine prefixes.

The solution I found is to run this command:
```
wineboot --update
```

Alerts about uninstalled Mono or other dependencies can be safely ignored. After the command finished running, I was more than happy to run `winecfg` and see drives configuration in its right place.

![winecfg 'Drives' tab]( {{ site.baseurl }}images/2021/winecfgdrives.png)