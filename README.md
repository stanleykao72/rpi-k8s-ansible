# rpi-k8s-ansible
Raspberry PI's running Kubernetes deployed with Ansible

Masters:
- rPi 3b+ x1

Workers:
- rPi 3b+ x2
- rPi 3b x4

CNI: Flannel (Weave support is there, but it crashes and reboot's the rPi's, see below)

# Preparing to install
## Preparing an SD card on Linux
```
# Write the image to the SD card, please use at least 2018-04-18 if you want to use WiFi
$ sudo dd if=YYYY-MM-DD-raspbian-stretch-lite.img of=/dev/sdX bs=16M status=progress

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
sudo dd if=2018-11-13-raspbian-stretch-lite.img of=/dev/<disk> bs=16M status=progress
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

Using Weave as the k8s CNI resulted in quite a few kernel oops and the rPi's rebooting:
```
pi@node00:~ $ kubectl nodes get
 kernel:[  152.913108] Internal error: Oops: 80000007 [#1] SMP ARM
 kernel:[  152.928828] Process weaver (pid: 4515, stack limit = 0x90266210)
 kernel:[  152.929514] Stack: (0x902679f0 to 0x90268000)
 kernel:[  152.930180] 79e0:                                     00000000 00000000 3d3aa8c0 90267a88
 kernel:[  152.931470] 7a00: 0000801a 0000cbb6 a8b538d0 a8b53898 90267d2c 7f75ead0 00000001 90267a5c
```

See https://gist.github.com/alexellis/fdbc90de7691a1b9edb545c17da2d975 for more discussion.

Instead I've decided to move to Flannel, which is working nicely so far.

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
