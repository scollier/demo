[![Build Status](https://travis-ci.org/scollier/demo.svg?branch=master)](https://travis-ci.org/scollier/demo)

# KubeVirt on top of OpenShift Demo

## Student Connection Process

In this lab, we are going to leverage a process known as [`oc cluster up`](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md). `oc cluster up` leverages the local docker daemon and enables us to quickly stand up a local OpenShift Container Platform to start our evaluation. The key result of `oc cluster up` is a reliable, reproducible OpenShift environment to iterate on.

### Find your GCP Instance
This lab is designed to accommodate many students. As a result, each student will be given a VM running on GCP with nested virtualization, 4 vcpus and 12Gb of RAM. The naming convention for the lab VMs is:

**student\<number\>**.cnvlab.gce.sysdeseng.com

You will be assigned a number by the instructor.

Retrieve the keys from the [instructor host](http://cnv-tlv-web-svr.e2e.bos.redhat.com/cnv_rsa) so that you can _SSH_ into the instances by accessing the password protected directory. Download the `cnv_rsa`  file to your local machine and change the permissions of the file to 600. This web server is not public. You must be signed into the Red Hat VPN to access the key. Let an instructor know if you have any questions here.

```
wget http://cnv-tlv-web-svr.e2e.bos.redhat.com/cnv_rsa
chmod 600 cnv_rsa
```

### Connecting to your GCP Instance
This lab should be performed on **YOUR ASSIGNED GCP INSTANCE** as `cnv` user unless otherwise instructed.

**_NOTE_**: Please be respectful and only connect to your assigned instance. Every instance for this lab uses the same public key so you could accidentally (or with malicious intent) connect to the wrong system. If you have any issues, please inform an instructor.

```
ssh -i cnv_rsa cnv@student<number>.cnvlab.gce.sysdeseng.com
```

## Getting Set Up


For the sake of time, some of the required setup has already been taken care of on your GCP VM. For future reference though, the easiest way to get started is to head over to the OpenShift Origin repo on github and follow the "[Getting Started](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md)" instructions. The instructions cover getting started on Windows, MacOS, and Linux.

### Requirements 

First, let's escalate privileges. The remaining commands will be run as _root_ on the GCP instance.

```
sudo -i
```

Create an SSH key that you will be using later.

```
ssh-keygen -t rsa -N '' -f /root/.ssh/id_rsa
```

Review the `requirements.sh` file. This script does the following:

- install *docker*, enable and start it as a service
- install *oc* and *kubectl* client
- pull relevant images

As GCP instances might randomly experience network issues at boot, double check requirements by relaunching the script. It will run missing steps if needed.

```
~/requirements.sh
```

### Openshift

All that's left to do is run OpenShift by executing the `openshift.sh` script in root home directory.

The remaining commands will be run as _root_ on the GCP instance. Review the `openshift.sh` file. Notice how additional flags are passed to `oc cluster up` to enable additional features like ansible service broker and to allow public access.

```
cat ~/openshift.sh
```

Now, let's start our local, containerized OpenShift environment. Running this script could take between 3-5 minutes.

```
~/openshift.sh
```

The resulting output should be something of this nature:

```
OpenShift server started.

The server is accessible via web console at:
    https://student002.cnvlab.gce.sysdeseng.com:8443

You are logged in as:
    User:     developer
    Password: <any value>

To login as administrator:
    oc login -u system:admin

Logged into "https://127.0.0.1:8443" as "system:admin" using existing credentials.

You have access to the following projects and can switch between them with 'oc project <projectname>':

    default
    kube-dns
    kube-proxy
    kube-public
    kube-service-catalog
    kube-system
  * myproject
    openshift
    openshift-apiserver
    openshift-automation-service-broker
    openshift-controller-manager
    openshift-core-operators
    openshift-infra
    openshift-node
    openshift-web-console

Using project "myproject".
origin
cluster role "cluster-admin" added: "developer"
```

The expected results here are that you recieved quite a bit of output. If you received something other than what is reported above, notify an instructor.

OK, so now that OpenShift is available, let's ask for a cluster status & take a look at our running containers:

```
oc version
oc v3.10.0-rc.0+c20e215
kubernetes v1.10.0+b81c8f8
features: Basic-Auth GSSAPI Kerberos SPNEGO

Server https://127.0.0.1:8443
openshift v3.10.0-rc.0+ad6a1da-30
kubernetes v1.10.0+b81c8f8
```

The important item from the `oc cluster status` command output is the `Web console URL`.

```
oc cluster status
Web console URL: https://student002.cnvlab.gce.sysdeseng.com:8443/console/

Config is at host directory 
Volumes are at host directory 
Persistent volumes are at host directory /root/openshift.local.clusterup/openshift.local.pv
Data will be discarded when cluster is destroyed
```

Also note how we are using local paths to provide storage

```
oc get pv 
oc get pv pv0001 -o yaml
```


As noted before, `oc cluster up` leverages docker for running
OpenShift. You can see that by checking out the containers and
images that are managed by docker:

```
docker ps
docker images
```

### Label your Node.

Label your node so the virt-launcher pod can be scheduled correctly. Confirm the label was applied.

```
oc label node localhost kubevirt.io/schedulable=true
oc get nodes --show-labels
```

## Basic OpenShift Commands

Common OpenShift commands can be found below. There are quite a few more though, so be sure to refer to the [OpenShift CLI reference documentation](https://docs.openshift.org/latest/cli_reference/basic_cli_operations.html#cli-reference-basic-cli-operations). Descriptions for the following commands can be found in the CLI guide too.

```
oc
oc whoami
oc projects
oc project default
oc config view
oc get all
oc status
oc get pods
oc describe pod <pod id>
oc logs -f router-<rest of pod id>
oc types
oc logout
```

Enable oc bash auto-completion. Try to use tab completion with the `oc` command before and after the next activity.

```
oc # Try tabbing for auto-completion now
oc completion bash >> /etc/bash_completion.d/oc_completion
source /etc/bash_completion.d/oc_completion
oc # Try tabbing for auto-completion now
```


#### Log into OpenShift

```
oc login -u system:admin
oc whoami
```

### Deploy a Container-based Application to OpenShift

The purpose of this section is to deploy an example application on top of OpenShift and demonstrate how containers and virtual machines can be orchestrated side by side. We are going to use the existing project `myproject`.

#### Move to Project and Add Template

```
oc project myproject
oc create -f https://raw.githubusercontent.com/scollier/demo/training/manifests/app-template.yaml
```


#### Deploy Application

The command below will deploy the application [ara](https://github.com/openstack/ara).

```
oc new-app --template ara
```
The following objects will be created:

- BuildConfig
- ImageStream
- DeploymentConfig
- Route
- Service


#### Review Objects


Let's show our `BuildConfig` and watch the container build log.

```
oc get bc
oc logs -f bc/ara
```

Once that is complete we can confirm that our `DeploymentConfig` has rolled out.


```
oc get dc
oc rollout status dc/ara
```

Now let's take a look at the `Service` and `Route`.

```
oc get svc
oc describe svc ara
oc get route
oc describe route ara
```

## Install KubeVirt

In this section, download the `kubevirt.yaml` file and explore it.  Then, apply it from the upstream github repo.

```
export VERSION=v0.7.0-alpha.2
```

Grab the kubevirt.yaml file to explore. Review the ClusterRole's, CRDs, ServiceAccounts, DaemonSets, Deployments, and Services.

```
wget https://github.com/kubevirt/kubevirt/releases/download/$VERSION/kubevirt.yaml
less kubevirt.yaml
```

Install KubeVirt. You should see several objects were created.
 
```
oc apply -f https://github.com/kubevirt/kubevirt/releases/download/$VERSION/kubevirt.yaml
```

Define the following policies for OpenShift.

```
oc adm policy add-scc-to-user privileged -n kube-system -z kubevirt-privileged
oc adm policy add-scc-to-user privileged -n kube-system -z kubevirt-controller
oc adm policy add-scc-to-user privileged -n kube-system -z kubevirt-apiserver
```

Give permissions to the qemu user for persistent volume claims 

```
setfacl -m user:107:rwx /root/openshift.local.clusterup/openshift.local.pv/pv*
```


Review the objects that KubeVirt added.

```
oc get sa --all-namespaces | grep kubevirt
oc describe sa kubevirt-apiserver --namespace=kube-system # Please feel free to explore the other objects as well. Get a feel for the expected output.
oc get pods --namespace=kube-system
oc describe pod -l kubevirt.io=virt-handler --namespace=kube-system
# review the files on the root of the filesystem of the pod, see the virt-handler executable, after replacing the pod name with yours
oc exec -it virt-handler-n9pxj --namespace=kube-system ls /  
oc get svc --namespace=kube-system
oc describe  svc virt-api --namespace=kube-system
```

There are other services and objects to take a look at.

To review the objects through the OpenShift web console, access the console and log in as the `developer` user at `https://student<number>.cnvlab.gce.sysdeseng.com:8443`

Open that URL in a browser, log in as the `developer` user with a password of `developer`.

![openshift](images/openshift-console-login.png)

Explore the kube-system project by clicking on the `View All` link in the right hand navigation pane.

![openshift](images/openshift-console-view-all.png)

Browse to the `kube-system` project and explore the objects. Click on the different objects, explore the environment.

![openshift](images/openshift-console-kube-system.png)

#### Install virtctl

Return to the CLI and install virtctl. This tool provides quick access to the serial and graphical ports of a VM, and handle start/stop operations. Also run `virtctl` to get an idea of it's options.

```
curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/$VERSION/virtctl-$VERSION-linux-amd64
chmod -v +x virtctl
./virtctl --help
```

## Use KubeVirt


### Create a Virtual Machine

Download the VM manifest and explore it. Note it uses a registry disk and as such doesn't persist data. Such registry disks currently exist for alpine, cirros and fedora.

```
wget https://raw.githubusercontent.com/kubevirt/demo/master/manifests/vm.yaml
less vm.yaml
```

Apply the manifest to OpenShift.

```
oc apply -f https://raw.githubusercontent.com/kubevirt/demo/master/manifests/vm.yaml
```


### Manage Virtual Machines (optional):

To get a list of existing Virtual Machines. Note the `running` status.

```
oc get vms
oc get vms -o yaml testvm
```

To start a Virtual Machine you can use:

```
./virtctl start testvm
```

To get a list of existing Virtual Machines. Note the `running` status.

```
oc get vms
oc get vms -o yaml testvm
```

### Accessing VMs (serial console & spice)

Connect to the serial console of the Cirros VM. Then disconnect from the virtual machine console `ctrl+]`.

```
./virtctl console testvm
```

### Communication Between Application and Virtual Machine

While in the console of the `testvm` let's run `curl` to confirm our virtual machine
can access the `Service` of the application deployment.
```
curl ara.myproject.svc.cluster.local:8080
```

Connect to the graphical display.

Note: Requires `remote-viewer` from the `virt-viewer` package. This is out of scope for this lab. 

```
./virtctl vnc testvm
```

### Controlling the State of the VM

To shut it down:

```
./virtctl stop testvm
```

To delete an offline Virtual Machine:

```
oc delete vms testvm
```

## Experiment with CDI

[CDI](https://github.com/kubevirt/containerized-data-importer) is an utility designed to import Virtual Machine images for use with Kubevirt. 

At a high level, a persistent volume claim (PVC) is created. A custom controller watches for importer specific claims, and when discovered, starts an import process to create a raw image named *disk.img* with the desired content into the associated PVC 

#### Install CDI

to install the components, we will execute `cdi.sh` script in root home directory. Be sure to review the contents of this file first

```
~/cdi.sh
```

Review the objects that were added.

```
oc get project| grep golden
oc get pods --namespace=golden-images
```

#### Use CDI

As an example, we will import a fedora28 cloud image as a pvc and launch a virtual machine making use of it

```
oc project myproject
oc create -f pvc_fedora.yml
```

This will create the pvc with a proper annotation so that CDI controller detects it and launches an importer pod to gather the image specified in the *kubevirt.io/storage.import.endpoint* annotation

```
oc get pvc fedora -o yaml
oc get pod
# replace with your importer pod name
oc logs importer-fedora-pnbqh
```

Once the importer pod completes, this PVC is ready for use in kubevirt.

Let us create a vm making use of it. Note that we priorly change the yaml definition of this vm to inject the default public key of root user in the GCP vm.


```
PUBKEY=`cat ~/.ssh/id_rsa.pub`
sed -i "s%ssh-rsa.*%$PUBKEY%" vm1_pvc.yml
oc create -f vm1_pvc.yml
```

This will create and start a vm named vm1. We can use the following command to check our vm is running and to gather its ip.

```
oc get pod -o wide
```

Since we are running an all in one setup, the corresponding VM is actually running on the same node, we can check its qemu process

```
ps -ef | grep qemu | grep vm1
```

Finally, use the gathered ip to connect to the vm, create some files, stop and restart the vm with virtctl and check how data persists

```
ssh fedora@VM_IP
```

## Conclusion

Now that you have completed the lab, lets review what was covered here -

* Deployed OCP with `oc cluster up`
* Explored the environment and some basic OpenShift commands
* Deployed an application on OpenShift
* Deployed and explored KubeVirt
* Explored OCP web console
* Deployed a virtual machine on OCP
* Accessed the virtual machine
* Deployed CDI

Thanks for participating and don't forget to submit that PR if you find issues.




