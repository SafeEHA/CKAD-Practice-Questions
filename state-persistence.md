# State Persistence - Practice Questions

This section covers persistent volumes, persistent volume claims, storage classes, and data persistence in Kubernetes.

## Question 12 | Storage, PV, PVC, Pod Volume

**Solve this question on instance:** `ssh ckad5601`

Create a new PersistentVolume named `earth-project-earthflower-pv`. It should have a capacity of 2Gi, accessMode ReadWriteOnce, hostPath /Volumes/Data and no storageClassName defined.

Next create a new PersistentVolumeClaim in Namespace `earth` named `earth-project-earthflower-pvc`. It should request 2Gi storage, accessMode ReadWriteOnce and should not define a storageClassName. The PVC should bound to the PV correctly.

Finally create a new Deployment `project-earthflower` in Namespace `earth` which mounts that volume at `/tmp/project-data`. The Pods of that Deployment should be of image `httpd:2.4.41-alpine`.

<details>
<summary>Answer</summary>

**Step 1: Create the PersistentVolume**

```bash
vim 12_pv.yaml
```

Create the PV yaml file (find examples at https://kubernetes.io/docs):

```yaml
# 12_pv.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
 name: earth-project-earthflower-pv
spec:
 capacity:
  storage: 2Gi
 accessModes:
  - ReadWriteOnce
 hostPath:
  path: "/Volumes/Data"
```

Apply the PV:

```bash
k -f 12_pv.yaml create
```

**Step 2: Create the PersistentVolumeClaim**

```bash
vim 12_pvc.yaml
```

Create the PVC yaml file:

```yaml
# 12_pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: earth-project-earthflower-pvc
  namespace: earth
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
     storage: 2Gi
```

Apply the PVC:

```bash
k -f 12_pvc.yaml create
```

**Step 3: Verify PV and PVC are bound**

```bash
k -n earth get pv,pvc
```

Expected output:
```
NAME                                 CAPACITY   ACCESS MODES   ...  STATUS   CLAIM 
persistentvolume/...earthflower-pv   2Gi        RWO            ...  Bound    ...er-pvc

NAME                                       STATUS   VOLUME                         CAPACITY
persistentvolumeclaim/...earthflower-pvc   Bound    earth-project-earthflower-pv   2Gi
```

**Step 4: Create Deployment with mounted volume**

Generate base deployment yaml:

```bash
k -n earth create deploy project-earthflower --image=httpd:2.4.41-alpine --dry-run=client -oyaml > 12_dep.yaml
```

Edit the deployment to mount the volume:

```bash
vim 12_dep.yaml
```

```yaml
# 12_dep.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: project-earthflower
  name: project-earthflower
  namespace: earth
spec:
  replicas: 1
  selector:
    matchLabels:
      app: project-earthflower
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: project-earthflower
    spec:
      volumes:                                      # add
      - name: data                                  # add
        persistentVolumeClaim:                      # add
          claimName: earth-project-earthflower-pvc  # add
      containers:
      - image: httpd:2.4.41-alpine
        name: container
        volumeMounts:                               # add
        - name: data                                # add
          mountPath: /tmp/project-data              # add
```

Apply the deployment:

```bash
k -f 12_dep.yaml create
```

**Step 5: Verify volume mounting**

```bash
k -n earth describe pod project-earthflower-d6887f7c5-pn5wv | grep -A2 Mounts:
```

Expected output:
```
    Mounts:
      /tmp/project-data from data (rw) # there it is
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-n2sjj (ro)
```


---

</details>

## Question 13 | Storage, StorageClass, PVC

**Solve this question on instance:** `ssh ckad9043`

Team Moonpie, which has the Namespace `moon`, needs more storage. Create a new PersistentVolumeClaim named `moon-pvc-126` in that namespace. This claim should use a new StorageClass `moon-retain` with the provisioner set to `moon-retainer` and the reclaimPolicy set to Retain. The claim should request storage of 3Gi, an accessMode of ReadWriteOnce and should use the new StorageClass.

The provisioner `moon-retainer` will be created by another team, so it's expected that the PVC will not boot yet. Confirm this by writing the event message from the PVC into file `/opt/course/13/pvc-126-reason` on ckad9043.

<details>
<summary>Answer</summary>

**Step 1: Create the StorageClass**

```bash
vim 13_sc.yaml
```

Find StorageClass examples at https://kubernetes.io/docs:

```yaml
# 13_sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: moon-retain
provisioner: moon-retainer
reclaimPolicy: Retain
```

Apply the StorageClass:

```bash
k create -f 13_sc.yaml
```

**Step 2: Create the PersistentVolumeClaim**

```bash
vim 13_pvc.yaml
```

```yaml
# 13_pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: moon-pvc-126            # name as requested
  namespace: moon               # important
spec:
  accessModes:
    - ReadWriteOnce             # RWO
  resources:
    requests:
      storage: 3Gi              # size
  storageClassName: moon-retain # uses our new storage class
```

Apply the PVC:

```bash
k -f 13_pvc.yaml create
```

**Step 3: Check PVC Status**

```bash
k -n moon get pvc
```

Expected output (PVC will be in Pending state):
```
NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
moon-pvc-126   Pending                                      moon-retain    2m57s
```

**Step 4: Get detailed event information**

```bash
k -n moon describe pvc moon-pvc-126
```

Expected output:
```
Name:          moon-pvc-126
...
Status:        Pending
...
Events:
  Type    Reason                Age                  From                         Message
  ----    ------                ----                 ----                         -------
Normal  ExternalProvisioning  4s (x19 over 4m28s)  persistentvolume-controller    Waiting for a volume to be created either by the external provisioner 'moon-retainer' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.
```

**Step 5: Write event message to file**

Create the file with the event message:

```bash
# /opt/course/13/pvc-126-reason
Waiting for a volume to be created either by the external provisioner 'moon-retainer' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.
```

</details>