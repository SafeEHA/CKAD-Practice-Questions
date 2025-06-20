# Core Concepts - Practice Questions

This section covers fundamental Kubernetes concepts including namespaces, pods, and basic resource management.

## Question 1 | Namespaces

**Solve this question on instance:** `ssh ckad5601`
 
The DevOps team would like to get the list of all Namespaces in the cluster. Get the list and save it to `/opt/course/1/namespaces` on ckad5601.

<details>
<summary>Answer</summary>

```bash
k get ns > /opt/course/1/namespaces
```

The content should then look like:
```bash
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

## Question 2 | Pods

**Solve this question on instance:** `ssh ckad5601`

Create a single Pod of image `httpd:2.4.41-alpine` in Namespace default. The Pod should be named `pod1` and the container should be named `pod1-container`.

Your manager would like to run a command manually on occasion to output the status of that exact Pod. Please write a command that does this into `/opt/course/2/pod1-status-command.sh` on ckad5601. The command should use kubectl.

<details>
<summary>Answer</summary>

First, generate the pod YAML:
```bash
k run pod1 --image=httpd:2.4.41-alpine --dry-run=client -oyaml > 2.yaml
```

Edit the YAML file to change the container name:
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

Create the pod:
```bash
k create -f 2.yaml
```

Verify the pod is running:
```bash
➜ k get pod
NAME   READY   STATUS    RESTARTS   AGE
pod1   1/1     Running   0          30s
```

Create the status command script:
```bash
vim /opt/course/2/pod1-status-command.sh
```

The content could be:
```bash
# /opt/course/2/pod1-status-command.sh
kubectl -n default describe pod pod1 | grep -i status:
```

Alternative solution using jsonpath:
```bash
# /opt/course/2/pod1-status-command.sh
kubectl -n default get pod pod1 -o jsonpath="{.status.phase}"
```

Test the command:
```bash
➜ sh /opt/course/2/pod1-status-command.sh
Running
```

</details>

## Question 7 | Pods, Namespaces

**Solve this question on instance:** `ssh ckad7326`

The board of Team Neptune decided to take over control of one e-commerce webserver from Team Saturn. The administrator who once setup this webserver is not part of the organisation any longer. All information you could get was that the e-commerce system is called my-happy-shop.

Search for the correct Pod in Namespace saturn and move it to Namespace neptune. It doesn't matter if you shut it down and spin it up again, it probably hasn't any customers anyways.

<details>
<summary>Answer</summary>

Let's see all those Pods:

```bash
➜ k -n saturn get pod
NAME                READY   STATUS    RESTARTS   AGE
webserver-sat-001   1/1     Running   0          111m
webserver-sat-002   1/1     Running   0          111m
webserver-sat-003   1/1     Running   0          111m
webserver-sat-004   1/1     Running   0          111m
webserver-sat-005   1/1     Running   0          111m
webserver-sat-006   1/1     Running   0          111m
```

The Pod names don't reveal any information. We assume the Pod we are searching has a label or annotation with the name my-happy-shop, so we search for it:

```bash
k -n saturn describe pod # describe all pods, then manually look for it

# or do some filtering like this
k -n saturn get pod -o yaml | grep my-happy-shop -A10
```

We see the webserver we're looking for is webserver-sat-003

```bash
k -n saturn get pod webserver-sat-003 -o yaml > 7_webserver-sat-003.yaml # export
vim 7_webserver-sat-003.yaml
```

Change the Namespace to neptune, also remove the status: section, the token volume, the token volumeMount and the nodeName, else the new Pod won't start. The final file could look as clean like this:

```yaml
# 7_webserver-sat-003.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    description: this is the server for the E-Commerce System my-happy-shop
  labels:
    id: webserver-sat-003
  name: webserver-sat-003
  namespace: neptune # new namespace here
spec:
  containers:
  - image: nginx:1.16.1-alpine
    imagePullPolicy: IfNotPresent
    name: webserver-sat
  restartPolicy: Always
```

Then we execute:

```bash
k -n neptune create -f 7_webserver-sat-003.yaml
```

```bash
➜ k -n neptune get pod | grep webserver
webserver-sat-003               1/1     Running            0          22s
```

It seems the server is running in Namespace neptune, so we can do:

```bash
k -n saturn delete pod webserver-sat-003 --force --grace-period=0
```

Let's confirm only one is running:

```bash
➜ k get pod -A | grep webserver-sat-003
neptune        webserver-sat-003         1/1     Running            0          6s
```

This should list only one pod called webserver-sat-003 in Namespace neptune, status running.

</details>

## Question 22 | Labels and Annotations

**Solve this question on instance:** `ssh ckad9043`

**Problem:** Team Sunny needs to identify some of their Pods in namespace `sun`. They ask you to add a new label `protected: true` to all Pods with an existing label `type: worker` or `type: runner`. Also add an annotation `protected: do not delete this pod` to all Pods having the new label `protected: true`.

<details>
<summary>Answer</summary>

1. **View current pods and their labels:**
```bash
k -n sun get pod --show-labels
```

This shows all pods with their current label assignments.

2. **Filter pods by specific labels (optional verification):**
```bash
k -n sun get pod -l type=runner    # only runner pods
k -n sun get pod -l type=worker    # only worker pods
```

3. **Add the new label to pods with type=runner:**
```bash
k -n sun label pod -l type=runner protected=true
```

4. **Add the new label to pods with type=worker:**
```bash
k -n sun label pod -l type=worker protected=true
```

**Alternative single command:**
```bash
k -n sun label pod -l "type in (worker,runner)" protected=true
```

5. **Verify the new labels were added:**
```bash
k -n sun get pod --show-labels
```

Should show `protected=true` on all pods that had `type=worker` or `type=runner`.

6. **Add annotations to all pods with the new protected label:**
```bash
k -n sun annotate pod -l protected=true protected="do not delete this pod"
```

7. **Optional: Verify annotations were added:**
```bash
k -n sun get pod -l protected=true -o yaml | grep -A 8 metadata:
```

---

</details>
