# SNO on KVM

Single Node OpenShift (SNO) on Kernel-based Virtual Machine (KVM)

- [SNO on KVM](#sno-on-kvm)
- [Basic installation](#basic-installation)
  - [Obtain tools](#obtain-tools)
  - [Prepare ISO](#prepare-iso)
  - [Embed the ignition file into the ISO](#embed-the-ignition-file-into-the-iso)
  - [Prepare DNS](#prepare-dns)
  - [Install OpenShift on KVM](#install-openshift-on-kvm)
  - [Remove VM from Host](#remove-vm-from-host)
- [Advanced configuration options during live boot](#advanced-configuration-options-during-live-boot)
  - [Prepare HTTP webserver for ignition](#prepare-http-webserver-for-ignition)
  - [Live boot the coreOS image](#live-boot-the-coreos-image)
  - [Run the coreos-installer on live boot](#run-the-coreos-installer-on-live-boot)

# Basic installation

The SNO node IP will be `192.168.122.10`. Per default the Cluster version will be `latest-4.15` on `x86_64` CPU architecture.

## Obtain tools

The required tools are:
- OpenShift client `oc`
- OpenShift installer `openshift-install`
- CoreOS installer `coreos-installer`

```sh
OCP_VERSION=latest-4.15
ARCH=x86_64

curl -k https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$OCP_VERSION/openshift-client-linux.tar.gz -o oc.tar.gz
tar zxf oc.tar.gz
chmod +x oc
mv oc /usr/local/bin/oc
rm -rf kubectl oc.tar.gz README.md 

curl -k https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$OCP_VERSION/openshift-install-linux.tar.gz -o openshift-install-linux.tar.gz
tar zxvf openshift-install-linux.tar.gz
chmod +x openshift-install
mv openshift-install /usr/local/bin/openshift-install
rm -rf openshift-install-linux.tar.gz README.md

curl https://mirror.openshift.com/pub/openshift-v4/clients/coreos-installer/latest/coreos-installer -o coreos-installer
mv ./coreos-installer /usr/local/bin && chmod +x /usr/local/bin/coreos-installer
```

## Prepare ISO

To prepare the `rhcos-live.iso` with the `install-config.yaml` it is required to have a SSH key in `$HOME/.ssh/id_rsa.pub` and the Red Hat pull secret in `$HOME/.docker/config.json`. The modified ISO will be placed in `/var/lib/libvirt/images/rhcos-live.iso`.

```sh
ISO_URL=$(openshift-install coreos print-stream-json | grep location | grep $ARCH | grep iso | cut -d\" -f4)
curl -L $ISO_URL -o rhcos-live.iso

export PULL_SECRET=$(cat $HOME/.docker/config.json)
export SSH_KEY=$(cat $HOME/.ssh/id_rsa.pub)

mkdir sno
cat << EOF > sno/install-config.yaml
apiVersion: v1
baseDomain: local
compute:
- name: worker
  replicas: 0 
controlPlane:
  name: master
  replicas: 1 
metadata:
  name: sno
networking: 
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.122.0/24 
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
bootstrapInPlace:
  installationDisk: /dev/sda
pullSecret: '${PULL_SECRET}' 
sshKey: |
  ${SSH_KEY}
EOF

openshift-install --dir=sno create single-node-ignition-config
```

## Embed the ignition file into the ISO

```sh
coreos-installer iso ignition embed -fi sno/bootstrap-in-place-for-live-iso.ign rhcos-live.iso
mv $HOME/rhcos-live.iso /var/lib/libvirt/images/
```

## Prepare DNS

```sh
dnf install -y dnsmasq bind-utils
echo -e "[main]\ndns=dnsmasq" | sudo tee /etc/NetworkManager/conf.d/openshift.conf
echo listen-address=127.0.0.1 > /etc/NetworkManager/dnsmasq.d/openshift.conf
echo bind-interfaces >> /etc/NetworkManager/dnsmasq.d/openshift.conf
echo server=185.12.64.1 >> /etc/NetworkManager/dnsmasq.d/openshift.conf
echo server=185.12.64.2 >> /etc/NetworkManager/dnsmasq.d/openshift.conf
echo server=8.8.8.8 >> /etc/NetworkManager/dnsmasq.d/openshift.conf
echo address=/sno.local/192.168.122.10 >> /etc/NetworkManager/dnsmasq.d/openshift.conf
systemctl reload NetworkManager

# Verify with either of those two commands:
nslookup master.sno.local
dig master.sno.local +short
```

## Install OpenShift on KVM

```sh
virsh net-update default add ip-dhcp-host "<host mac='52:54:00:65:aa:da' name='master.sno.local' ip='192.168.122.10'/>" --live --config

virt-install --name="master-sno" \
    --vcpus=4 \
    --ram=32768 \
    --disk path=/var/lib/libvirt/images/master-sno.qcow2,bus=sata,size=120 \
    --network network=default,model=virtio \
    -m 52:54:00:65:aa:da \
    --boot menu=on \
    --graphics vnc --console pty,target_type=serial --noautoconsole \
    --cpu host-passthrough \
    --cdrom /var/lib/libvirt/images/rhcos-live.iso \
    --os-variant=rhel9.2

# In case you want to follow the installation process on the node
# The installation process takes around 50 minutes
virsh domifaddr master-sno
ssh core@192.168.122.10
journalctl -b -f -u release-image.service -u bootkube.service


openshift-install --dir=sno wait-for bootstrap-complete --log-level=debug
openshift-install --dir=sno wait-for install-complete --log-level=debug
export KUBECONFIG=sno/auth/kubeconfig
oc get nodes
```

## Remove VM from Host

```sh
# Stop the domain gracefully
virsh shutdown master-sno

# Remove VM without storage
virsh undefine master-sno

# Remove VM and storage
virsh undefine master-sno --remove-all-storage

Domain 'master-sno' has been undefined
Volume 'sda'(/var/lib/libvirt/images/master-sno.qcow2) removed.
```

# Advanced configuration options during live boot

**This chapter is work in progress.**
The goal is to use advanced configuration options such as [multipath](https://docs.openshift.com/container-platform/4.15/installing/installing_bare_metal/installing-bare-metal.html#rhcos-enabling-multipath_installing-bare-metal). We could take a shortcut by leveraging the `--extra-args` option of during `virt-install` similar to the [SNO installation on IBM Z](https://docs.openshift.com/container-platform/4.15/installing/installing_sno/install-sno-installing-sno.html#installing-sno-on-ibm-z-kvm_install-sno-installing-sno-with-the-assisted-installer), but that is not the goal of the exercise. 

As most of the steps are similar, I will assume that the following chapters from above are already completed:
- [Obtain tools](#obtain-tools)
- [Prepare ISO](#prepare-iso)
- [Prepare DNS](#prepare-dns)

## Prepare HTTP webserver for ignition

```sh
dnf install -y httpd
systemctl start httpd
cp sno/bootstrap-in-place-for-live-iso.ign /var/www/html/
chmod +r /var/www/html/bootstrap-in-place-for-live-iso.ign
```

## Live boot the coreOS image

Instead of [embedding the ignition file into the ISO from the previous chapter](#embed-the-ignition-file-into-the-iso) we will boot the standard ISO and modify it during live boot.

```sh
tree sno/
sno/
├── auth
│   ├── kubeadmin-password
│   └── kubeconfig
├── bootstrap-in-place-for-live-iso.ign
├── metadata.json
└── worker.ign

1 directory, 5 files

cp $HOME/rhcos-live.iso /var/lib/libvirt/images/

# In case this was not completed before
virsh net-update default add ip-dhcp-host "<host mac='52:54:00:65:aa:da' name='master.sno.local' ip='192.168.122.10'/>" --live --config

virt-install --name="master-sno-live" \
    --vcpus=4 \
    --ram=32768 \
    --disk path=/var/lib/libvirt/images/master-sno-live.qcow2,bus=sata,size=120 \
    --network network=default,model=virtio \
    -m 52:54:00:65:aa:da \
    --boot menu=on \
    --graphics vnc --console pty,target_type=serial --noautoconsole \
    --cpu host-passthrough \
    --cdrom /var/lib/libvirt/images/rhcos-live.iso \
    --os-variant=rhel9.2
```

## Run the coreos-installer on live boot

As the live boot does not have an IP available, you have to connect to the server differently. One approach would be to use `virt-viewer` and connect to the KVM server from externally. This allows you to enable the serial console on the guest/domain/VM.
```sh
virt-viewer --connect qemu+ssh://<username>@<serverIP>/system master-sno-live
sudo systemctl start serial-getty@ttyS0.service
```

After enabling the serial console, you can connect from the KVM host.
```sh
virsh console master-sno-live
```

Test if you can download the ignition from the host.
```sh
curl http://192.168.122.1:80/bootstrap-in-place-for-live-iso.ign -o bootstrap.ign
```

Run the `coreos-installer` with optional advanced configuration via `--append-karg`.

```sh
sudo coreos-installer install --ignition-url=http://192.168.122.1:80/bootstrap-in-place-for-live-iso.ign --insecure-ignition /dev/sda
> Read disk 3.6 GiB/3.6 GiB (100%)     
Writing Ignition config
Install complete.

sudo reboot

virsh start master-sno-live
ssh core@192.168.122.10
journalctl -b -f -u release-image.service -u bootkube.service

openshift-install --dir=sno wait-for bootstrap-complete --log-level=debug
openshift-install --dir=sno wait-for install-complete --log-level=debug
export KUBECONFIG=sno/auth/kubeconfig
oc get nodes
```

Stop the `httpd` server on the host and remove the ignition file.
```sh
systemctl stop httpd
rm /var/www/html/bootstrap-in-place-for-live-iso.ign
```
