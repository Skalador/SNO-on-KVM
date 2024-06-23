# SNO on KVM

Single Node OpenShift (SNO) on Kernel-based Virtual Machine (KVM)

- [SNO on KVM](#sno-on-kvm)
- [Basic installation](#basic-installation)
  - [Obtain tools](#obtain-tools)
  - [Prepare ISO](#prepare-iso)
  - [Prepare DNS](#prepare-dns)
  - [Install OpenShift on KVM](#install-openshift-on-kvm)
  - [Remove VM from Host](#remove-vm-from-host)

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
journalctl -b -f -u release-image.service -u bootkube.service
ssh core@192.168.122.10

openshift-install --dir=sno wait-for bootstrap-complete --log-level=debug
openshift-install --dir=sno wait-for install-complete --log-level=debug
export KUBECONFIG=sno/auth/kubeconfig
oc get nodes
```

## Remove VM from Host

```sh
# Remove VM without storage
virsh undefine master-sno

# Remove VM and storage
virsh undefine master-sno --remove-all-storage
```

