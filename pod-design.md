# Pod Design - Practice Questions

This section covers workload management including Jobs, CronJobs, and pod scheduling patterns.

## Question 3 | Job

**Solve this question on instance:** `ssh ckad7326`

Team Neptune needs a Job template located at `/opt/course/3/job.yaml`. This Job should run image `busybox:1.31.0` and execute `sleep 2 && echo done`. It should be in namespace `neptune`, run a total of 3 times and should execute 2 runs in parallel.

Start the Job and check its history. Each pod created by the Job should have the label `id: awesome-job`. The job should be named `neb-new-job` and the container `neb-new-job-container`.

<details>
<summary>Answer</summary>

### Generate the initial Job YAML:
```bash
k -n neptune create job neb-new-job --image=busybox:1.31.0 --dry-run=client -oyaml > /opt/course/3/job.yaml -- sh -c "sleep 2 && echo done"
```

Edit the YAML file to add required specifications:
```yaml
# /opt/course/3/job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: neb-new-job
  namespace: neptune      # add
spec:
  completions: 3          # add
  parallelism: 2          # add
  template:
    metadata:
      creationTimestamp: null
      labels:             # add
        id: awesome-job   # add
    spec:
      containers:
      - command:
        - sh
        - -c
        - sleep 2 && echo done
        image: busybox:1.31.0
        name: neb-new-job-container # update
        resources: {}
      restartPolicy: Never
status: {}
```

Create the Job:
```bash
k -f /opt/course/3/job.yaml create
```

Monitor the Job execution - you should see two pods running in parallel at most, but three total:

```bash
➜ k -n neptune get pod,job | grep neb-new-job
pod/neb-new-job-jhq2g              0/1     ContainerCreating   0          4s
pod/neb-new-job-vf6ts              0/1     ContainerCreating   0          4s
job.batch/neb-new-job   0/3           4s         5s

➜ k -n neptune get pod,job | grep neb-new-job
pod/neb-new-job-gm8sz              0/1     ContainerCreating   0          0s
pod/neb-new-job-jhq2g              0/1     Completed           0          10s
pod/neb-new-job-vf6ts              1/1     Running             0          10s
job.batch/neb-new-job   1/3           10s        11s

➜ k -n neptune get pod,job | grep neb-new-job
pod/neb-new-job-gm8sz              0/1     Completed          0          12s
pod/neb-new-job-jhq2g              0/1     Completed          0          22s
pod/neb-new-job-vf6ts              0/1     Completed          0          22s
job.batch/neb-new-job   3/3           21s        23s
```

Check the Job history:
```bash
➜ k -n neptune describe job neb-new-job
...
Events:
  Type    Reason            Age    From            Message
  ----    ------            ----   ----            -------
  Normal  SuccessfulCreate  2m52s  job-controller  Created pod: neb-new-job-jhq2g
  Normal  SuccessfulCreate  2m52s  job-controller  Created pod: neb-new-job-vf6ts
  Normal  SuccessfulCreate  2m42s  job-controller  Created pod: neb-new-job-gm8sz
```

The events show that two pods ran in parallel initially, and the third pod was created after one of the first two completed.

</details>

## Question 8 | Deployment, Rollouts

**Solve this question on instance:** `ssh ckad7326`

There is an existing Deployment named api-new-c32 in Namespace neptune. A developer did make an update to the Deployment but the updated version never came online. Check the Deployment history and find a revision that works, then rollback to it. Could you tell Team Neptune what the error was so it doesn't happen again?

<details>
<summary>Answer</summary>

```bash
k -n neptune get deploy # overview
k -n neptune rollout -h
k -n neptune rollout history -h
```

```bash
➜ k -n neptune rollout history deploy api-new-c32
deployment.extensions/api-new-c32 
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl edit deployment api-new-c32 --namespace=neptune
3         kubectl edit deployment api-new-c32 --namespace=neptune
4         kubectl edit deployment api-new-c32 --namespace=neptune
5         kubectl edit deployment api-new-c32 --namespace=neptune
```

We see 5 revisions, let's check Pod and Deployment status:

```bash
➜ k -n neptune get deploy,pod | grep api-new-c32
deployment.extensions/api-new-c32    3/3     1            3           141m

pod/api-new-c32-65d998785d-jtmqq    1/1     Running            0          141m
pod/api-new-c32-686d6f6b65-mj2fp    1/1     Running            0          141m
pod/api-new-c32-6dd45bdb68-2p462    1/1     Running            0          141m
pod/api-new-c32-7d64747c87-zh648    0/1     ImagePullBackOff   0          141m
```

Let's check the pod for errors:

```bash
➜ k -n neptune describe pod api-new-c32-7d64747c87-zh648 | grep -i error
  ...  Error: ImagePullBackOff
```

```bash
➜ k -n neptune describe pod api-new-c32-7d64747c87-zh648 | grep -i image
    Image:          ngnix:1.16.3
    Image ID:
      Reason:       ImagePullBackOff
  Warning  Failed  4m28s (x616 over 144m)  kubelet, gke-s3ef67020-28c5-45f7--default-pool-248abd4f-s010  Error: ImagePullBackOff
```

Someone seems to have added a new image with a spelling mistake in the name ngnix:1.16.3, that's the reason we can tell Team Neptune!

Now let's revert to the previous version:

```bash
k -n neptune rollout undo deploy api-new-c32
```

Does this one work?

```bash
➜ k -n neptune get deploy api-new-c32
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
api-new-c32   3/3     3            3           146m
```

Yes! All up-to-date and available.

Also a fast way to get an overview of the ReplicaSets of a Deployment and their images could be done with:

```bash
k -n neptune get rs -o wide | grep api-new-c32
```

</details>

## Question 9 | Pod -> Deployment

**Solve this question on instance:** `ssh ckad9043`

In Namespace pluto there is single Pod named holy-api. It has been working okay for a while now but Team Pluto needs it to be more reliable.

Convert the Pod into a Deployment named holy-api with 3 replicas and delete the single Pod once done. The raw Pod template file is available at /opt/course/9/holy-api-pod.yaml.

In addition, the new Deployment should set allowPrivilegeEscalation: false and privileged: false for the security context on container level.

Please create the Deployment and save its yaml under /opt/course/9/holy-api-deployment.yaml on ckad9043.

<details>
<summary>Answer</summary>

There are multiple ways to do this, one is to copy an Deployment example from https://kubernetes.io/docs and then merge it with the existing Pod yaml. That's what we will do now:

```bash
cp /opt/course/9/holy-api-pod.yaml /opt/course/9/holy-api-deployment.yaml # make a copy!

vim /opt/course/9/holy-api-deployment.yaml
```

Now copy/use a Deployment example yaml and put the Pod's metadata: and spec: into the Deployment's template: section:

```yaml
# /opt/course/9/holy-api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: holy-api        # name stays the same
  namespace: pluto      # important
spec:
  replicas: 3           # 3 replicas
  selector:
    matchLabels:
      id: holy-api      # set the correct selector
  template:
    # => from here down its the same as the pods metadata: and spec: sections
    metadata:
      labels:
        id: holy-api
      name: holy-api
    spec:
      containers:
      - env:
        - name: CACHE_KEY_1
          value: b&MTCi0=[T66RXm!jO@
        - name: CACHE_KEY_2
          value: PCAILGej5Ld@Q%{Q1=#
        - name: CACHE_KEY_3
          value: 2qz-]2OJlWDSTn_;RFQ
        image: nginx:1.17.3-alpine
        name: holy-api-container
        securityContext:                   # add
          allowPrivilegeEscalation: false  # add
          privileged: false                # add
        volumeMounts:
        - mountPath: /cache1
          name: cache-volume1
        - mountPath: /cache2
          name: cache-volume2
        - mountPath: /cache3
          name: cache-volume3
      volumes:
      - emptyDir: {}
        name: cache-volume1
      - emptyDir: {}
        name: cache-volume2
      - emptyDir: {}
        name: cache-volume3
```

To indent multiple lines using vim you should set the shiftwidth using :set shiftwidth=2. Then mark multiple lines using Shift v and the up/down keys.
To then indent the marked lines press > or < and to repeat the action press .

Next create the new Deployment:

```bash
k -f /opt/course/9/holy-api-deployment.yaml create
```

and confirm it's running:

```bash
➜ k -n pluto get pod | grep holy
NAME                        READY   STATUS    RESTARTS   AGE
holy-api                    1/1     Running   0          19m
holy-api-5dbfdb4569-8qr5x   1/1     Running   0          30s
holy-api-5dbfdb4569-b5clh   1/1     Running   0          30s
holy-api-5dbfdb4569-rj2gz   1/1     Running   0          30s
```

Finally delete the single Pod:

```bash
k -n pluto delete pod holy-api --force --grace-period=0
```

```bash
➜ k -n pluto get pod,deployment | grep holy
pod/holy-api-5dbfdb4569-8qr5x   1/1     Running   0          2m4s
pod/holy-api-5dbfdb4569-b5clh   1/1     Running   0          2m4s
pod/holy-api-5dbfdb4569-rj2gz   1/1     Running   0          2m4s

deployment.extensions/holy-api   3/3     3            3           2m4s
```

</details>