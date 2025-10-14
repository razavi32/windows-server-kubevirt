# Windows Server 2022 Migration to AWS EKS with KubeVirt

## Overview
This project documents the migration of a Windows Server 2022 VM from Hyper-V on a local laptop to AWS EKS using KubeVirt.

## Prerequisites

- An AWS account with EKS cluster deployed and kubectl access.
- KubeVirt installed and working on your EKS cluster.
- Sufficient cluster resources to run a Windows Server VM (CPU, RAM, storage).
- Admin access to your current Hyper‑V VM.
- virtctl, kubectl, and qemu-img installed where you plan to work (i.e. laptop, admin machine)
- You'll need a Storage Class that uses Immediate provisioning to create a dataVolume in step 2 below
- (Optional) Install TigerVNC or a VNC viewer to open the Windows VM console remotely

## 1. Install VirtIO Drivers in Hyper-V Before Migration
- Download VirtIO Driver ISO
    - URL: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/
- Save the ```virtio-win.iso ``` to your Windows host.
- Mount VirtIO ISO in Hyper‑V VM
    - Right click your VM > Settings
    - Click SCSI Controller
    - Click DVD Drive > Add
    - Browse to the ISO downloaded above
    - Click Apply
- Install drivers on Windows Server 2022
    - Browse to the drive that corresponds to the VirtIO ISO
    - Execute the virtio-win-guest-tools.exe 
- Confirm installation in Add or Remove Programs

## 2. Export VM from Hyper-V
- Shut down the VM to be migrated to EKS
- Export the VM to be migrated
    - Right click your VM > Export
    - Specify where to save the files
    - Click Export
- Locate the `.vhdx` file
- Convert to `qcow2` with `qemu-img` using the following command:
```bash
qemu-img convert -p -c -O qcow2 YourVM.vhdx YourVM.qcow2
```

## 3. Upload the Disk Image to Your EKS Cluster Storage
- Install Containerized Data Importer (CDI) if not already installed in your EKS cluster:
```bash
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/latest/download/cdi-operator.yaml
```
- Upload your disk image to an S3 bucket.
```bash
aws s3 cp YourVM.qcow2 s3://YourBucket/
```
- Create a presigned URL for the disk image in S3
```bash
aws s3 presign s3://YourBucket/YourVM.qcow2 --expires-in 3600
```
- Import QCOW2 into EKS as a DataVolume
    - Create a dataVolume resource pointing to the presigned URL in S3
```bash
cat <<EOF >./dv.yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: win2022-datavolume
spec:
  source:
    http:
      url: "https://your-bucket.s3.amazonaws.com/YourVM.qcow2" # This is the output from the presigned URL step above. 
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 60Gi   # must be >= size of the qcow2 image
    volumeMode: Filesystem
    storageClassName: gp3-immediate # must use immediate provisioning vs wait for consumer or else the PVC wont bind
EOF
```
    - Apply the dataVolume to the cluster
```bash
kubectl apply -f dv.yaml
```
## 4. Create a VM in the cluster
- Create KubeVirt VM manifest
```bash
cat <<EOF >./vm.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: win2022
spec:
  running: true
  template:
    spec:
      nodeSelector:
        node-type: c5a-compute
      tolerations:
      - key: "kubevirt.io/c5a-test"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      networks:
        - name: default
          pod: {}
      domain:
        cpu:
          cores: 4
        resources:
          requests:
            memory: 16Gi
        firmware:
          bootloader:
            efi:
              secureBoot: false
        devices:
          disks:
            - name: disk0
              disk:
                bus: sata
          interfaces:
            - name: default
              masquerade: {}
      volumes:
        - name: disk0
          dataVolume:
            name: win2022-datavolume
EOF
```
- Apply the manifest to the cluster
```bash
kubectl apply -f vm.yaml
```

## 5. Update the web server home page that's running in EKS
- Connect to the VM console using VNC
```bash
virtctl vnc win2022
```
- Log into the Windows Server using your admin credentials
- Open the Windows command line
- Navigate to the index.html file (C:\inetpub\wwwroot) that's being served in IIS
```powershell
cd C:\inetpub\wwwroot
```
- Edit the file using any text editor
```powershell
notepad index.html
```
- Save it using "Ctrl + S"

## 6. Expose the web server running on the VM
- Verify pod selector
```bash
kubectl get pods -l vm.kubevirt.io/name=win2022
```
- Create a Load Balancer Service YAML
```bash
cat <<EOF >./lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: win2022-web-lb
spec:
  selector:
    vm.kubevirt.io/name: win2022
  ports:
    - name: http
      port: 80
      targetPort: 80
  type: LoadBalancer
EOF
```
- Apply the Service to the cluster
```bash
kubectl apply -f lb.yaml
```
- Get the external IP of the Load Balancer
```bash
  kubectl get svc -o wide
```
- Try to reach the web server using curl or your browser
```bash
curl http://<external-ip>
```