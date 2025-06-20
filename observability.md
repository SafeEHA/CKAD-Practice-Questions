# Observability - Practice Questions

This section covers Kubernetes observability concepts including health probes, debugging, monitoring, and troubleshooting application issues.

## Preview Question | Liveness Probe Implementation

**Solve this question on instance:** `ssh ckad9043`

**Problem:** In Namespace `pluto` there is a Deployment named `project-23-api`. It has been working okay for a while but Team Pluto needs it to be more reliable. Implement a liveness-probe which checks the container to be reachable on port 80. Initially the probe should wait 10, periodically 15 seconds.

The original Deployment yaml is available at `/opt/course/p1/project-23-api.yaml`. Save your changes at `/opt/course/p1/project-23-api-new.yaml` and apply the changes.

<details>
<summary>Answer</summary>

1. **Get overview of current resources:**
```bash
k -n pluto get all -o wide
```

Note the running pods and their IP addresses.

2. **Test current pod connectivity (optional verification):**
```bash
# Using curl
k run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5 <POD_IP>

# Using wget
k run tmp --restart=Never --rm --image=busybox -i -- wget -O- <POD_IP>
```

Both should return nginx welcome page, confirming pods are working.

3. **Copy the original deployment file:**
```bash
cp /opt/course/p1/project-23-api.yaml /opt/course/p1/project-23-api-new.yaml
```

4. **Edit the deployment to add liveness probe:**
```bash
vim /opt/course/p1/project-23-api-new.yaml
```

**Add the liveness probe configuration:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: project-23-api
  namespace: pluto
spec:
  replicas: 3
  selector:
    matchLabels:
      app: project-23-api
  template:
    metadata:
      labels:
        app: project-23-api
    spec:
      containers:
      - image: httpd:2.4-alpine
        name: httpd
        # ... other container config ...
        livenessProbe:                  # ADD THIS SECTION
          tcpSocket:                    # Use TCP socket check
            port: 80                    # Check port 80
          initialDelaySeconds: 10       # Wait 10 seconds before first probe
          periodSeconds: 15             # Check every 15 seconds
```

5. **Apply the updated deployment:**
```bash
k apply -f /opt/course/p1/project-23-api-new.yaml
```

6. **Verify pods are running with liveness probe:**
```bash
k -n pluto get pod
```

Wait for the rollout to complete (new pods should appear).

7. **Verify liveness probe configuration:**
```bash
# Check on a specific pod
k -n pluto describe pod <pod-name> | grep Liveness

# Check on the deployment
k -n pluto describe deploy project-23-api | grep Liveness
```

Should show: `Liveness: tcp-socket :80 delay=10s timeout=1s period=15s #success=1 #failure=3`
---

</details>

## Preview Question | Service Troubleshooting

**Solve this question on instance:** `ssh ckad5601`

**Problem:** Management of EarthAG recorded that one of their Services stopped working. Dirk, the administrator, left already for the long weekend. All the information they could give you is that it was located in Namespace `earth` and that it stopped working after the latest rollout. All Services of EarthAG should be reachable from inside the cluster.

Find the Service, fix any issues and confirm it's working again. Write the reason of the error into file `/opt/course/p3/ticket-654.txt` so Dirk knows what the issue was.

<details>
<summary>Answer</summary>

1. **Get overview of all resources in earth namespace:**
```bash
k -n earth get all
```

Look for:
- Pods not in Ready state (0/1 instead of 1/1)
- Deployments with incorrect Ready counts
- Multiple ReplicaSets (indicates recent rollouts)

2. **Check service endpoints:**
```bash
k -n earth get ep
```

Services without endpoints indicate connectivity issues.

3. **Test service connectivity:**
```bash
# Test each service
k run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5 <service-name>.earth:<port>
```

Identify which service fails to connect (timeouts).

4. **Investigate the problematic deployment:**
```bash
k -n earth get deploy <deployment-name>
```

Look for deployments where READY count doesn't match desired replicas.

5. **Check deployment configuration:**
```bash
k -n earth edit deploy <deployment-name>
```

Common issues to look for:
- **Incorrect probe ports** (readiness/liveness probes pointing to wrong ports)
- **Image pull issues**
- **Resource constraints**
- **Container startup failures**

6. **Fix the issue (example - incorrect readiness probe port):**
```yaml
spec:
  template:
    spec:
      containers:
      - name: nginx
        readinessProbe:
          tcpSocket:
            port: 80          # Change from incorrect port (e.g., 82) to correct port
          initialDelaySeconds: 10
          periodSeconds: 20
```

7. **Wait for deployment to roll out:**
```bash
k -n earth get pod -l <deployment-selector>
```

Wait for initialDelaySeconds, then check pods become Ready (1/1).

8. **Verify service is working:**
```bash
k run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5 <service-name>.earth:<port>
```

Should now return successful response.

9. **Document the issue:**
```bash
vim /opt/course/p3/ticket-654.txt
```

**Example content:**
```
Service earth-3cc-web was failing because the readiness probe was configured 
to check port 82 instead of port 80. After the latest rollout, pods were 
running but not passing readiness checks, so they weren't added to service 
endpoints. Fixed by correcting the readiness probe port to 80.
```


### Common Troubleshooting Commands

```bash
# Check service endpoints
kubectl get endpoints

# Describe service for detailed info
kubectl describe service <service-name>

# Check pod readiness and restart counts
kubectl get pods -o wide

# Check deployment rollout status
kubectl rollout status deployment/<deployment-name>

# View deployment rollout history
kubectl rollout history deployment/<deployment-name>

# Check pod logs for errors
kubectl logs <pod-name>

# Describe pod for events and status
kubectl describe pod <pod-name>
```
</details>

## Question 6 | ReadinessProbe

**Solve this question on instance:** `ssh ckad5601`

Create a single **Pod** named **pod6** in **Namespace** **default** of image **busybox:1.31.0**. The **Pod** should have a readiness-probe executing **cat /tmp/ready**. It should initially wait 5 and periodically wait 10 seconds. This will set the container ready only if the file **/tmp/ready** exists.

The **Pod** should run the command **touch /tmp/ready && sleep 1d**, which will create the necessary file to be ready and then idles. Create the **Pod** and confirm it starts.

<details>
<summary>Answer</summary>

```bash
# Step 1: Generate the Pod YAML with the required command
k run pod6 --image=busybox:1.31.0 --dry-run=client -oyaml --command -- sh -c "touch /tmp/ready && sleep 1d" > 6.yaml

# Step 2: Edit the YAML file to add the readiness probe
vim 6.yaml
```

**Search for a readiness-probe example on https://kubernetes.io/docs, then copy and alter the relevant section for the task:**

```yaml
# 6.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod6
  name: pod6
spec:
  containers:
  - command:
    - sh
    - -c
    - touch /tmp/ready && sleep 1d
    image: busybox:1.31.0
    name: pod6
    resources: {}
    readinessProbe:                             # add
      exec:                                     # add
        command:                                # add
        - sh                                    # add
        - -c                                    # add
        - cat /tmp/ready                        # add
      initialDelaySeconds: 5                    # add
      periodSeconds: 10                         # add
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

**Step 3: Create the Pod**

```bash
k -f 6.yaml create
```

**Step 4: Verify the Pod status**

Running `k get pod pod6` we should see the job being created and completed:

```bash
➜ k get pod pod6
NAME   READY   STATUS              RESTARTS   AGE
pod6   0/1     ContainerCreating   0          2s

➜ k get pod pod6
NAME   READY   STATUS    RESTARTS   AGE
pod6   0/1     Running   0          7s

➜ k get pod pod6
NAME   READY   STATUS    RESTARTS   AGE
pod6   1/1     Running   0          15s
```

We see that the **Pod** is finally ready.

</details>

