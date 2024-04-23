# This repo describes how to run docker containers with ip's assigned from a local LAN
## Prerequisites
* Docker
* Docker zfs storage driver
* /var/lib/docker as the mountpoint for your zfs dataset hosting docker files
* Docker macvlan configuration

### Docker configuration
* Install docker from the docker repo
* Do not start docker
* Make your docker dataset has its mountpoint set to /var/lib/docker
```
zfs create zfs_pool/docker
zfs set mountpoint=/var/lib/docker zfs_pool/docker
```

* Make sure docker is using the zfs storage driver.  This will ensure every layer of a docker container is an ephermeral dataset. These are not mounted but are visible when zfs dataset list in invoked
* The contents of your /etc/docker/daemon.json file should look like
```
{
	  "storage-driver": "zfs"
}
```
* The output of `docker info` should contain:

```
[root@membersrv ~]# docker info | grep -A7 "Storage Driver"
 Storage Driver: zfs
  Zpool: zfs_bootes
  Zpool Health: ONLINE
  Parent Dataset: zfs_bootes/docker
  Space Used By Parent: 25159651328
  Space Available: 19515017777152
  Parent Quota: no
  Compression: lz4
```

### Create macvlan(s) for your docker containers
* Create vlan 10 on your router/switch with a virtual interface as its gateway
* On the docker host create the macvlans, you can use any interface but it needs to be connected to a switch port configured as an uplink port, with vlan ID 10 tagged

```
docker network create -d macvlan --subnet=10.0.0.0/24 --gateway=10.0.0.1 -o parent=enp5s0f0.10 macvlan10
```

* ifconfig output should display <interfacename.vlanid>

```
ifconfig | grep enp5s0f0
enp5s0f0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
enp5s0f0.10: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
enp5s0f0.20: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
```

#### Container configuration considerations
Containers, when created properly seperate configuration and data from the application or service it is providing.  This ensures persistence of configuration/data when the container is rebooted.  Since we are running zfs in this instance, we create another dataset for our docker configurations.  Use zfs snapshots to keep your configuration backed up for easy roll back

```
zfs create zfs_pool/docker-registry
zfs set mountpoint=/opt
```

* For our pihole container we create the directory /opt/docker-pi-hole
* Run the pihole container manually

```
/usr/bin/docker run --rm --net=macvlan20 --hostname='pihole' --ip=10.0.20.2 --expose 53 --expose 53/udp -p 7171:80 -v /opt/docker-pi-hole/02-lan.conf:/etc/dnsmasq.d/02-lan.conf -v /opt/docker-pi-hole/resolv.conf:/etc/resolv.conf -v /opt/docker-pi-hole/pihole/:/etc/pihole/ --name pihole  -e ADMIN_PASS=pihole -e DNS1=1.1.1.1 -e DNS2=1.0.0.1 -e ServerIP=10.0.20.2 pihole/pihole:latest
```

* To finish up place the following into systemd unit file
* docker-pihole.service

```
[Unit]
Description=pihole (Docker)
# start this unit only after docker has started
After=docker.service
Requires=docker.service
 
[Service]
TimeoutStartSec=0
Restart=always
# The following lines start with '-' because they are allowed to fail without
# causing startup to fail.
#
# Kill the old instance, if it's still running for some reason
ExecStartPre=-/usr/bin/docker kill pihole
# Remove the old instance, if it stuck around
#ExecStartPre=-/usr/bin/docker rm pihole
# Pull the latest version of the container; NOTE you should be careful to
# pull a tagged version, that way you won't accidentially pull a major-version
# upgrade and break your service!
#ExecStartPre=-/usr/bin/docker pull "pihole/pihole:latest"
# Start the actual service; note we remove the instance after it exits
# 02-lan.conf specifies /etc/pihole/lan.list for local name resolution
#
#ExecStart=/usr/bin/docker run --rm --net=macvlan20 --hostname='pihole' --ip=10.0.20.2 --expose 53 --expose 53/udp -p 7171:80 -v /opt/docker-pi-hole/02-lan.conf:/etc/dnsmasq.d/02-lan.conf -v /opt/docker-pi-hole/resolv.conf:/etc/resolv.conf -v /opt/docker-pi-hole/pihole/:/etc/pihole/ --name pihole -e ADMIN_PASS=pihole -e DNS1=1.1.1.1 -e DNS2=1.0.0.1 -e ServerIP=10.0.20.2 lplab/pihole:latest

ExecStart=/usr/bin/docker run --rm \
	--net=macvlan20 \
	--hostname='pihole' \
	--ip=10.0.20.2 \
	--expose 53 \
	--expose 53/udp -p 7171:80 \
	-v /opt/docker-pi-hole/02-lan.conf:/etc/dnsmasq.d/02-lan.conf \
	-v /opt/docker-pi-hole/resolv.conf:/etc/resolv.conf \
	-v /opt/docker-pi-hole/pihole/:/etc/pihole/ \
	--name pihole \
	-e ADMIN_PASS=pihole \
	-e DNS1=1.1.1.1 \
	-e DNS2=1.0.0.1 \
	-e ServerIP=10.0.20.2 \
	pihole/pihole:latest

# On exit, stop the container
ExecStop=/usr/bin/docker stop pihole
 
[Install]
WantedBy=multi-user.target
```


