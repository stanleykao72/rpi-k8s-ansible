# rpi-k8s-ansible
Raspberry PI's running Kubernetes deployed with Ansible

Masters:
- rPi 3b+ x1

Workers:
- rPi 3b+ x2
- rPi 3b x4

CNI: Weave (Flannel is dead, last commits 1+ years ago)

# Preparing to install
## Preparing an SD card on Linux
```
# Write the image to the SD card, please use at least 2020-02-13 if you want to use WiFi
$ sudo dd if=YYYY-MM-DD-raspbian-buster-lite.img of=/dev/sdX bs=16M status=progress

# Provision wifi settings on first boot
$ cat bootstrap/wpa_supplicant.conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=AU

network={
    ssid=""
    psk=""
    key_mgmt=WPA-PSK
}

$ cp bootstrap/wpa_supplicant.conf /mnt/boot/

# Enable SSH on first boot
$ cp bootstrap/ssh /mnt/boot/ssh
```

```
Example flash and ssh/wifi:
sudo umount /media/<user>/boot
sudo umount /media/<user>/rootfs
sudo dd if=2020-02-13-raspbian-stretch-lite.img of=/dev/<disk> bs=16M status=progress
sync

# Unplug/replug SD card

cp bootstrap/wpa_supplicant.conf /media/<user>/boot/
cp bootstrap/ssh /media/<user>/boot/

sync
sudo umount /media/<user>/boot
sudo umount /media/<user>/rootfs
```

## Updating cluster.yml to match your environment
This is where there individual rPi's are set to be a master or a slave.  
I have not changed any passwords or configured SSH keys as this cannot be easily done with a headless rPi setup.  
I am currently using DHCP static assignment to ensure each PI's MAC address is given the same IP address.  
Update the file as required for your specific setup.

# Install Kubernetes
## apt-get upgrade
```
ansible-playbook -i cluster.yml playbooks/upgrade.yml
```

## rPi Overclocks (optional)
Ensure you update cluster.yml with the correct children mappings for each rpi model
```
ansible-playbook -i cluster.yml playbooks/overclock-rpis.yml
```

## Install k8s
With the below commands, you need to include the master node (node00) in all executions for the token to be set correctly.
```
# Bootstrap the master and all slaves
ansible-playbook -i cluster.yml site.yml

# Bootstrap a single slave (node05)
ansible-playbook -i cluster.yml site.yml -l node00,node05

# When running again, feel free to ignore the common tag as this will reboot the rpi's
ansible-playbook -i cluster.yml site.yml --skip-tags common
```

## Copy over the .kube/config
This logs into node00 and copies the .kube/config file back into the local users ~/.kube/config file
Allowing a locally installed kubectl/etc to be able to query the cluster

```
ansible-playbook -i cluster.yml playbooks/copy-kube-config.yml
```

# Running things on the cluster!
You can look in the README.md in the kubernetes subfolder to see a few examples of things to run!

# Extra misc commands
```
# Shutdown all nodes
ansible -i cluster.yml -a "shutdown -h now" all

# Ensure NFS mount is active across cluster
ansible -i cluster.yml -a "mount -a" all
```

# Upgrading your cluster
First you need to upgrade the control plane (node00).
The below example is an upgrade from v1.15.0 -> v1.15.1

## Install the target kubeadm
```
sudo apt-get install kubeadm=1.15.1-00
```

## Plan the upgrade
```
sudo kubeadm upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.15.0
[upgrade/versions] kubeadm version: v1.15.1
[upgrade/versions] Latest stable version: v1.15.1
[upgrade/versions] Latest version in the v1.15 series: v1.15.1

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     7 x v1.15.0   v1.15.1

Upgrade to the latest version in the v1.15 series:

COMPONENT            CURRENT   AVAILABLE
API Server           v1.15.0   v1.15.1
Controller Manager   v1.15.0   v1.15.1
Scheduler            v1.15.0   v1.15.1
Kube Proxy           v1.15.0   v1.15.1
CoreDNS              1.3.1     1.3.1
Etcd                 3.3.10    3.3.10

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.15.1

_____________________________________________________________________
```

## Apply the upgrade
```
sudo kubeadm upgrade apply v1.15.1
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade/version] You have chosen to change the cluster version to "v1.15.1"
[upgrade/versions] Cluster version: v1.15.0
[upgrade/versions] kubeadm version: v1.15.1
[upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]: y

...

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.15.1". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

## Update kube_version
```
cat roles/kubernetes/defaults/main.yml
---
kube_version: "1.15.1-00"

<snip>
```

## Upgrade the kubeXYZ tooling
```
ansible-playbook -i cluster.yml site.yml --tags upgrade,kubernetes
```
