---
title: Debugging Linux permissions? Check your SELinux labels
date: 2021-08-01T00:00:00-00:00
draft: true
---

# Debugging Linux permissions? Check your SELinux labels

In short, if you’re running into permission issues, begin debugging in the following order:

1. File permissions
2. SELinux labels
3. Seccomp & AppArmor

Most reading this may already be familiar with basic linux file permissions (the 10 bits which describe r/w/x for users and groups). If not, start [here](https://www.linux.com/training-tutorials/understanding-linux-file-permissions/).

And in rare cases you may need to investigate [Seccomp & AppArmor](https://security.stackexchange.com/questions/196881/docker-when-to-use-apparmor-vs-seccomp-vs-cap-drop). But that’s out of scope for today’s discussion.

Let’s talk SELinux. What is it? And what does it look like in action?

## SELinux Primer

SELinux (Security Enhanced Linux) enforces security policies just like the basic "File Permissions", but is much more expansive.

For example, SELinux can:

- define sensitivity levels/categories for files
- confine a set of processes and limit them to minimum privileges
- confine user login sessions

Even if you don't ever need to define your own [SELinux Policies](https://selinuxproject.org/page/PolicyLanguage), you may still need to overcome its hurdles someday. Let's use an example scenario.

## Example Scenario: Podman Mount

### The Problem

Let's run a container. And mount its directory to a host directory.

```bash
$ sudo podman run -it -v /vagrant:/test docker.io/library/centos:7
$$ ls
...
dr-xr-xr-x. 13 root root     0 Jun 28 02:50 sys
drwxr-xr-x.  2 1000 1000    42 Jun 28 02:50 test
drwxrwxrwt.  7 root root   132 Nov 13  2020 tmp
...
```

First issue. `test` is owned by UID 1000 and GID 1000. Running the following (outside the container) will show that this is owned by `user vagrant`

```bash
$ cat /etc/passwd
...
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
chrony:x:998:995::/var/lib/chrony:/sbin/nologin
vagrant:x:1000:1000:vagrant:/home/vagrant:/bin/bash
```

### Failed Solution

Quick fix:

```bash
$ chown -R root:root /vagrant
```

But then why does this still happen?

```bash
$ sudo podman run -it -v /vagrant:/test docker.io/library/centos:7
$$ ls -la
...
dr-xr-xr-x. 13 root root     0 Jun 28 02:50 sys
drwxr-xr-x.  2 root root    42 Jun 28 02:50 test
drwxrwxrwt.  7 root root   132 Nov 13  2020 tmp
...
$$ ls /test
ls: cannot open directory .: Permission denied
```

### SELinux

Let’s check the SELinux labels

```bash
$$ ls -Z
...
drwxr-xr-x. root root system_u:object_r:container_file_t:s0:c36,c800 test
drwxrwxrwt. root root system_u:object_r:container_file_t:s0:c537,c562 tmp
drwxr-xr-x. root root system_u:object_r:container_file_t:s0:c537,c562 usr
...
```

A breakdown of what the SELinux label for the `test` directory means:

```bash
# system_u          = user
# object_r          = role
# container_file    = type
# s0                = sensitivity
# c36,c800          = category
system_u:object_r:container_file_t:s0:c36,c800
```

All files in this directory are of sensitivity `s0` (lowest). However the `test` directory is the only file that belongs to the categories `c36` and `c800`. In other words, SELinux has compartmentalized the host directory `/vagrant` such that it is not accessible as `/test` within the container.

### Solution

To fix the problem, we need to use the `:Z` option when defining the volume mount. This tells podman to relabel the container's files to match that of the host volume.

```bash
$ sudo podman run -it -v /vagrant:/test:Z docker.io/library/centos:7
$$ ls -Z
...
drwxr-xr-x. root root system_u:object_r:container_file_t:s0:c491,c529 test
drwxrwxrwt. root root system_u:object_r:container_file_t:s0:c491,c529 tmp
drwxr-xr-x. root root system_u:object_r:container_file_t:s0:c491,c529 usr
...
$$ ls /test
README.md  Vagrantfile
```

## References

- <https://selinuxproject.org/page/NB_MLS>
- <https://linuxhint.com/how-to-check-selinux-status/>
- <https://docs.podman.io/en/latest/markdown/podman-run.1.html>
