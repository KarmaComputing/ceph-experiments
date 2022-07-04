# Create a ceph cluster in 20 minutes

> *Insecure by design* Demonstration only.

- understand the components of ceph by manually building a cluster
- create and mount a CephFS filesystem
- Add/remove disks
- Add/remove hosts

# 1. Create 3 vultr VPS servers

ubuntu 20 LTS

or use this script: https://github.com/KarmaComputing/high-availability-web-services/blob/8-storage/vultr/vultr-create-n-servers.sh

```
./vultr/vultr-create-n-servers.sh 3 vhp-1c-2gb-intel lhr
```
> DO NOT USE THE delete servers script



# 2. Once you have at least 3 servers, install ceph requirements on every host

- podman
- cephadm

```
apt-get update -y
apt-get install curl wget gnupg2 -y

source /etc/os-release
sh -c "echo 'deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /' > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list"

wget -nv https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/xUbuntu_${VERSION_ID}/Release.key -O- | apt-key add -

apt-get update -qq -y
apt-get -qq --yes install podman

podman --version
```


### install cephadm

```
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm

chmod +x cephadm


./cephadm add-repo --release quincy

./cephadm install

cephadm install ceph-common

ceph -v
```


# 3. Bootstrap the first ceph host

```
cephadm bootstrap --mon-ip *<mon-ip>*
```
Remember to copy the dashboard login/password to your password manager.

```
ceph status
```

### so cephadm can setup between hosts:


ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-b

(to every other vps)

### Adding hosts

From the first host:


```
ceph orch host add ceph-b <host-ip> _admin
```

e.g. 
```
ceph orch host add ceph-b <host-ip> _admin
```

### list hosts

```
root@ceph-a:~# ceph orch host ls
HOST    ADDR             LABELS  STATUS  
ceph-a  <ip>  _admin          
ceph-b  <ip>   _admin          
2 hosts in cluster
```

# 4. Add monitors

> Read about what ceph monitor are.

Manual management of monitor placement

Put in manual magement mode:
```
ceph orch apply mon --unmanaged
```

Add each monitor: (every host has 1 monitor in our setup)
```
ceph orch daemon add mon *<host1:ip-or-network1/32>
```

e.g.
```
root@ceph-a:~# ceph orch daemon add mon ceph-b:<ip-addr>/32
Deployed mon.ceph-b on host 'ceph-b'

root@ceph-a:~# ceph orch daemon add mon ceph-c:<ip-addr>/32
Deployed mon.ceph-c on host 'ceph-c'
```

# 5. Adding OSDs from disks


Add a volume (block) to each of the servers and attach it, without any formatting.

This tells ceph to 'look' for blank disks/volumes and will 
automatically start using them/adding them to the storage
pool.
```
ceph orch device ls
ceph orch apply osd --all-available-devices
```

### Create a CephFS
https://docs.ceph.com/en/latest/cephadm/services/mds/#orchestrator-cli-cephfs

```
ceph fs volume create myfs
```

### Access web dashboard

	     URL: https://41dbb9cb-9c1a-4682-a9de-f8a14ec52488:8443/
	    User: admin
	Password: pmpdcsiwsp




# 6. connecting a ceph client ubuntu host

src: https://docs.ceph.com/en/quincy/cephfs/mount-prerequisites/

```
# on client host setup mount point

Client needs to understand ceph:

```
apt install ceph-common
```

Create authentication so client can mount the Cephfs

```
mkdir -p -m 755 /etc/ceph
ssh root@<ip> "sudo ceph config generate-minimal-conf" | sudo tee /etc/ceph/ceph.conf
```

perms
```
chmod 644 /etc/ceph/ceph.conf
```

Create a CephX user and get its secret key:


```
ssh root@<ip> "sudo ceph fs authorize myfs client.ceph-client-a / rw" | sudo tee /etc/ceph/ceph.client.ceph-client-a.keyring
```

### Mount the cephfs
successful mount example:

```
mount -t ceph <ip>:/ /mnt/myfs/ -o name=ceph-client-a,secret=sfgdhjhgsfsgdhfjg==
```


# Notes 

### Add another host steps

- install podman
- update ssh config on an admin/all nodes
- copy ssh key
- copy ceph key:
	ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-x
- go onto another existing ceph _admin node and add new host

### Useful

list hosts:
root@ceph-a:~# ceph orch host ls

ceph orch host add <host-name> <host-ip> _admin
e.g.
ceph orch host add ceph-d <ip> _admin

add monitor on new hosts if want to

e.g. (from another already live ceph node)
ceph orch daemon add mon ceph-d:<ip>/32



# Observations
Highest IOps seen on Vultr: 243 input/outs 

Adding a disk automatically increases avalable storage,




# Clear crashes
src https://forum.proxmox.com/threads/health_warn-1-daemons-have-recently-crashed.63105/


ceph crash ls

# to fix

Jul  3 16:40:17 ceph-d ceph-mon[47267]: Filtered out host ceph-b: does not belong to mon public_network (<ip>/23)
Jul  3 16:40:17 ceph-d ceph-mon[47267]: Filtered out host ceph-c: does not belong to mon public_network (<ip>/23)

Turn off managed mode.  
https://tracker.ceph.com/issues/52881

