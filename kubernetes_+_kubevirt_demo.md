
# Demo KubeVirt with Kubernetes

This will demo functionality of KubeVirt using the configuration deployed in the [install][kubernetes_+kubevirt_install.md] document.

The following is shown:
* Show CRDs and Operator deployment
* (Optional) Upload a disk using a DV
* (Optional) Import a disk using a DV
* Define a virtual machine
* Start a virtual machine
* Connect to a virtual machine console
* Expose a virtual machine
* (Optional) Live migrate a virtual machine


## Demos

* Show KubeVirt CRDs
  
  ```bash
  # show the CRDs associated with KubeVirt
  k get crd | grep kubevirt

  # show the Operator(s)
  k get pod -n kubevirt
  k get pod -n cdi
  ```

* Upload a disk using a `DataVolume` and `virtctl`
  
  Before starting, port forward the service:
  
  ```bash
  # this will stay open in the terminal, make sure you have more than one
  # open or are using tmux to continue
  k port-forward -n cdi service/cdi-uploadproxy 8443:443
  ```
  
  Or, create a `NodePort` to upload:
  
  ```bash
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: Service
  metadata:
    name: cdi-uploadproxy-nodeport
    namespace: cdi
    labels:
      cdi.kubevirt.io: "cdi-uploadproxy"
  spec:
    type: NodePort
    ports:
      - port: 443
        targetPort: 8443
        nodePort: 31001
        protocol: TCP
    selector:
      cdi.kubevirt.io: cdi-uploadproxy
  EOF
  ```
  
  If using the port forward method, then the `uploadproxy-url` below will be `https://localhost:8443`.  If using the `NodePort` method, then `uploadproxy-url` will be `https://node:port`.
  
  ```bash
  # upload the image
  # this will not fork, so if you want to show the logs you'll need another
  # session. If using the port forward method above, that makes three!
  virtctl image-upload dv fedora \
    --size=10Gi \
    --insecure \
    --uploadproxy-url=https://localhost:8443 \
    --image-path=templates/Fedora-Cloud-Base-33-1.2.x86_64.qcow2
  
  # show the upload pod
  k get pod
  
  # show the import logs
  k logs <pod name>
  ```

* Import a disk using a DV
  
  Make this process faster by hosting the image locally instead of pulling from the internet.
  
  ```bash
  # create the DataVolume
  k apply -f http-dv.yaml
  
  # view dv for status
  k get dv

  # get the import pod
  k get pod
  
  # monitor logs for detailed status
  k logs -f importer-http-dv
  ```

* Define a virtual machine
  
  Two options: create a disk with a `DataVolumeTemplate` or use an existing disk.

  **Create a disk**
  ```bash
  # this assumes that you have the qcow2 hosted somewhere to download
  k create -f vm-fedora-dvt.yaml
  ```

  **Use an existing PVC**
  ```bash
  # use one of the above methods to create a PVC
  k create -f vm-fedora-pvc.yaml
  ```

* Start a virtual machine

* Connect to the VM console

* Expose a VM

* Live migrate a VM