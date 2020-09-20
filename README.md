# homelab
Helm charts and other scripting/automation for home lab.

# Hardware

* pi
* [case](https://www.c4labs.com/product/8-slot-stackable-cluster-case-raspberry-pi-3b-and-other-single-board-computers-color-options/?attribute_pa_cloudlet-case-color=black-lime)
* hub
* power supplies
* SSD connector
* SSD
* Memory cards

# Set up

* download image 64-bit
* balendaetcher
* burn card
* eeprom update
* burn ssd & cards
* touch /Volumes/boot/ssh

# Configuration

## All
```bash
# CHANGE ROOT PASSWORD
sudo passwd

# Update software
sudo apt update
sudo apt full-upgrade

# Enable cgroups not enabled by default
#  You can check if needed with "cat /proc/cgroups"
echo " cgroup_memory=1 cgroup_enable=memory" | sudo tee -a /boot/cmdline.txt

# Enable legacy iptables
# https://rancher.com/docs/k3s/latest/en/advanced/#enabling-legacy-iptables-on-raspbian-buster
sudo iptables -F
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy

# Misc configuration
sudo timedatectl set-timezone America/New_York

# Use raspi-config to set the following
sudo raspi-config
#	Advanced -> Memory Split (16 MB)
#	Network -> Hostname (k3s-server, k3s-agent-1, k3s-agent-2)

sudo reboot
```

## Server

```bash
# Install k3s
#  Disable local storage because we're going to use NFS.
#  Note: attempts to leave local and set nfs as default were
#        being reset after reboot
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable local-storage" sh -
```

## Agents
```bash
# Install k3s
# On server: sudo cat /var/lib/rancher/k3s/server/node-token
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.200:6443 K3S_TOKEN=<the string from the file above> sh -
```

# Configure NFS
For now, using nfs on the server (SSD) for persistent storage
## Server
```bash
# Install nfs
sudo apt install nfs-kernel-server
mkdir /mnt/k3s-storage
sudo chown -R pi:pi /srv/k3s-storage
sudo chmod 755 /srv/k3s-storage

# Configure nfs
echo "/srv/k3s-storage *(rw,no_root_squash,insecure,async,no_subtree_check,anonuid=1000,anongid=1000)" | sudo tee -a /etc/exports

sudo exportfs -ra
```

## Agents
```bash
# Install nfs
sudo apt install nfs-common -y
```

# Kubernetes config
 Find config in /etc/rancher/k3s/k3s.yaml
	It’s readable by root only.
	To use remotely, save in \~/.kube/k3s.yaml and edit the host/IP address
    export KUBECONFIG=\~/.kube/k3s.yaml


## Persistent volumes via NFS

```bash
# Install provisioner
helm install --set nfs.server=192.168.0.200 --set nfs.path=/srv/k3s-storage --set image.repository=quay.io/external_storage/nfs-client-provisioner-arm nfs-client-provisioner stable/nfs-client-provisioner
```

## Sealed Secrets
Allows storing secrets in public git repo
```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.12.5/controller.yaml
```
Note: Extract and save the key somewhere.

## Flux
Continuous delivery with gitops
```bash
# https://docs.fluxcd.io/en/1.20.2/tutorials/get-started/
# https://github.com/raspbernetes/multi-arch-images/tree/master/build/flux

kubectl create ns flux
```

## Let's Encrypt
https for your stuff, automatically
```bash
# https://opensource.com/article/20/3/ssl-letsencrypt-k3s
```

# Applications

From this point on, everything should be deployed via gitops (FluxCD)

* Tesla Mate



