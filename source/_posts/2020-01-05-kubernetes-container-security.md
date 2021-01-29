---
title: Kubernetes容器安全之Apparmor
date: 2020-01-05
tags:
    - kubernetes
---
本文简单介绍一下Apparmor在Kubernetes容器安装中的时候，由于之前在外企工作，所以用英文写的本文。

<!-- more -->

## 1. Introduction
AppArmor (Application Armor) is a Linux Security Module (LSM). It protects the operating system by applying profiles to individual applications or containers. For example, AppArmor can restrict file operations, linux capabilities, network access, file permissions, etc.
Related-Doc: https://gitlab.com/apparmor/apparmor/-/wikis/QuickProfileLanguage

## 2. Installation of apparmor
```bash
# 本文用的photon os
tdnf install -y apparmor-utils     systemctl start apparmor
systemctl restart kubelet && systemctl restart containerd
```
## 3. How to use apparmor on Pods
通过Pod Annotation来进行配置

**Pod Annotation**

Specifying the profile a container will run with:
- key: container.apparmor.security.beta.kubernetes.io/<container_name> Where <container_name> matches the name of a container in the Pod. A separate profile can be specified for each container in the Pod.
- value: a profile reference, described below

**Profile Reference**

- runtime/default: Refers to the default runtime profile.
  - Equivalent to not specifying a profile (without a PodSecurityPolicy default), except it still requires AppArmor to be enabled.
  - For Docker, this resolves to the docker-default profile for non-privileged containers, and unconfined (no profile) for privileged containers. For Containerd, this resolves to "cri-containerd.apparmor.d"
- localhost/<profile_name>: Refers to a profile loaded on the node (localhost) by name.
  - The possible profile names are detailed in the core policy reference.
- unconfined: This effectively disables AppArmor on the container.
Any other profile reference format is invalid.

Example:
```python
cat > /etc/apparmor.d/no_raw_net <<EOF
#include <tunables/global>

profile no-ping flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  network inet tcp,
  network inet udp,
  network inet icmp,

  deny network raw,
  deny network packet,
  file,
  mount,
}
EOF
```

You can use command "apparmor_parser" to load profile to kernel, just as the following show. But if you want to keep this profile after node reboot, you need to store this profile content into file under /etc/apparmor.d/. Otherwise this profile won't be loaded after node reboot.
```bash
/sbin/apparmor_parser --replace --write-cache /etc/apparmor.d/no_raw_net
```

举例分析
```
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor
  annotations:
    container.apparmor.security.beta.kubernetes.io/hello: localhost/no-ping
spec:
  containers:
  - name: hello
    image: busybox
    command: [ "sh", "-c", "echo 'Hello AppArmor!' && sleep 1h" ]
```

## 4. Default profile in Conatinerd
After apparmor installed, there is a default apparmor profile for containerd called "cri-containerd.apparmor.d". The content is as following:

Related code: https://github.com/containerd/containerd/blob/834665d9db028c8733479b5063e4fd477e549364/contrib/apparmor/template.go

```bash
#include <tunables/global>
profile cri-containerd.apparmor.d flags=(attach_disconnected,mediate_deleted) {
#include <abstractions/base>
  network,
  capability,
  file,
  umount,
  deny @{PROC}/* w,   # deny write for all files directly in /proc (not in a subdir)
  # deny write to files not in /proc/<number>/** or /proc/sys/**
  deny @{PROC}/{[^1-9],[^1-9][^0-9],[^1-9s][^0-9y][^0-9s],[^1-9][^0-9][^0-9][^0-9]*}/** w,
  deny @{PROC}/sys/[^k]** w,  # deny /proc/sys except /proc/sys/k* (effectively /proc/sys/kernel)
  deny @{PROC}/sys/kernel/{?,??,[^s][^h][^m]**} w,  # deny everything except shm* in /proc/sys/kernel/
  deny @{PROC}/sysrq-trigger rwklx,
  deny @{PROC}/mem rwklx,
  deny @{PROC}/kmem rwklx,
  deny @{PROC}/kcore rwklx,
  deny mount,
  deny /sys/[^f]*/** wklx,
  deny /sys/f[^s]*/** wklx,
  deny /sys/fs/[^c]*/** wklx,
  deny /sys/fs/c[^g]*/** wklx,
  deny /sys/fs/cg[^r]*/** wklx,
  deny /sys/firmware/** rwklx,
  deny /sys/kernel/security/** rwklx,
}
```

The workflow is as following: 

Related-code:https://github.com/containerd/cri/blob/62c91260d2f43b57fff408a9263a800b7a06a647/pkg/server/container_create_unix.go

- 1.A pod is created
- 2.Check if apparmor is enabled or not
  - 2.1 If apparmor is enabled
  Check if this is privileged container
      - 2.1.1 if it's privileged container
        do nothing, continue to create container
       - 2.1.2 if it's not privileged container
        Check the value of container.apparmor.security.beta.kubernetes.io anotation
        	- 2.1.2.1 if equals to "runtime/default" or you don't configure anotation "container.apparmor.security.beta.kubernetes.io"
return "cri-containerd.apparmor.d" profile. At this time. it will check if this profile is active on this node or not. If not. Create a new one with the template above.
It will use save the template above to a temporary file. And use command "apparmor_parser -Kr <temporary_path>" to load the profile. Finally it will remove the  temporary file. So after reboot, this profile will be lost and when container is created, this profile will be re-created automatically.
        	- 2.1.2.2 if equals “unconfined”
do nothing, continue to create container
        	- 2.1.2.3 if the format is "localhost/<profile_name>"
            Check if profile_name is exited or not on this node. If not, return error. If yes, use this profile to start container.
  - 2.2 if apparmor is not enabled
    do nothing. continue to create container

## 5. How to configure default containerd profile
If you want to manually configure "cri-containerd.apparmor.d" profile. Just create a profile under "/etc/apparmor.d" and reboot the vm.