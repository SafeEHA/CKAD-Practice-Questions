## Question 16 | Logging Sidecar

**Solve this question on instance:** `ssh ckad7326`

The Tech Lead of Mercury2D decided it's time for more logging, to finally fight all these missing data incidents. There is an existing container named `cleaner-con` in Deployment `cleaner` in Namespace `mercury`. This container mounts a volume and writes logs into a file called `cleaner.log`.

The yaml for the existing Deployment is available at `/opt/course/16/cleaner.yaml`. Persist your changes at `/opt/course/16/cleaner-new.yaml` on ckad7326 but also make sure the Deployment is running.

Create a sidecar container named `logger-con`, image `busybox:1.31.0`, which mounts the same volume and writes the content of `cleaner.log` to stdout, you can use the `tail -f` command for this. This way it can be picked up by `kubectl logs`.

Check if the logs of the new container reveal something about the missing data incidents.

<details>
<summary>Answer</summary>

**Step 1: Copy and modify the existing Deployment**

```bash
cp /opt/course/16/cleaner.yaml /opt/course/16/cleaner-new.yaml
vim /opt/course/16/cleaner-new.yaml
```

**Step 2: Add sidecar container as initContainer**

Modern Kubernetes supports sidecar containers as initContainers with `restartPolicy: Always`:

```yaml
# /opt/course/16/cleaner-new.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  name: cleaner
  namespace: mercury
spec:
  replicas: 2
  selector:
    matchLabels:
      id: cleaner
  template:
    metadata:
      labels:
        id: cleaner
    spec:
      volumes:
      - name: logs
        emptyDir: {}
      initContainers:
      - name: init
        image: bash:5.0.11
        command: ['bash', '-c', 'echo init > /var/log/cleaner/cleaner.log']
        volumeMounts:
        - name: logs
          mountPath: /var/log/cleaner
      - name: logger-con                                                # add
        image: busybox:1.31.0                                           # add
        restartPolicy: Always                                           # add
        command: ["sh", "-c", "tail -f /var/log/cleaner/cleaner.log"]   # add
        volumeMounts:                                                   # add
        - name: logs                                                    # add
          mountPath: /var/log/cleaner                                   # add
      containers:
      - name: cleaner-con
        image: bash:5.0.11
        args: ['bash', '-c', 'while true; do echo `date`: "remove random file" >> /var/log/cleaner/cleaner.log; sleep 1; done']
        volumeMounts:
        - name: logs
          mountPath: /var/log/cleaner
```

**Step 3: Legacy approach (for older Kubernetes versions)**

In earlier K8s versions, sidecar containers were defined as additional application containers:

```yaml
# LEGACY example of defining sidecar containers in earlier K8s versions
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  name: cleaner
  namespace: mercury
spec:
# ...
  template:
# ...
    spec:
# ...
      initContainers:
      - name: init
        image: bash:5.0.11
# ...
      containers:
      - name: cleaner-con
        image: bash:5.0.11
# ...
      - name: logger-con                                                # LEGACY example
        image: busybox:1.31.0                                           # LEGACY example
        command: ["sh", "-c", "tail -f /var/log/cleaner/cleaner.log"]   # LEGACY example
        volumeMounts:                                                   # LEGACY example
        - name: logs                                                    # LEGACY example
          mountPath: /var/log/cleaner                                   # LEGACY example
```

**Step 4: Apply the changes**

```bash
k -f /opt/course/16/cleaner-new.yaml apply
```

**Step 5: Monitor the deployment rollout**

```bash
k -n mercury rollout history deploy cleaner
k -n mercury rollout history deploy cleaner --revision 1
k -n mercury rollout history deploy cleaner --revision 2
```

**Step 6: Check Pod statuses**

```bash
k -n mercury get pod
```

Initial output (during rollout):
```
NAME                       READY   STATUS        RESTARTS   AGE
cleaner-86b7758668-9pw6t   2/2     Running       0          6s
cleaner-86b7758668-qgh4v   0/2     Init:0/1      0          1s
```

Final output (after rollout):
```
NAME                       READY   STATUS        RESTARTS   AGE
cleaner-86b7758668-9pw6t   2/2     Running       0          14s
cleaner-86b7758668-qgh4v   2/2     Running       0          9s
```

**Step 7: Check the sidecar logs**

```bash
k -n mercury logs cleaner-576967576c-cqtgx -c logger-con
```

Expected output:
```
init
Wed Sep 11 10:45:44 UTC 2099: remove random file
Wed Sep 11 10:45:45 UTC 2099: remove random file
Wed Sep 11 10:45:46 UTC 2099: remove random file
...
```

### Mystery Solved

The logs reveal that something is "removing random files" - this explains the missing data incidents! The sidecar pattern allows us to capture and monitor application logs effectively.

---

</details>

## Question 17 | InitContainer

**Solve this question on instance:** `ssh ckad5601`

Last lunch you told your coworker from department Mars Inc how amazing InitContainers are. Now he would like to see one in action. There is a Deployment yaml at `/opt/course/17/test-init-container.yaml`. This Deployment spins up a single Pod of image `nginx:1.17.3-alpine` and serves files from a mounted volume, which is empty right now.

Create an InitContainer named `init-con` which also mounts that volume and creates a file `index.html` with content `check this out!` in the root of the mounted volume. For this test we ignore that it doesn't contain valid html.

The InitContainer should be using image `busybox:1.31.0`. Test your implementation for example using curl from a temporary nginx:alpine Pod.

<details>
<summary>Answer</summary>

**Step 1: Copy and modify the existing Deployment**

```bash
cp /opt/course/17/test-init-container.yaml ~/17_test-init-container.yaml
vim 17_test-init-container.yaml
```

**Step 2: Add the InitContainer**

```yaml
# 17_test-init-container.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-init-container
  namespace: mars
spec:
  replicas: 1
  selector:
    matchLabels:
      id: test-init-container
  template:
    metadata:
      labels:
        id: test-init-container
    spec:
      volumes:
      - name: web-content
        emptyDir: {}
      initContainers:                 # initContainer start
      - name: init-con
        image: busybox:1.31.0
        command: ['sh', '-c', 'echo "check this out!" > /tmp/web-content/index.html']
        volumeMounts:
        - name: web-content
          mountPath: /tmp/web-content # initContainer end
      containers:
      - image: nginx:1.17.3-alpine
        name: nginx
        volumeMounts:
        - name: web-content
          mountPath: /usr/share/nginx/html
        ports:
        - containerPort: 80
```

**Step 3: Create the Deployment**

```bash
k -f 17_test-init-container.yaml create
```

**Step 4: Verify the Pod is running**

```bash
k -n mars get pod
```

Expected output:
```
NAME                                   READY   STATUS    RESTARTS   AGE
test-init-container-7d5c7b9f8d-xyz12   1/1     Running   0          30s
```

**Step 5: Test the configuration**

Get the Pod IP:

```bash
k -n mars get pod -o wide
```

Test with curl:

```bash
k run tmp --restart=Never --rm -i --image=nginx:alpine -- curl 10.0.0.67
```

Expected output:
```
check this out!
```
This pattern is perfect for ensuring that the nginx server has the required content before it starts serving requests.

</details>