[![Build Status](https://travis-ci.org/kubevirt/demo.svg?branch=master)](https://travis-ci.org/kubevirt/demo)

# KubeVirt on top of OpenShift Demo

### Student connection process

In this lab, we are going to leverage a process known as [`oc cluster up`](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md). `oc cluster up` leverages the local docker daemon and enables us to quickly stand up a local OpenShift Container Platform to start our evaluation. The key result of `oc cluster up` is a reliable, reproducible OpenShift environment to iterate on.

#### Find your GCP Instance
This lab is designed to accommodate many students. As a result, each student will be given a VM running on GCP. The naming convention for the lab VMs is:

**student-\<number\>**.labs.sysdeseng.com

You will be assigned a number by the instructor.

Retrieve the key from the [instructor host](https://instructor.labs.sysdeseng.com/summit/L1108.pem) so that you can _SSH_ into the instances by accessing the password protected directory. Download the _cnv.pem_ file to your local machine and change the permissions of the file to 600.

```
$ PASSWD=<password from instructor>
$ wget --no-check-certificate --user student --password ${PASSWD} https://instructor.labs.sysdeseng.com/cnv.pem
$ chmod 600 cnv.pem
```

#### Connecting to your GCP Instance
This lab should be performed on **YOUR ASSIGNED AWS INSTANCE** as `cnv-user` unless otherwise instructed.

**_NOTE_**: Please be respectful and only connect to your assigned instance. Every instance for this lab uses the same public key so you could accidentally (or with malicious intent) connect to the wrong system. If you have any issues please inform an instructor.

```
$ ssh -i cnv.pem cnv-user@student-<number>.labs.sysdeseng.com
```

#### Getting Set Up
For the sake of time, some of the required setup has already been taken care of on your GCP VM. For future reference though, the easiest way to get started is to head over to the OpenShift Origin repo on github and follow the "[Getting Started](https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md)" instructions. The instructions cover getting started on Windows, MacOS, and Linux.

All that's left to do is run OpenShift by executing the `openshift.sh` script in your home directory. First, let's take a look at what this script is doing, it's grabbing GCP instance metadata so that it can configure OpenShift to start up properly on GCP:

```
$ sudo -i
```

The remaining commands will be run as _root_ on the GCP instance. Review the `openshift.sh` file. Notice how the GCP metadata is parced to gather the external IP address to pass to `oc cluster up`. Next, notice the options that `oc cluster up` is being started with.

```
cat ~/openshift.sh
```

Now, let's start our local, containerized OpenShift environment.

```
~/openshift.sh
```

The resulting output should be something of this nature:

```
OpenShift server started.

The server is accessible via web console at:
    https://35.232.205.57.xip.io:8443

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
    <snip>
    openshift-web-console

Using project "myproject".
```

You should get a lot of feedback about the launch of OpenShift. As long as you don't get any errors you are in good shape.

OK, so now that OpenShift is available, let's ask for a cluster status & take a look at our running containers:

```
oc version
oc cluster status
```

As noted before, `oc cluster up` leverages docker for running
OpenShift. You can see that by checking out the containers and
images that are managed by docker:

```
sudo docker ps
sudo docker images
```

We can also check out the OpenShift console. Open a browser and navigate to `https://<public-hostname>:8443`. Be sure to use http*s* otherwise you will get weird web page. Once it loads (and you bypass the certificate errors), you can log in to the console using the default developer username (use any password).

#### Label your node.

Label your node so the virt-launcher pod can be scheduled correctly. Confirm the label was applied.

```
oc label node localhost kubevirt.io/schedulable=true
oc get nodes --show-labels
```

### Basic OpenShift commands

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

Enable oc bash auto-completion

```
oc completion bash >> /etc/bash_completion.d/oc_completion
source /etc/bash_completion.d/oc_completion
oc <tab> <tab>
```


#### Log into OpenShift

```
oc login -u system:admin
oc whoami
```

### Install KubeVirt

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

Review the objects that KubeVirt added.

```
oc get sa --all-namespaces | grep kubevirt
oc describe sa kubevirt-apiserver --namespace=kube-system
oc get pods --namespace=kube-system
oc describe pod virt-handler-j8gn7 --namespace=kube-system  # Replace your virt-handler pod info
oc exec -it virt-handler-j8gn7 --namespace=kube-system ls /  # review the files on the root of the filesystem of the pod, see the virt-handler executable
oc get svc --namespace=kube-system
oc describve  svc virt-api --namespace=kube-system
```

There are other services and objects to take a look at.

To review the objects through the OpenShift web console, access the console and log in as the `developer` user. To get the console URL:

```
oc cluster status | grep Web
```

Open that URL in a browser, log in as the `developer` user with any password.

![openshift](images/openshift-console-login.png)

Explore the kube-system project by clicking on the `View All` link in the right hand navigation pane.

![openshift](images/openshift-console-view-all.png)

Browse to the `kube-system` project and explore the objects. Click on the different objects, explore the environment.

![openshift](images/openshift-console-kube-system.png)

#### Install virtctl
This tool provides quick access to the serial and graphical ports of a VM, and handle start/stop operations.

```
curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/$VERSION/virtctl-$VERSION-linux-amd64
chmod -v +x virtctl
```


### Create an Offline  VM

Download the VM manifest and explore it.

```
wget https://raw.githubusercontent.com/kubevirt/demo/master/manifests/vm.yaml
less vm.yaml
```

Apply the manifest to OpenShift.

```
oc apply -f https://raw.githubusercontent.com/kubevirt/demo/master/manifests/vm.yaml
```


### Manage Virtual Machines (optional):

To get a list of existing offline Virtual Machines. Note the `running` status.

```
oc get vms
oc get vms -o yaml testvm
```

To start an offline VM you can use:

```
./virtctl start testvm
```

To get a list of existing virtual machines. Note the `running` status.

```
oc get vms
oc get vms -o yaml testvm
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

### Controlling the state of the VM

To shut it down:

```
./virtctl stop testvm
```

To delete an offline Virtual Machine:

```
oc delete vms testvm
```

#### Appendix:

Install kvm driver if not exists:
```
yum install -y qemu-kvm
```

 
