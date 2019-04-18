# CIFS/SMB FlexVolume driver for Kubernetes - Tested on Pivotal Container Service (PKS) - Preview
 - supported Kubernetes version: available from v1.12 (PKS version 1.3)
 - supported agent OS: Linux 

# About
This driver allows Kubernetes to access SMB server by using [CIFS/SMB](https://en.wikipedia.org/wiki/Server_Message_Block) protocol. It was adapted from the work done by Microsoft for AKS on Azure. https://github.com/Azure/kubernetes-volume-drivers.
The main limitation for supporting this on PKS was the fact that `jq` and `cifs-utils` were needed on be installed on each worker node. However, in PKS the worker nodes are managed by BOSH and can be recreated at any point. `cifs` is already a part of the default worker node, but `jq` is not. This fork adds `jq` along with the driver to that no additional installations are required.

### Latest Container Image:
`odedia/smb-flexvolume:latest`

# Install smb FlexVolume driver on a kubernetes cluster

## 1. install smb FlexVolume driver on every agent VM via Daemon Set

create daemonset to install smb driver
```
kubectl apply -f https://raw.githubusercontent.com/odedia/kubernetes-volume-drivers/master/flexvolume/smb/deployment/smb-flexvol-installer.yaml
```
 - check daemonset status:
```
watch kubectl describe daemonset smb-flexvol-installer --namespace=kube-system
watch kubectl get po --namespace=kube-system -o wide
```

# Basic Usage
## 2. create a secret which stores smb account name and password
```
kubectl create secret generic smbcreds --from-literal username=USERNAME --from-literal password="PASSWORD" --type="microsoft.com/smb"
```

## 2. create a pod with smb flexvolume mount on linux
#### Option#1 Ties a flexvolume volume explicitly to a pod
- download `nginx-flex-smb.yaml` file and modify `source` field
```
wget -O nginx-flex-smb.yaml https://raw.githubusercontent.com/odedia/kubernetes-volume-drivers/master/flexvolume/smb/nginx-flex-smb.yaml
vi nginx-flex-smb.yaml
```
 - create a pod with smb flexvolume driver mount
```
kubectl create -f nginx-flex-smb.yaml
```

#### Option#2 Create smb flexvolume PV & PVC and then create a pod based on PVC
 > Note: access modes of smb PV supports ReadWriteOnce(RWO), ReadOnlyMany(ROX) and ReadWriteMany(RWX)
 - download `pv-smb-flexvol.yaml` file, modify `source` field and create a smb flexvolume persistent volume(PV)
```
wget https://raw.githubusercontent.com/odedia/kubernetes-volume-drivers/master/flexvolume/smb/pv-smb-flexvol.yaml
kubectl create -f pv-smb-flexvol.yaml
```

 - create a smb flexvolume persistent volume claim(PVC)
```
 kubectl create -f https://raw.githubusercontent.com/odedia/kubernetes-volume-drivers/master/flexvolume/smb/pvc-smb-flexvol.yaml
```

 - check status of PV & PVC until its Status changed from `Pending` to `Bound`
 ```
 kubectl get pv
 kubectl get pvc
 ```
 
 - create a pod with smb flexvolume PVC
```
 kubectl create -f https://raw.githubusercontent.com/odedia/kubernetes-volume-drivers/master/flexvolume/smb/nginx-flex-smb-pvc.yaml
 ```

## 3. enter the pod container to do validation
 - watch the status of pod until its Status changed from `Pending` to `Running`
```
watch kubectl describe po nginx-flex-smb
```
 - enter the pod container
```
kubectl exec -it nginx-flex-smb -- bash
```

```
root@nginx-flex-smb:/# df -h
Filesystem                                 Size  Used Avail Use% Mounted on
overlay                                    291G  3.2G  288G   2% /
tmpfs                                      3.4G     0  3.4G   0% /dev
tmpfs                                      3.4G     0  3.4G   0% /sys/fs/cgroup
//xiazhang3.file.core.windows.net/k8stest   25G   64K   25G   1% /data
/dev/sda1                                  291G  3.2G  288G   2% /etc/hosts
shm                                         64M     0   64M   0% /dev/shm
tmpfs                                      3.4G   12K  3.4G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                      3.4G     0  3.4G   0% /sys/firmware
```
In the above example, there is a `/data` directory mounted as smb filesystem.

### smb flexvolume Parameters
Name | Meaning | Example | Mandatory 
--- | --- | --- | ---
source | smb server address | `//STORAGE-ACCOUNT.file.core.windows.net/SHARE-NAME` for auzre file format | Yes
mountoptions | mount options | `vers=3.0,dir_mode=0777,file_mode=0777` | No

#### Debugging skills
 - Check smb flexvolume installation result on the node:
```
sudo cat /var/log/smb-flexvol-installer.log
```
 - Get smb driver version:
```
kubectl get po -n kube-system | grep smb
kubectl describe po smb-flexvol-installer-xxxxx -n kube-system | grep smb-flexvolume
```
 - If there is pod mounting error like following:
```
MountVolume.SetUp failed for volume "test" : invalid character 'C' looking for beginning of value
```
Please checl the log at `/var/log/smb-driver.log` on the worker nodes.

### Links
[CIFS/SMB wiki](https://en.wikipedia.org/wiki/Server_Message_Block)

[Flexvolume doc](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md)

[Persistent Storage Using FlexVolume Plug-ins](https://docs.openshift.org/latest/install_config/persistent_storage/persistent_storage_flex_volume.html)
