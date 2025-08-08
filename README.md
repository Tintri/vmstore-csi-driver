# Vmstore-csi-file-driver

Releases can be found here - https://github.com/Tintri/vmstore-csi-driver/releases

## Feature List
|Feature|Feature Status|CSI Driver Version|CSI Spec Version|Kubernetes Version|
|--- |--- |--- |--- |--- |
|Static Provisioning|GA|>= 1.0.0|>= 1.0.0|>=1.18|
|Dynamic Provisioning|GA|>= 1.0.0|>= 1.0.0|>=1.18|
|RW mode|GA|>= 1.0.0|>= 1.0.0|>=1.18|
|RO mode|GA|>= 1.0.0|>= 1.0.0|>=1.18|
|Expand volume|GA|>= 1.0.0|>= 1.1.0|>=1.18|
|StorageClass Secrets|GA|>= 1.0.0|>=1.0.0|>=1.18|
|Mount options|GA|>= 1.0.0|>= 1.0.0|>=1.18|
|Topology|GA|>= 2.0.0|>= 1.0.0|>=1.17|

## Access Modes support
|Access mode| Supported in version|
|--- |--- |
|ReadWriteOnce| >=1.0.0 |
|ReadOnlyMany| >=1.0.0 |
|ReadWriteOncePod| >=1.0.0 |

## Compatibility matrix
|CSI driver version| VMstore version|
|--- |--- |
|>=v1.0.1| >=5.6.0.1 |

## Requirements

- Kubernetes cluster must allow privileged pods, this flag must be set for the API server and the kubelet
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enable-privileged-pods)):
  ```
  --allow-privileged=true
  ```

## Prerequisites
Make sure that the top level folder where the CSI volumes will be created exists on VMStore and is shared over NFS.
For example `tintri/csi`.

## Installation
Clone or untar driver (depending on where you get the driver)
```bash
git clone -b <driver version> https://github.com/Tintri/vmstore-csi-driver.git
```
e.g:-
```bash
git clone -b 1.0.0 https://github.com/Tintri/vmstore-csi-driver.git
```

Edit `deploy/kubernetes/vmstore-csi-file-driver-config.yaml` file. Driver configuration example:
   ```yaml
   vmstore_map:
     vmstore1:
       mountPoint: /tintri-csi                           # mountpoint on the host where the tintri fs will be mounted
       dataIp: 172.30.200.37
       managementAddress: tafzq23.tintri.com
       vmstoreFS: /tintri/fs01
       zone: zone-1
       username: admin
       password: password

     vmstore2:
       mountPoint: /tintri-csi                           # mountpoint on the host where the tintri fs will be mounted
       dataIp: 10.136.40.71
       managementAddress: tafbm05.tintri.com
       vmstoreFS: /tintri/csi
       zone: zone-2
       username: admin
       password: password


   debug: true                                           # more logs

   ```

3. Create Kubernetes secret from the file:
   ```bash
   kubectl create secret generic vmstore-csi-file-driver-config --from-file=deploy/kubernetes/vmstore-csi-file-driver-config.yaml
   ```

4. Register driver to Kubernetes:
   ```bash
   kubectl apply -f deploy/kubernetes/vmstore-csi-file-driver.yaml
   ```

## Uninstall
Using the same files as for installation:

```bash
# delete driver
kubectl delete -f deploy/kubernetes/vmstore-csi-file-driver.yaml

# delete secret
kubectl delete secret vmstore-csi-file-driver-config
```

## Usage

### Dynamically provisioned volumes

For dynamic volume provisioning, the administrator needs to set up a _StorageClass_ in the PV yaml (examples/nginx-dynamic-volume.yaml for this example) pointing to the driver.
For dynamically provisioned volumes, Kubernetes generates volume name automatically (for example `pvc-vmstore-cfc67950-fe3c-11e8-a3ca-005056b857f8`).

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: vmstore-csi-file-driver-sc
provisioner: vmstore.csi.tintri.com
allowVolumeExpansion: true
```

#### Example

Run Nginx pod with dynamically provisioned volume:

```bash
kubectl apply -f examples/nginx-dynamic-volume.yaml

# to delete this pod:
kubectl delete -f examples/nginx-dynamic-volume.yaml
```

### Static (pre-provisioned) volumes

The driver can use already existing Vmstore filesystem,
in this case, _StorageClass_, _PersistentVolume_ and _PersistentVolumeClaim_ should be configured.

#### _StorageClass_ configuration

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: vmstore-csi-file-driver-sc-nginx-persistent
provisioner: vmstore.csi.tintri.com
allowVolumeExpansion: true
allowedTopologies:
- matchLabelExpressions:
  - key: topology.vmstore.csi.tintri.com/zone
    values:
    - zone-1
mountOptions:                         # list of options for `mount -o ...` command
#  - noatime                          #
```

#### _PersistentVolume_ configuration

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: vmstore-csi-file-driver-pv-nginx-persistent
  labels:
    name: vmstore-csi-file-driver-pv-nginx-persistent
spec:
  storageClassName: vmstore-csi-file-driver-sc-nginx-persistent
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  csi:
    driver: vmstore.csi.tintri.com
    volumeHandle: vmstore1/172.30.230.128:/tintri/csi/vmstorefs-mnt/persistent/volumefile-persistent.img
```

#### _PersistentVolumeClaim_ (pointing to created _PersistentVolume_)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vmstore-csi-file-driver-pvc-nginx-persistent
spec:
  storageClassName: vmstore-csi-file-driver-sc-nginx-persistent
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      # to create 1-1 relationship for pod - persistent volume use unique labels
      name: vmstore-csi-file-driver-pv-nginx-persistent
```

### Topology configuration
In order to configure CSI driver with kubernetes topology, use the `zone` parameter in driver config or storageClass parameters. Example config file with zones:
  ```bash
  vmstore_map:
    vmstore1:
      mountPoint: /tintri-csi                            # mountpoint on the host where the tintri fs will be mounted
      dataIp: 172.30.200.37
      managementAddress: tafzq23.tintri.com
      vmstoreFS: /tintri/fs01
      zone: zone-1
      username: admin
      password: password

    vmstore2:
      mountPoint: /tintri-csi                            # mountpoint on the host where the tintri fs will be mounted
      dataIp: 10.136.40.71
      managementAddress: tafbm05.tintri.com
      vmstoreFS: /tintri/csi
      zone: zone-2
      username: admin
      password: password
  ```

This will assign volumes to be created on Vmstore that correspond with the zones requested by allowedTopologies values.


### Defaults/Configuration/Parameter options

| Default  | Config               |  Parameter    |  Desc                             |
|----------|----------------------|---------------|-----------------------------------|
|   -      | dataIp               |   dataIp      | Data address used to mount Vmstore filesystem |
|	-	   | mountPoint			  |		 -  	  |	Mount point inside of the driver container where Vmstore filesystem will be mounted|
|   -      | username             |      -        | Vmstore API username |
|   -      | password             |      -        | Vmstore API password |
|   -      | zone                 |      -        | Zone to match topology.vmstore.csi.tintri.com/zone            |
|   -      | vmstoreFS            |   vmstoreFS   | Path to filesystem on Vmstore |
|   -      | managementAddress    |      -        | FQDN for Vmstore management operations |
|   -      | defaultMountOptions  |   mountOptions| Mount options |
|   0777   | mountPointPermissions|   mountPointPermissions | File permissions to be set on mount point |

#### Example

Run nginx server using PersistentVolume.

**Note:** Pre-configured volume file should exist on the Vmstore:
`/tintri/csi/vmstorefs-mnt/persistent/volumefile-persistent.img`.

```bash
kubectl apply -f nginx-persistent-volume.yaml

# to delete this pod and volume:
kubectl delete -f nginx-persistent-volume.yaml
```

## Snapshots
To use CSI snapshots, the snapshot CRDs along with the csi-snapshotter must be installed.
```bash
kubectl apply -f deploy/kubernetes/snapshots/
```

After that the snapshot class for VMStore CSI must be created

```bash
kubectl apply -f examples/snapshot-class.yaml
```

Now a snapshot can be created, for example
```bash
kubectl apply -f examples/snapshot-from-dynamic.yaml
```

We can create a volume using the snapshot
```bash
kubectl apply -f examples/nginx-from-snapshot.yaml
```

## Troubleshooting
### Driver logs
To collect all driver related logs, you can use the `kubectl logs` command.
All in on command:
```bash
mkdir vmstore-csi-logs
for name in $(kubectl get pod -owide | grep vmstore | awk '{print $1}'); do kubectl logs $name --all-containers > vmstore-csi-logs/$name; done
```

To get logs from all containers of a single pod 
```bash
kubectl logs <pod_name> --all-containers
```

Logs from a single container of a pod
```bash
kubectl logs <pod_name> -c driver
```

#### Driver secret
```bash
kubectl get secret vmstore-csi-file-driver-config -o json | jq '.data | map_values(@base64d)' > vmstore-csi-logs/vmstore-csi-file-driver-config
```

#### PV/PVC/Pod data
```bash
kubectl get pvc > vmstore-csi-logs/pvcs
kubectl get pv > vmstore-csi-logs/pvs
kubectl get pod > vmstore-csi-logs/pods
```

#### Extended info about a PV/PVC/Pod:
```bash
kubectl describe pvc <pvc_name> > vmstore-csi-logs/<pvc_name>
kubectl describe pv <pv_name> > vmstore-csi-logs/<pv_name>
kubectl describe pod <pod_name> > vmstore-csi-logs/<pod_name>
```
