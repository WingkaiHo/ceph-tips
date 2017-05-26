1. Configure the volume plugin

```
$ sudo apt-get install -y golang librados-dev librbd-dev ceph-common xfsprogs
$ export GOPATH=$HOME
$ export PATH=$PATH:$GOPATH/bin
$ go get github.com/yp-engineering/rbd-docker-plugin
$ sudo rbd-docker-plugin -h
Usage of rbd-docker-plugin:
  -cluster="": Ceph cluster
  -config="": Ceph cluster config
  -create=false: Can auto Create RBD Images
  -fs="xfs": FS type for the created RBD Image (must have mkfs.type)
  -logdir="/var/log": Logfile directory
  -mount="/var/lib/docker/volumes": Mount directory for volumes on host
  -name="rbd": Docker plugin name for use on --volume-driver option
  -plugins="/run/docker/plugins": Docker plugin directory for socket
  -pool="rbd": Default Ceph Pool for RBD operations
  -remove=false: Can Remove (destroy) RBD Images (default: false, volume will be renamed zz_name)
  -size=20480: RBD Image size to Create (in MB) (default: 20480=20GB)
  -user="admin": Ceph user
  -version=false: Print version
```

As you see, the RBD volume plugin supports several options cluster name, user
The volume plugin has 2 different methods to provision volumes:

- manually, where you have to create the RBD image and put a filesystem on it by yourself.This is interesting when you know that the size of each volume can vary. If it does not you should probably configure the plugin to do it for you.

- automatically, where the plugin will create the image and the filesystem for you.The service can work with systemd with this unit file. For the purpose of this tutorial, I will run it through stdout.

2 Run it!
  Before starting the service, I am going to configure Ceph for it:
```
$ sudo ceph osd pool create docker 128
  pool 'docker' created
$ sudo ceph auth get-or-create client.docker mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=docker' -o /etc/ceph/ceph.client.docker.keyring
```
  I like the fact that the volume plugin can configure the volume for me so I will configure it to do so. Let’s start the service:

```
$ sudo rbd-docker-plugin --create --user=docker --pool=docker &
```
  The driver writes a socket under /run/docker/plugins/rbd.sock, this socket will be used by Docker to perform the necessary actions (create the volume, do the bindmount etc…).
