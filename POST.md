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

Most reading this may already be familiar with basic linux file permissions (the 10 bits which describe r/w/x for users and groups). If not, start [here](https://www.redhat.com/sysadmin/linux-access-control-lists) and [here](https://www.linux.com/training-tutorials/understanding-linux-file-permissions/).

And in rare cases you may need to investigate [Seccomp & AppArmor](https://security.stackexchange.com/questions/196881/docker-when-to-use-apparmor-vs-seccomp-vs-cap-drop). But that’s out of scope for today’s discussion.

Let’s talk SELinux. What is it? And what does it look like in action?

## SELinux Primer

SELinux (Security Enhanced Linux) enforces “access control”, like basic file permissions, but is more descriptive.

It lets you specify additional types of actions, and additional levels of permissions, on additional types of files.

## Example Scenario: Podman Mount

We’ll run a container and mount a directory in the container to a directory on the host.

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

Let’s check the SELinux policies

```bash
$$ ls -Z
...
drwxr-xr-x. root root system_u:object_r:container_file_t:s0:c36,c800 test
drwxrwxrwt. root root system_u:object_r:container_file_t:s0:c537,c562 tmp
drwxr-xr-x. root root system_u:object_r:container_file_t:s0:c537,c562 usr
...
```

## Conclusion

## References

- <https://docs.podman.io/en/latest/markdown/podman-run.1.html>
- <https://linuxhint.com/how-to-check-selinux-status/>
- <https://selinuxproject.org/page/NB_MLS>
