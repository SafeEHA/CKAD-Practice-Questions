# CKAD Test Questions

## Question 1 | Namespaces

**Solve this question on instance:** `ssh ckad5601`

The DevOps team would like to get the list of all Namespaces in the cluster. Get the list and save it to `/opt/course/1/namespaces` on ckad5601.

<details>
<summary>Click to reveal answer</summary>

```bash
k get ns > /opt/course/1/namespaces
```

The content should then look like:
```
# /opt/course/1/namespaces
NAME              STATUS   AGE
default           Active   136m
earth             Active   105m
jupiter           Active   105m
kube-node-lease   Active   136m
kube-public       Active   136m
kube-system       Active   136m
mars              Active   105m
shell-intern      Active   105m
```

</details>

---

## Question 2 | Pods

**Solve this question on instance:** `ssh ckad5601`

Create a single Pod of image `httpd:2.4.41-alpine` in Namespace default. The Pod should be named `pod1` and the container should be named `pod1-container`.

Your manager would like to run a command manually on occasion to output the status of that exact Pod. Please write a command that does this into `/opt/course/2/pod1-status-command.sh` on ckad5601. The command should use kubectl.

<details>
<summary>Click to reveal answer</summary>

```bash
k run # help

k run pod1 --image=httpd:2.4.41-alpine --dry-run=client -oyaml > 2.yaml

vim 2.yaml
```

Change the container name in `2.yaml` to `pod1-container`:

```yaml
# 2.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod1
  name: pod1
spec:
  containers:
  - image: httpd:2.4.41-alpine
    name: pod1-container # change
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Then run:

```bash
➜ k create -f 2.yaml
pod/pod1 created

➜ k get pod
NAME   READY   STATUS              RESTARTS   AGE
pod1   0/1     ContainerCreating   0          6s

➜ k get pod
NAME   READY   STATUS    RESTARTS   AGE
pod1   1/1     Running   0          30s
```

Next create the requested command:

```bash
vim /opt/course/2/pod1-status-command.sh
```

The content of the command file could look like:

```bash
# /opt/course/2/pod1-status-command.sh
kubectl -n default describe pod pod1 | grep -i status:
```

Another solution would be using jsonpath:

```bash
# /opt/course/2/pod1-status-command.sh
kubectl -n default get pod pod1 -o jsonpath="{.status.phase}"
```

To test the command:

```bash
➜ sh /opt/course/2/pod1-status-command.sh
Running
```

</details>

---

## Question 3 | Job

**Solve this question on instance:** `ssh ckad7326`

Team Neptune needs a Job template located at `/opt/course/3/job.yaml`. This Job should run image `busybox:1.31.0` and execute `sleep 2 && echo done`. It should be in namespace `neptune`, run a total of 3 times and should execute 2 runs in parallel.

Start the Job and check its history. Each pod created by the Job should have the label `id: awesome-job`. The job should be named `neb-new-job` and the container `neb-new-job-container`.

<details>
<summary>Click to reveal answer</summary>

```bash
k -n neptun create job -h

k -n neptune create job neb-new-job --image=busybox:1.31.0 --dry-run=client -oyaml > /opt/course/3/job.yaml -- sh -c "sleep 2 && echo done"

vim /opt/course/3/job.yaml
```

Make the required changes in the yaml:

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

Then to create it:

```bash
k -f /opt/course/3/job.yaml create # namespace already set in yaml
```

Check Job and Pods, you should see two running parallel at most but three in total:

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
pod/neb-new-job-gm8sz              0/1     ContainerCreating   0          5s
pod/neb-new-job-jhq2g              0/1     Completed           0          15s
pod/neb-new-job-vf6ts              0/1     Completed           0          15s
job.batch/neb-new-job   2/3           15s        16s

➜ k -n neptune get pod,job | grep neb-new-job
pod/neb-new-job-gm8sz              0/1     Completed          0          12s
pod/neb-new-job-jhq2g              0/1     Completed          0          22s
pod/neb-new-job-vf6ts              0/1     Completed          0          22s

job.batch/neb-new-job   3/3           21s        23s
```

Check history:

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

At the age column we can see that two pods run parallel and the third one after that. Just as it was required in the task.

</details>

---

## Question 4 | Helm Management

**Solve this question on instance:** `ssh ckad7326`

Team Mercury asked you to perform some operations using Helm, all in Namespace mercury:

1. Delete release `internal-issue-report-apiv1`
2. Upgrade release `internal-issue-report-apiv2` to any newer version of chart `bitnami/nginx` available
3. Install a new release `internal-issue-report-apache` of chart `bitnami/apache`. The Deployment should have two replicas, set these via Helm-values during install
4. There seems to be a broken release, stuck in pending-install state. Find it and delete it

<details>
<summary>Click to reveal answer</summary>

**Helm Concepts:**
- **Helm Chart:** Kubernetes YAML template-files combined into a single package, Values allow customisation
- **Helm Release:** Installed instance of a Chart
- **Helm Values:** Allow to customise the YAML template-files in a Chart when creating a Release

### 1. Delete release

First we should delete the required release:

```bash
➜ helm -n mercury ls
NAME                            NAMESPACE     STATUS          CHART           APP VERSION
internal-issue-report-apiv1     mercury       deployed        nginx-9.5.0     1.21.1     
internal-issue-report-apiv2     mercury       deployed        nginx-9.5.0     1.21.1     
internal-issue-report-app       mercury       deployed        nginx-9.5.0     1.21.1  

➜ helm -n mercury uninstall internal-issue-report-apiv1
release "internal-issue-report-apiv1" uninstalled

➜ helm -n mercury ls
NAME                            NAMESPACE     STATUS          CHART           APP VERSION
internal-issue-report-apiv2     mercury       deployed        nginx-9.5.0     1.21.1     
internal-issue-report-app       mercury       deployed        nginx-9.5.0     1.21.1  
```

### 2. Upgrade release

Next we need to upgrade a release, for this we could first list the charts of the repo:

```bash
➜ helm repo list
NAME    URL                               
bitnami https://charts.bitnami.com/bitnami

➜ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈

➜ helm search repo nginx
NAME                  CHART VERSION   APP VERSION     DESCRIPTION          
bitnami/nginx         9.5.2           1.21.1          Chart for the nginx server             ...
```

Here we see that a newer chart version 9.5.2 is available. But the task only requires us to upgrade to any newer chart version available, so we can simply run:

```bash
➜ helm -n mercury upgrade internal-issue-report-apiv2 bitnami/nginx
Release "internal-issue-report-apiv2" has been upgraded. Happy Helming!
NAME: internal-issue-report-apiv2
LAST DEPLOYED: Tue Aug 31 17:40:42 2021
NAMESPACE: mercury
STATUS: deployed
REVISION: 2
TEST SUITE: None
...

➜ helm -n mercury ls
NAME                            NAMESPACE     STATUS          CHART           APP VERSION
internal-issue-report-apiv2     mercury       deployed        nginx-9.5.2     1.21.1     
internal-issue-report-app       mercury       deployed        nginx-9.5.0     1.21.1 
```

Looking good!

**INFO:** Also check out `helm rollback` for undoing a helm rollout/upgrade

### 3. Install new release

Now we're asked to install a new release, with a customised values setting. For this we first list all possible value settings for the chart, we can do this via:

```bash
helm show values bitnami/apache # will show a long list of all possible value-settings

helm show values bitnami/apache | yq e # parse yaml and show with colors
```

Huge list, if we search in it we should find the setting `replicaCount: 1` on top level. This means we can run:

```bash
➜ helm -n mercury install internal-issue-report-apache bitnami/apache --set replicaCount=2
NAME: internal-issue-report-apache
LAST DEPLOYED: Tue Aug 31 17:57:23 2021
NAMESPACE: mercury
STATUS: deployed
REVISION: 1
TEST SUITE: None
...
```

If we would also need to set a value on a deeper level, for example `image.debug`, we could run:

```bash
helm -n mercury install internal-issue-report-apache bitnami/apache \
  --set replicaCount=2 \
  --set image.debug=true
```

Install done, let's verify what we did:

```bash
➜ helm -n mercury ls
NAME                            NAMESPACE     STATUS          CHART           APP VERSION
internal-issue-report-apache    mercury       deployed        apache-8.6.3    2.4.48
...

➜ k -n mercury get deploy internal-issue-report-apache
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
internal-issue-report-apache   2/2     2            2           96s
```

We see a healthy deployment with two replicas!

### 4. Find and delete broken release

By default releases in pending-upgrade state aren't listed, but we can show all to find and delete the broken release:

```bash
➜ helm -n mercury ls -a
NAME                            NAMESPACE     STATUS          CHART           APP VERSION
internal-issue-report-apache    mercury       deployed        apache-8.6.3    2.4.48     
internal-issue-report-apiv2     mercury       deployed        nginx-9.5.2     1.21.1     
internal-issue-report-app       mercury       deployed        nginx-9.5.0     1.21.1     
internal-issue-report-daniel    mercury       pending-install nginx-9.5.0     1.21.1 

➜ helm -n mercury uninstall internal-issue-report-daniel
release "internal-issue-report-daniel" uninstalled
```

Thank you Helm for making our lifes easier! (Till something breaks)

</details>

---

## Question 5 | ServiceAccount, Secret

**Solve this question on instance:** `ssh ckad7326`

Team Neptune has its own ServiceAccount named `neptune-sa-v2` in Namespace `neptune`. A coworker needs the token from the Secret that belongs to that ServiceAccount. Write the base64 decoded token to file `/opt/course/5/token` on ckad7326.

<details>
<summary>Click to reveal answer</summary>

Since K8s 1.24, Secrets won't be created automatically for ServiceAccounts any longer. But it's still possible to create a Secret manually and attach it to a ServiceAccount by setting the correct annotation on the Secret. This was done for this task.

```bash
k -n neptune get sa # get overview
k -n neptune get secrets # shows all secrets of namespace
k -n neptune get secrets -oyaml | grep annotations -A 1 # shows secrets with first annotation
```

If a Secret belongs to a ServiceAccount, it'll have the annotation `kubernetes.io/service-account.name`. Here the Secret we're looking for is `neptune-secret-1`.

```bash
➜ k -n neptune get secret neptune-secret-1 -o yaml
apiVersion: v1
data:
...
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkltNWFaRmRxWkRKMmFHTnZRM0JxV0haT1IxZzFiM3BJY201SlowaEhOV3hUWmt3elFuRmFhVEZhZDJNaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUp1WlhCMGRXNWxJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbTVsY0hSMWJtVXRjMkV0ZGpJdGRHOXJaVzR0Wm5FNU1tb2lMQ0pyZFdKbGNtNWxkR1Z6TG1sdkwzTmxjblpwWTJWaFkyTnZkVzUwTDNObGNuWnBZMlV0WVdOamIzVnVkQzV1WVcxbElqb2libVZ3ZEhWdVpTMXpZUzEyTWlJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZ5ZG1salpTMWhZMk52ZFc1MExuVnBaQ0k2SWpZMlltUmpOak0yTFRKbFl6TXROREpoWkMwNE9HRTFMV0ZoWXpGbFpqWmxPVFpsTlNJc0luTjFZaUk2SW5ONWMzUmxiVHB6WlhKMmFXTmxZV05qYjNWdWREcHVaWEIwZFc1bE9tNWxjSFIxYm1VdGMyRXRkaklpZlEuVllnYm9NNENUZDBwZENKNzh3alV3bXRhbGgtMnZzS2pBTnlQc2gtNmd1RXdPdFdFcTVGYnc1WkhQdHZBZHJMbFB6cE9IRWJBZTRlVU05NUJSR1diWUlkd2p1Tjk1SjBENFJORmtWVXQ0OHR3b2FrUlY3aC1hUHV3c1FYSGhaWnp5NHlpbUZIRzlVZm1zazVZcjRSVmNHNm4xMzd5LUZIMDhLOHpaaklQQXNLRHFOQlF0eGctbFp2d1ZNaTZ2aUlocnJ6QVFzME1CT1Y4Mk9KWUd5Mm8tV1FWYzBVVWFuQ2Y5NFkzZ1QwWVRpcVF2Y3pZTXM2bno5dXQtWGd3aXRyQlk2VGo5QmdQcHJBOWtfajVxRXhfTFVVWlVwUEFpRU43T3pka0pzSThjdHRoMTBseXBJMUFlRnI0M3Q2QUx5clFvQk0zOWFiRGZxM0Zrc1Itb2NfV013
kind: Secret
...
```

This shows the base64 encoded token. To get the decoded one we could pipe it manually through `base64 -d` or we simply do:

```bash
➜ k -n neptune describe secret neptune-secret-1
...
Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Im5aZFdqZDJ2aGNvQ3BqWHZOR1g1b3pIcm5JZ0hHNWxTZkwzQnFaaTFad2MifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJuZXB0dW5lIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6Im5lcHR1bmUtc2EtdjItdG9rZW4tZnE5MmoiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoibmVwdHVuZS1zYS12MiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQv