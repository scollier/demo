[![Build Status](https://travis-ci.org/kubevirt/demo.svg?branch=master)](https://travis-ci.org/kubevirt/demo)

# KubeVirt on top of OpenShift Demo

This demo will deploy [KubeVirt](https://www.kubevirt.io) on top of [Minishift v1.17.0](https://www.openshift.org/minishift/).

#### Sudo to the root user

```
sudo -i
```

#### Enable nesting and label your node.

You may need to use software emulation.

```
oc create configmap -n kube-system kubevirt-config --from-literal debug.allowEmulation=true
```

Label your node so the virt-launcher pod can be scheduled correctly.

```
oc label node localhost kubevirt.io/schedulable=true
```

#### Log into OpenShift

```
oc login -u system:admin
```


#### Install KubeVirt

In this section, download the `kubevirt.yaml` file and explore it.  Then, apply it from the upstream github repo.

```
export VERSION=v0.7.0-alpha.2
```

Grab the kubevirt.yaml file to explore.

```
wget https://github.com/kubevirt/kubevirt/releases/download/$VERSION/kubevirt.yaml
```

Install KubeVirt
 
```
oc apply -f https://github.com/kubevirt/kubevirt/releases/download/$VERSION/kubevirt.yaml
```

Define the following policies for OpenShift.

```
oc adm policy add-scc-to-user privileged -n kube-system -z kubevirt-privileged
oc adm policy add-scc-to-user privileged -n kube-system -z kubevirt-controller
oc adm policy add-scc-to-user privileged -n kube-system -z kubevirt-apiserver
```


#### Install virtctl
This tool provides quick access to the serial and graphical ports of a VM, and handle start/stop operations.

```
curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/$VERSION/virtctl-$VERSION-linux-amd64
chmod +x virtctl
```


#### Create an Offline  VM

```
oc apply -f https://raw.githubusercontent.com/kubevirt/demo/master/manifests/vm.yaml
```


#### Manage Virtual Machines (optional):

To get a list of existing offline Virtual Machines:
```
oc get vms
oc get vms -o yaml testvm
```

To start an offline VM you can use:
```
./virtctl start testvm
```

To get a list of existing virtual machines:
```
oc get vms
oc get vms -o yaml testvm
```

To shut it down again:
```
./virtctl stop testvm
```

To delete an offline Virtual Machine:
```
oc delete vms testvm
```

#### Accessing VMs (serial console & spice)

Connect to the serial console

```
./virtctl console testvm
```

Connect to the graphical display
Note: Requires `remote-viewer` from the `virt-viewer` package.
```
./virtctl vnc testvm
```


#### Appendix:

Install kvm driver if not exists:
```
yum install -y qemu-kvm
```

 
