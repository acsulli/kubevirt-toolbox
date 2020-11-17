
For the 2020-11-19 live stream of the KubeVirt office hour.

This demo will show:

* Create a VM disk by uploading with a `DataVolume`
* Create a VM which uses the `DataVolume` PVC
* Start the VM, connect to it, show functionality
* Install a simple web server to the VM, expose and consume it from another Pod
* Live migrate the VM

## Pre-reqs

* A k8s cluster with KubeVirt and CDI deployed
  * `LiveMigration` feature gate for KubeVirt enabled
* It's recommended to download the Fedora Cloud image and host it on a local webserver.  Update the `virctl image-upload` command to use the correct location.
* The CDI upload proxy needs to be exposed. I have used a `NodePort` here, adjust as needed if using an `Ingress` for port forward.
* Persistent storage
  * Does not need to be dynamically provisioned, but PVs must be available for binding
  * If using live migration, RWX PVs must be available

## Demo

I have aliased `k` to `kubectl` to save typing.

```bash
#
### create and start a VM ###
#

# show the kubevirt crds
k get crd | grep kubevirt

# create a vm disk by uploading
virtctl image-upload dv fedora-upload \
  --size=10Gi \
  --access-mode ReadWriteMany \
  --insecure \
  --uploadproxy-url=https://kube03.work.lan:31001 \
  --image-path=templates/Fedora-Cloud-Base-33-1.2.x86_64.qcow2

# show the DV and PVC
k get dv
k get pvc

# define a VM which uses the PVC
vim vm-fedora-pvc.yaml
k create -f vm-fedora-pvc.yaml

# show the VM definition
k get vm

# show the VM is not running
k describe vm
k get vmi

# start the VM
virtctl start fedora

# show the vmi
k get vmi

# connect to the console
virtctl console fedora

# or open a VNC window if remote-viewer is installed
# virtctl vnc fedora

# login
# username: fedora, password: redhat

# show sdn connectivity
ip a

# optionally disconnect if done
# ctrl + ]
```

Before beginning this section, create the helper pod: `k create -f helper.yaml`.

```bash
#
### show pod/VM connectivity ###
#

# connect to the console if not already
# virtctl console fedora

# install a webserver
sudo dnf -y install httpd

# create an index page
echo "hello world!" | sudo tee /var/www/html/index.html

# prove it's there
curl localhost

# disconnect from the VM
# ctrl + ]

# create a service for the VM
virtctl expose vmi fedora --port=80 --name=fedora-http

# show the service
k get svc

# connect to the helper pod running
k exec -it helper /bin/sh

# connect to the VM based service
curl fedora-http.default.svc
```

```bash
#
### live migrate the vm ###
#

# determine whic host the VM is on
k get vmi

# trigger a live migration
k create -f migrate.yaml
```

```bash
#
### cleanup ###
#

# remove objects
virtctl stop fedora
k delete vm fedora
k delete dv fedora
k delete svc fedora-http
```

## other

```bash
#
### import a disk from http server ###
#

# create the DV, note you must have the image hosted somewhere
k create -f http-dv.yaml

# show the DV
k get dv

# show the import progress
k logs importer-fedora-http-import
```

```bash
#
### create an all-in-one VM definition using a DataVolumeTemplate ###
#

# make sure the image is hosted somewhere accessible before creating the VM
# this will create a VM named fedora-aio
k create -f vm-fedora-dvt.yaml
```