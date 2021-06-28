# Debugging Linux permissions? Check your SELinux labels.


## Setup
```
vagrant up
vagrant ssh
```


## Example Scenario: Podman Mount
```
$ sudo podman run -it -v /vagrant:/test docker.io/library/centos:7
# ls
...
dr-xr-xr-x. 13 root root     0 Jun 28 02:50 sys
drwxr-xr-x.  2 1000 1000    42 Jun 28 02:50 test
drwxrwxrwt.  7 root root   132 Nov 13  2020 tmp
...
```

Why is our mounted directory owned by UID 1000 and not `root`? Firstly it may be helpful to note that UID 1000 belongs to `user vagrant`. This is shown by running (outside the container):
```
$ cat /etc/passwd
...
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
chrony:x:998:995::/var/lib/chrony:/sbin/nologin
vagrant:x:1000:1000:vagrant:/home/vagrant:/bin/bash
```

We can fix this by running:
```
$ chown -R root:root /vagrant
```

But then why does this still happen?
```
$ sudo podman run -it -v /vagrant:/test docker.io/library/centos:7
# ls -la
...
dr-xr-xr-x. 13 root root     0 Jun 28 02:50 sys
drwxr-xr-x.  2 root root    42 Jun 28 02:50 test
drwxrwxrwt.  7 root root   132 Nov 13  2020 tmp
...
# ls /test
ls: cannot open directory .: Permission denied
```

Let's check the SELinux policies
```
# ls -Z
...
drwxr-xr-x. root root system_u:object_r:container_file_t:s0:c36,c800 test
drwxrwxrwt. root root system_u:object_r:container_file_t:s0:c537,c562 tmp
drwxr-xr-x. root root system_u:object_r:container_file_t:s0:c537,c562 usr
...
```


## Fixing the mounted directory's SELinux Labels
```
$ sudo podman run -it -v /vagrant:/test:Z docker.io/library/centos:7
# ls -Z
...
drwxr-xr-x. root root system_u:object_r:container_file_t:s0:c491,c529 test
drwxrwxrwt. root root system_u:object_r:container_file_t:s0:c491,c529 tmp
drwxr-xr-x. root root system_u:object_r:container_file_t:s0:c491,c529 usr
...
# ls /test
README.md  Vagrantfile
```


## Resources
- https://docs.podman.io/en/latest/markdown/podman-run.1.html
- https://linuxhint.com/how-to-check-selinux-status/
- https://selinuxproject.org/page/NB_MLS
