# Services and Networking - Practice Questions

This section covers Kubernetes networking concepts including Services, DNS resolution, NodePort configuration, and NetworkPolicies for traffic control.

## Question 10 | Service, Logs

**Solve this question on instance:** `ssh ckad9043`

Team Pluto needs a new cluster internal Service. Create a ClusterIP Service named project-plt-6cc-svc in Namespace pluto. This Service should expose a single Pod named project-plt-6cc-api of image nginx:1.17.3-alpine, create that Pod as well. The Pod should be identified by label project: plt-6cc-api. The Service should use tcp port redirection of 3333:80.

Finally use for example curl from a temporary nginx:alpine Pod to get the response from the Service. Write the response into /opt/course/10/service_test.html on ckad9043. Also check if the logs of Pod project-plt-6cc-api show the request and write those into /opt/course/10/service_test.log on ckad9043.

<details>
<summary>Answer</summary>

```bash
k -n pluto run project-plt-6cc-api --image=nginx:1.17.3-alpine --labels project=plt-6cc-api
```

This will create the requested Pod. In yaml it would look like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    project: plt-6cc-api
  name: project-plt-6cc-api
spec:
  containers:
  - image: nginx:1.17.3-alpine
    name: project-plt-6cc-api
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Next we create the service:

```bash
k -n pluto expose pod -h # help

k -n pluto expose pod project-plt-6cc-api --name project-plt-6cc-svc --port 3333 --target-port 80
```

Expose will create a yaml where everything is already set for our case and no need to change anything:

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    project: plt-6cc-api
  name: project-plt-6cc-svc   # good
  namespace: pluto            # great
spec:
  ports:
  - port: 3333                # awesome
    protocol: TCP
    targetPort: 80            # nice
  selector:
    project: plt-6cc-api      # beautiful
status:
  loadBalancer: {}
```

We could also use create service but then we would need to change the yaml afterwards:

```bash
k -n pluto create service -h # help
k -n pluto create service clusterip -h #help
k -n pluto create service clusterip project-plt-6cc-svc --tcp 3333:80 --dry-run=client -oyaml
# now we would need to set the correct selector labels
```

Check the Service is running:

```bash
➜ k -n pluto get pod,svc | grep 6cc
pod/project-plt-6cc-api         1/1     Running   0          9m42s

service/project-plt-6cc-svc   ClusterIP   10.31.241.234   <none>        3333/TCP   2m24s
```

Does the Service has one Endpoint?

```bash
➜ k -n pluto describe svc project-plt-6cc-svc
Name:              project-plt-6cc-svc
Namespace:         pluto
Labels:            project=plt-6cc-api
Annotations:       <none>
Selector:          project=plt-6cc-api
Type:              ClusterIP
IP:                10.3.244.240
Port:              <unset>  3333/TCP
TargetPort:        80/TCP
Endpoints:         10.28.2.32:80 
Session Affinity:  None
Events:            <none>
```

Or even shorter:

```bash
➜ k -n pluto get ep
NAME                  ENDPOINTS       AGE
project-plt-6cc-svc   10.28.2.32:80   84m
```

Yes, endpoint there! Finally we check the connection using a temporary Pod:

```bash
➜ k run tmp --restart=Never --rm --image=nginx:alpine -i -- curl http://project-plt-6cc-svc.pluto:3333
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0  32210      0 --:--:-- --:--:-- --:--:-- 32210
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
...
```

Great! Notice that we use the Kubernetes Namespace dns resolving (project-plt-6cc-svc.pluto) here. We could only use the Service name if we would also spin up the temporary Pod in Namespace pluto .

And now really finally copy or pipe the html content into /opt/course/10/service_test.html.

```bash
# /opt/course/10/service_test.html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
...
```

Also the requested logs:

```bash
k -n pluto logs project-plt-6cc-api > /opt/course/10/service_test.log
```

```bash
# /opt/course/10/service_test.log
10.44.0.0 - - [22/Jan/2021:23:19:55 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.69.1" "-"
```
</details>

## Question 18 | Service Misconfiguration

**Solve this question on instance:** `ssh ckad5601`

**Problem:** There seems to be an issue in Namespace `mars` where the ClusterIP service `manager-api-svc` should make the Pods of Deployment `manager-api-deployment` available inside the cluster. You can test this with `curl manager-api-svc.mars:4444` from a temporary nginx:alpine Pod. Check for the misconfiguration and apply a fix.

<details>
<summary>Answer</summary>

1. **Get an overview of resources in the mars namespace:**
```bash
k -n mars get all
```

Expected output shows running pods, service, and deployment but connection tests fail.

2. **Test the service connection:**
```bash
k -n mars run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5 manager-api-svc:4444
```

This should timeout, indicating a service configuration issue.

3. **Test direct pod connection to verify pods are working:**
```bash
k -n mars get pod -o wide  # get pod IPs
k -n mars run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5 <POD_IP>
```

If direct pod connection works, the issue is with the service configuration.

4. **Investigate the service endpoints:**
```bash
k -n mars describe service manager-api-svc
# or
k -n mars get ep
```

If endpoints show `<none>`, the service selector is incorrect.

5. **Check and fix the service selector:**
```bash
k -n mars edit service manager-api-svc
```

**Key Fix:** Change the selector from deployment name to pod labels:
```yaml
spec:
  selector:
    # id: manager-api-deployment  # WRONG - points to deployment
    id: manager-api-pod           # CORRECT - points to pod labels
```

6. **Verify endpoints are now populated:**
```bash
k -n mars get ep
```

Should now show multiple endpoints with pod IPs and ports.

7. **Test the service connection again:**
```bash
k -n mars run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5 manager-api-svc:4444
```

Should now return the nginx welcome page.

### Key Learning Points

- **Services select Pods directly**, not Deployments
- Service selectors must match Pod labels exactly
- Use `kubectl get ep` to verify service endpoints
- Cross-namespace DNS: `service-name.namespace.svc.cluster.local`

---

</details>

## Question 19 | Service ClusterIP to NodePort

**Solve this question on instance:** `ssh ckad5601`

**Problem:** In Namespace `jupiter` you'll find an apache Deployment (with one replica) named `jupiter-crew-deploy` and a ClusterIP Service called `jupiter-crew-svc` which exposes it. Change this service to a NodePort one to make it available on all nodes on port 30100. Test the NodePort Service using the internal IP of all available nodes and the port 30100 using curl.

<details>
<summary>Answer</summary>

1. **Get overview of jupiter namespace:**
```bash
k -n jupiter get all
```

2. **Optional: Test the current ClusterIP service:**
```bash
k -n jupiter run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5 jupiter-crew-svc:8080
```

3. **Convert ClusterIP to NodePort service:**
```bash
k -n jupiter edit service jupiter-crew-svc
```

**Key Changes:**
```yaml
spec:
  ports:
  - name: 8080-80
    port: 8080
    protocol: TCP
    targetPort: 80
    nodePort: 30100      # ADD this line
  type: NodePort         # CHANGE from ClusterIP
```

4. **Verify the service type change:**
```bash
k -n jupiter get svc
```

Should show `NodePort` type with `8080:30100/TCP` in PORT(S) column.

5. **Get node internal IPs:**
```bash
k get nodes -o wide
```

Note the INTERNAL-IP column values.

6. **Test external access via NodePort:**
```bash
curl <NODE_INTERNAL_IP>:30100
```

Should return the apache "It works!" page.

7. **Optional: Verify ClusterIP still works:**
```bash
k -n jupiter run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5 jupiter-crew-svc:8080
```

---

</details>

## Question 20 | NetworkPolicy

**Solve this question on instance:** `ssh ckad7326`

**Problem:** In Namespace `venus` you'll find two Deployments named `api` and `frontend`. Both Deployments are exposed inside the cluster using Services. Create a NetworkPolicy named `np1` which restricts outgoing tcp connections from Deployment `frontend` and only allows those going to Deployment `api`. Make sure the NetworkPolicy still allows outgoing traffic on UDP/TCP ports 53 for DNS resolution.

Test using: `wget www.google.com` and `wget api:2222` from a Pod of Deployment frontend.

<details>
<summary>Answer</summary>

1. **Get overview of venus namespace:**
```bash
k -n venus get all
```

2. **Optional: Test current connectivity:**
```bash
# Test services work
k -n venus run tmp --restart=Never --rm -i --image=busybox -- wget -O- frontend:80
k -n venus run tmp --restart=Never --rm -i --image=busybox -- wget -O- api:2222

# Test frontend pod can reach external and internal
k -n venus exec <frontend-pod-name> -- wget -O- www.google.com
k -n venus exec <frontend-pod-name> -- wget -O- api:2222
```

3. **Create the NetworkPolicy:**
```bash
vim np1.yaml
```

**NetworkPolicy Configuration:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: np1
  namespace: venus
spec:
  podSelector:
    matchLabels:
      id: frontend          # Apply policy to frontend pods
  policyTypes:
  - Egress                  # Control outgoing traffic only
  egress:
  - to:                     # Rule 1: Allow egress to api pods
    - podSelector:
        matchLabels:
          id: api
  - ports:                  # Rule 2: Allow DNS resolution
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

4. **Apply the NetworkPolicy:**
```bash
k apply -f np1.yaml
```

5. **Test the restrictions:**
```bash
# External access should be blocked
k -n venus exec <frontend-pod-name> -- wget -O- -T 5 www.google.com
# Should timeout

# Internal api access should work
k -n venus exec <frontend-pod-name> -- wget -O- api:2222
# Should return "It works!" page
```

### NetworkPolicy Logic Examples

**Correct (OR logic):**
```yaml
egress:
- to: [...pod selector...]     # Rule 1
- ports: [...DNS ports...]     # Rule 2
# Traffic allowed if: (matches pods) OR (DNS ports)
```

**Incorrect (AND logic):**
```yaml
egress:
- to: [...pod selector...]     # Single rule
  ports: [...DNS ports...]     # Same rule, additional selector
# Traffic allowed if: (matches pods) AND (DNS ports)
```

---

</details>