# Elasticsearch on Ubuntu 20.04 LTS in Proxmox

This project aims to install an experimental Elasticsearch 8.x cluster in a home lab running Proxmox VE 7.x. The Kibana interface will be installed as well. 

## Prerequisite

The following assumes that DHCP is configured with reservations for the Elasticsearch cluster nodes. The following scheme is used to organize the hostnames and MAC addresses:

| Hostname        | MAC Address       |
|-----------------|-------------------|
| elasticsearch01 | 00:00:00:00:01:D1 |
| elasticsearch02 | 00:00:00:00:01:D2 |
| elasticsearch0x | 00:00:00:00:01:Dx |

Before spinning up the VM(s), create the reservations in advance so everybody has a good time.

## Proxmox Setup
The following are common variables for subsequent scripts, to be executed on a Proxmox cluster node.

```shell
pve_image_storage=/locker/images
ubuntu_cloud_image_base=https://cloud-images.ubuntu.com/focal/current
dist_image=focal-server-cloudimg-amd64-disk-kvm.img
custom_image="${dist_image::-4}-custom.img"
cloud_init_user_sshkey_local="/tmp/elasticsearch-user-sshkey"

# Fetch the latest cloud image (unless it exists)
test -r "${pve_image_storage}/${dist_image}" || curl -o "${pve_image_storage}/${dist_image}" "${ubuntu_cloud_image_base}/${dist_image}"

# Patch the image because it is missing sudo for some reason
# This assumes that `apt install libguestfs-tools` has already been performed on the PVE cluster node
if [ ! -r "${pve_image_storage}/${custom_image}" ]; then
  cp "${pve_image_storage}/${dist_image}" "${pve_image_storage}/${custom_image}"
  virt-customize -a "${pve_image_storage}/${custom_image}" --install sudo
fi

# Put the desired SSH key in place for the clout-init user; used later
curl -s -o "$cloud_init_user_sshkey_local" https://lamoree.com/joseph@lamoree.com.pub

# Create the Elasticsearch VMs as needed, incrementing the vmid, vmname, and vmmac for each
vmid=301
vmname=elasticsearch01
vmmac=00:00:00:00:01:D1
storage_sys=pool1
qm create $vmid --name $vmname --memory 8192 --cores 2 --net0 virtio,bridge=vmbr0,macaddr=$vmmac
qm importdisk $vmid "${pve_image_storage}/${custom_image}" $storage_sys
qm set $vmid --scsihw virtio-scsi-pci --scsi0 $storage_sys:vm-$vmid-disk-0
qm set $vmid --boot c --bootdisk scsi0
qm set $vmid --ide2 local-lvm:cloudinit
qm set $vmid --serial0 socket --vga serial0
qm set $vmid --agent enabled=1
qm set $vmid --ipconfig0 ip=dhcp
qm resize $vmid scsi0 16G
qm set $vmid --sshkeys "${cloud_init_user_sshkey_local}"
qm start $vmid
```

## Ansible Playbook for Elasticsearch

Add the new instance(s) to the inventory and verify SSH configuration.

Before running the Ansible Playbook, it might be a good idea to let the VM reboot and apply any updates:
```shell
ssh elasticsearch01 'sudo apt update; sudo apt upgrade -y; sudo shutdown -r now'
```

The playbook should get Elasticsearch and Kibana running:
```shell
ansible-playbook elasticsearch.yaml
```

## Services

The Elasticsearch service should now be available. Verify like so:
```shell
curl -k -u elastic https://elastic@elasticsearch01:9200/
```

The Kibana UI should be accessible at http://elasticsearch01:5601/
