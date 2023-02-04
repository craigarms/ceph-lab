# TP Ceph

Install and configure a distributed self healing File system.

Replace IPs in this guide by the IP that your Hypervisor assigns to the VMs

Interesting Talk: https://www.youtube.com/watch?v=q1fRalyix_Y

## 1. Environment Prerequisits

 - Ceph Nodes: Install 3 Ubuntu Server LTS VM with 3 HDD each
   - Install Minimal Server with OpenSSH and docker packages
 - iSCSI Initiator: Install 1 Windows VM with HyperV
 - iSCSI Gateway: Install 1 Centos >= 7.5

## 2. Prepare the Ceph nodes

- Do updates
- Do auto-remove on all hosts

Configure name resolution in the servers host files
```			
	vim /etc/hosts
		192.168.65.132   ceph1.micsi.local        ceph1
		192.168.65.133   ceph2.micsi.local        ceph2
		192.168.65.134   ceph3.micsi.local        ceph3
```

Install NFS Package
```
	apt-get install nfs-common
```
	
Do all hosts above first then these steps

## 3. Installing Ceph
Make sure ntp is working on the host and install if it isn’t
``` 
cat /etc/ntp/ntp.conf
```
if nothing shows run the below

```
apt-get install ntp
systemctl start ntp.service
systemctl enable ntp.service
```

Install cephadm using the standalone script

		curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm

		chmod +x cephadm
		./cephadm add-repo --release quincy

		apt-get update
		./cephadm install
		cephadm install ceph-common
		
Or install using the distribution's package manager, the standalone script doesn't yet work for Ubuntu 22.04

	apt install cephadm -y

Confirm that cephadm is now in your PATH by running which:

	which cephadm

A successful which cephadm command will return this:

	/usr/sbin/cephadm

## 4. Boostraping the New Cluster

The first step in creating a new Ceph cluster is running the cephadm bootstrap command on the Ceph cluster’s first host. The act of running the cephadm bootstrap command on the Ceph cluster’s first host creates the Ceph cluster’s first “monitor daemon”, and that monitor daemon needs an IP address. 

	cephadm bootstrap --mon-ip 192.168.65.132

Now push to other hosts from lead host

	ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph2 
	ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph3
	ceph orch host add ceph2
	ceph orch host add ceph3

		
Now setup monitors

		ceph config set mon public_network 192.168.65.0/24
		ceph orch apply mon ceph1,ceph2,ceph3
		ceph orch host label add ceph1 mon
		ceph orch host label add ceph2 mon
		ceph orch host label add ceph3 mon
	
To add storage to the cluster, you can tell Ceph to consume any available and unused device(s):

	ceph orch apply osd --all-available-devices
	
	ceph orch device zap ceph1 /dev/sdb --force
	ceph orch device zap ceph1 /dev/sdc --force
	ceph orch device zap ceph1 /dev/sdd --force
	ceph orch daemon add osd ceph1:/dev/sdb
	ceph orch daemon add osd ceph1:/dev/sdc
	ceph orch daemon add osd ceph1:/dev/sdd
	ceph orch device zap ceph2 /dev/sdb --force
	ceph orch device zap ceph2 /dev/sdc --force
	ceph orch device zap ceph2 /dev/sdd --force
	ceph orch daemon add osd ceph2:/dev/sdb
	ceph orch daemon add osd ceph2:/dev/sdc
	ceph orch daemon add osd ceph2:/dev/sdd
	ceph orch device zap ceph1 /dev/sdb --force
	ceph orch device zap ceph1 /dev/sdc --force
	ceph orch device zap ceph1 /dev/sdd --force
	ceph orch daemon add osd ceph1:/dev/sdb
	ceph orch daemon add osd ceph1:/dev/sdc
	ceph orch daemon add osd ceph1:/dev/sdd
	
		
## 5. Create an iSCSI target

https://docs.ceph.com/en/latest/rbd/iscsi-target-cli/

## 6. Connect to the iSCSI LUN

From the Windows server add the iSCSI Target through the iSCSI Initiator setup wizard
