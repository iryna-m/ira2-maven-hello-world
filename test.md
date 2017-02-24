## Installing Docker 1.12.6 on CentOS 7.2

Install LVM2 toolset, that provide logical volume management facilities on Linux, and yum utilities, for manipulating repositories and extended package management.

```
$ sudo yum -y install yum-utils
$ sudo yum -y install lvm2*
```
**Remove unofficial Docker packages**

Red Hat’s operating system repositories contain an older version of Docker, with the package name *docker* instead of *docker-engine* . If you installed this version of Docker, remove it using the following command:
```
$ sudo yum -y remove docker docker-common container-selinux
```
You may also have to remove the package *docker-selinux* which conflicts with the official *docker-engine* package. Remove it with the following command:
```
$ sudo yum -y remove docker-selinux
```

Then you can install docker 1.12.6 from Nominum repository:

```
$ sudo yum -y install docker-engine-1.12.6-1.el7.centos docker-engine-selinux-1.12.6-1.el7.centos
```
### Configure logical volumes and create a thin pool

The preferred configuration for production deployments is direct-lvm. This mode uses block devices to create the thin pool. The following procedure shows you how to configure a Docker host to use the devicemapper storage driver in a direct-lvm configuration.
> **Caution**: If you have already run the Docker daemon on your Docker host and have images you want to keep, push them to docker.nominum.com before attempting this procedure.

Create physical volume:
```
pvcreate /dev/sdb
```
> You may check the results by typing *pvdisplay* .

Create virtual group:

```
vgcreate docker /dev/sdb
```
> You may check the results with *vgs* command.

Create a new logical volumes in docker virtual group:
```
lvcreate --wipesignatures y -n thinpool docker -l 95%VG
lvcreate --wipesignatures y -n thinpoolmeta docker -l 1%VG
```
Convert created logical volumes to thin pool and storage for thin pool metadata accordingly.
```
lvconvert -y --zero n -c 512K --thinpool docker/thinpool --poolmetadata docker/thinpoolmeta
```
If you received after running command
```
Cannot convert pool docker/thinpool with active volumes.
```
check the volumes with *lvdisplay*. If _LV Status_ is _available_, you need to disable logical volumes first with this oneliner:
```
for dm in /dev/mapper/docker-*; do umount $dm; dmsetup remove $dm; done
```
Then convert logical volumes once again and bring them active back with:
```
lvchange -ay docker/thinpool
lvchange -ay docker/thinpoolmeta
```
Set autoxtend settings for thinpool in docker-thinpool.profile. 
```
vi /etc/lvm/profile/docker-thinpool.profile
```

Set the limits for autoextend and a threshold for autoextend in percents:
```
activation {
thin_pool_autoextend_threshold=80
thin_pool_autoextend_percent=20
}
```
Apply the resulted lvm profile to thinpool and check that logical volume is monitored.
```
lvchange --metadataprofile docker-thinpool docker/thinpool
lvs -o+seg_monitor
```

### Setup docker service to listen for tcp connection and to use thin pool

To setup docker as a service do the following
```
sudo mkdir /etc/systemd/system/docker.service.d
sudo vi /etc/systemd/system/docker.service.d/docker.conf
```
Put in docker.conf this lines:
```
[Service]
ExecStart=
ExecStart=/usr/bin/docker  daemon  -D --tls=false -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375 --storage-driver=devicemapper --storage-opt=dm.thinpooldev=/dev/mapper/docker-thinpool --storage-opt=dm.use_deferred_removal=true --storage-opt=dm.use_deferred_deletion=true
```
Save the results and close the text editor.

Update service configuration, enable start on boot and launch the docker.service.
```
sudo systemctl daemon-reload
sudo systemctl enable docker.service
sudo systemctl start docker
```

### Troubleshooting
> If you have trouble with starting of docker.service, check the logs with _systemctl status docker -l_.

```
level=error msg="[graphdriver] prior storage driver "devicemapper" failed: devmapper: Base Device UUID and Filesystem verification failed.devicemapper: Error running deviceCreate (ActivateDevice) dm_task_run failed"
docker[32236]: time="timestamp" level=fatal msg="Error starting daemon: error initializing graphdriver: devmapper: Base Device UUID and Filesystem verification 
failed.
devicemapper: Error running deviceCreate (ActivateDevice) dm_task_run failed"
```

Means that _/var/lib/docker_ is not empty. Run _rm -rf /var/lib/docker_ and restart docker.service.

```
docker[14447]: unable to configure the Docker daemon with file /etc/docker/daemon.json: the following directives are specified both as a flag and in the configuration file: 
```

Means that there is a conflict in flags and/or configuration in _/etc/systemd/system/docker.service.d/docker.conf_  and _/etc/docker/daemon.json_. You can move or delete _daemon.json_.
```
rm /etc/docker/daemon.json  
```
or 
```
mv /etc/docker/daemon.json  /etc/ docker/daemon.json.bak
```
Then reload service configuration and restart docker.
```
systemctl daemon-reload 
systemctl restart docker
```
