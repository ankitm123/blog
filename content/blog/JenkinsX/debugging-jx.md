---
title: Debugging JenkinsX
tags: ["jenkinsX", "debug"] 
date: '2020-06-22'
---
* Use `jx version --short` to get the version of the binary.
* You can use `jx diagnose` to print diagnostic information about your system and kubernetes cluster.
  * To print diagnostic information about the system use:
    ```bash
    jx diagnose version
    ```
  * For kubernetes resources like pods, deployments, configmaps etc use:
    ```bash
    jx diagnose pods
    ```
* You can enable debug logs using the `--verbose` flag after a command.
```bash
jx boot --verbose
```

* You can specify the start and stop steps for jx boot using `JX_BOOT_START_STEP` and `JX_BOOT_STOP_STEP`. For example:
```bash
JX_BOOT_START_STEP=install-jenkins-x JX_BOOT_END_STEP=install-jenkins-x jx boot --verbose
```
This will only run one boot step, and exit