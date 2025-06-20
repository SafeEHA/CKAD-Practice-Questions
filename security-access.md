# Security and Access - Practice Questions

This section covers ServiceAccounts, Secrets, RBAC, and security configurations in Kubernetes.

## Preview Question | Deployment with ServiceAccount

**Solve this question on instance:** `ssh ckad9043`

**Problem:** Team Sun needs a new Deployment named `sunny` with 4 replicas of image `nginx:1.17.3-alpine` in Namespace `sun`. The Deployment and its Pods should use the existing ServiceAccount `sa-sun-deploy`.

Expose the Deployment internally using a ClusterIP Service named `sun-srv` on port 9999. The nginx containers should run as default on port 80. The management of Team Sun would like to execute a command to check that all Pods are running on occasion. Write that command into file `/opt/course/p2/sunny_status_command.sh`.

<details>
<summary>Answer</summary>

1. **Generate deployment YAML template:**
```bash
k -n sun create deployment sunny --image=nginx:1.17.3-alpine --dry-run=client -o yaml > p2_sunny.yaml
```

2. **Edit the deployment to add requirements:**
```bash
vim p2_sunny.yaml
```

**Updated deployment configuration:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: sunny
  name: sunny
  namespace: sun
spec:
  replicas: 4                               # Change from default 1
  selector:
    matchLabels:
      app: sunny
  template:
    metadata:
      labels:
        app: sunny
    spec:
      serviceAccountName: sa-sun-deploy     # Add ServiceAccount
      containers:
      - image: nginx:1.17.3-alpine
        name: nginx
```

3. **Create the deployment:**
```bash
k create -f p2_sunny.yaml
```

4. **Verify deployment is running:**
```bash
k -n sun get pod
```

Check the AGE column - new pods should appear with recent timestamps.

5. **Expose the deployment with a service:**
```bash
k -n sun expose deployment sunny --name sun-srv --port 9999 --target-port 80
```

This creates a ClusterIP service automatically with correct selectors.

6. **Test the service connectivity:**
```bash
k run tmp --restart=Never --rm -i --image=nginx:alpine -- curl -m 5 sun-srv.sun:9999
```

Should return nginx welcome page. Note the cross-namespace DNS: `sun-srv.sun`

7. **Create the status check command:**
```bash
vim /opt/course/p2/sunny_status_command.sh
```

**Command content:**
```bash
#!/bin/bash
kubectl -n sun get deployment sunny
```

8. **Test the status command:**
```bash
sh /opt/course/p2/sunny_status_command.sh
```

Should show: `sunny   4/4     4            4           <AGE>`

</details>

## Question 5 | ServiceAccount, Secret

**Solve this question on instance:** `ssh ckad7326`

Team Neptune has its own ServiceAccount named `neptune-sa-v2` in Namespace `neptune`. A coworker needs the token from the Secret that belongs to that ServiceAccount. Write the base64 decoded token to file `/opt/course/5/token` on ckad7326.

<details>
<summary>Answer</summary>

**Important Note:** Since K8s 1.24, Secrets won't be created automatically for ServiceAccounts any longer. However, it's still possible to create a Secret manually and attach it to a ServiceAccount by setting the correct annotation on the Secret. This was done for this task.

First, explore the namespace to find the relevant Secret:
```bash
k -n neptune get sa # get overview
k -n neptune get secrets # shows all secrets of namespace
k -n neptune get secrets -oyaml | grep annotations -A 1 # shows secrets with first annotation
```

If a Secret belongs to a ServiceAccount, it'll have the annotation `kubernetes.io/service-account.name`. In this case, the Secret we're looking for is `neptune-secret-1`.

Get the Secret details:
```bash
➜ k -n neptune get secret neptune-secret-1 -o yaml
apiVersion: v1
data:
...
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkltNWFaRmRxWkRKMmFHTnZRM0JxV0haT1IxZzFiM3BJY201SlowaEhOV3hUWmt3elFuRmFhVEZhZDJNaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUp1WlhCMGRXNWxJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbTVsY0hSMWJtVXRjMkV0ZGpJdGRHOXJaVzR0Wm5FNU1tb2lMQ0pyZFdKbGNtNWxkR1Z6TG1sdkwzTmxjblpwWTJWaFkyTnZkVzUwTDNObGNuWnBZMlV0WVdOamIzVnVkQzV1WVcxbElqb2libVZ3ZEhWdVpTMXpZUzEyTWlJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZ5ZG1salpTMWhZMk52ZFc1MExuVnBaQ0k2SWpZMlltUmpOak0yTFRKbFl6TXROREpoWkMwNE9HRTFMV0ZoWXpGbFpqWmxPVFpsTlNJc0luTjFZaUk2SW5ONWMzUmxiVHB6WlhKMmFXTmxZV05qYjNWdWREcHVaWEIwZFc1bE9tNWxjSFIxYm1VdGMyRXRkaklpZlEuVllnYm9NNENUZDBwZENKNzh3alV3bXRhbGgtMnZzS2pBTnlQc2gtNmd1RXdPdFdFcTVGYnc1WkhQdHZBZHJMbFB6cE9IRWJBZTRlVU05NUJSR1diWUlkd2p1Tjk1SjBENFJORmtWVXQ0OHR3b2FrUlY3aC1hUHV3c1FYSGhaWnp5NHlpbUZIRzlVZm1zazVZcjRSVmNHNm4xMzd5LUZIMDhLOHpaaklQQXNLRHFOQlF0eGctbFp2d1ZNaTZ2aUlocnJ6QVFzME1CT1Y4Mk9KWUd5Mm8tV1FWYzBVVWFuQ2Y5NFkzZ1QwWVRpcVF2Y3pZTXM2bno5dXQtWGd3aXRyQlk2VGo5QmdQcHJBOWtfajVxRXhfTFVVWlVwUEFpRU43T3pka0pzSThjdHRoMTBseXBJMUFlRnI0M3Q2QUx5clFvQk0zOWFiRGZxM0Zrc1Itb2NfV013
kind: Secret
...
```

This shows the base64 encoded token. To get the decoded token, use the describe command which automatically decodes it:

```bash
➜ k -n neptune describe secret neptune-secret-1
...
Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Im5aZFdqZDJ2aGNvQ3BqWHZOR1g1b3pIcm5JZ0hHNWxTZkwzQnFaaTFad2MifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJuZXB0dW5lIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6Im5lcHR1bmUtc2EtdjItdG9rZW4tZnE5MmoiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoibmVwdHVuZS1zYS12MiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjY2YmRjNjM2LTJlYzMtNDJhZC04OGE1LWFhYzFlZjZlOTZlNSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpuZXB0dW5lOm5lcHR1bmUtc2EtdjIifQ.VYgboM4CTd0pdCJ78wjUwmtalh-2vsKjANyPsh-6guEwOtWEq5Fbw5ZHPtvAdrLlPzpOHEbAe4eUM95BRGWbYIdwjuN95J0D4RNFkVUt48twoakRV7h-aPuwsQXHhZZzy4yimFHG9Ufmsk5Yr4RVcG6n137y-FH08K8zZjIPAsKDqNBQtxg-lZvwVMi6viIhrrzAQs0MBOV82OJYGy2o-WQVc0UUanCf94Y3gT0YTiqQvczYMs6nz9ut-XgwitrBY6Tj9BgPprA9k_j5qEx_LUUZUpPAiEN7OzdkJsI8ctth10lypI1AeFr43t6ALyrQoBM39abDfq3FksR-oc_WMw
ca.crt:     1066 bytes
namespace:  7 bytes
```

Copy the token (the part under `token:`) and save it to the required file:

```bash
vim /opt/course/5/token
```

The file `/opt/course/5/token` should contain the decoded token:

```bash
# /opt/course/5/token
eyJhbGciOiJSUzI1NiIsImtpZCI6Im5aZFdqZDJ2aGNvQ3BqWHZOR1g1b3pIcm5JZ0hHNWxTZkwzQnFaaTFad2MifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJuZXB0dW5lIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6Im5lcHR1bmUtc2EtdjItdG9rZW4tZnE5MmoiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoibmVwdHVuZS1zYS12MiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjY2YmRjNjM2LTJlYzMtNDJhZC04OGE1LWFhYzFlZjZlOTZlNSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpuZXB0dW5lOm5lcHR1bmUtc2EtdjIifQ.VYgboM4CTd0pdCJ78wjUwmtalh-2vsKjANyPsh-6guEwOtWEq5Fbw5ZHPtvAdrLlPzpOHEbAe4eUM95BRGWbYIdwjuN95J0D4RNFkVUt48twoakRV7h-aPuwsQXHhZZzy4yimFHG9Ufmsk5Yr4RVcG6n137y-FH08K8zZjIPAsKDqNBQtxg-lZvwVMi6viIhrrzAQs0MBOV82OJYGy2o-WQVc0UUanCf94Y3gT0YTiqQvczYMs6nz9ut-XgwitrBY6Tj9BgPprA9k_j5qEx_LUUZUpPAiEN7OzdkJsI8ctth10lypI1AeFr43t6ALyrQoBM39abDfq3FksR-oc_WMw
```

</details>

## Question 14 | Secret, Secret-Volume, Secret-Env

**Solve this question on instance:** `ssh ckad9043`

You need to make changes on an existing Pod in Namespace `moon` called `secret-handler`. Create a new Secret `secret1` which contains `user=test` and `pass=pwd`. The Secret's content should be available in Pod `secret-handler` as environment variables `SECRET1_USER` and `SECRET1_PASS`. The yaml for Pod `secret-handler` is available at `/opt/course/14/secret-handler.yaml`.

There is existing yaml for another Secret at `/opt/course/14/secret2.yaml`, create this Secret and mount it inside the same Pod at `/tmp/secret2`. Your changes should be saved under `/opt/course/14/secret-handler-new.yaml` on ckad9043. Both Secrets should only be available in Namespace `moon`.

<details>
<summary>Answer</summary>

**Step 1: Create Secret1 with literal values**

```bash
k -n moon create secret generic secret1 --from-literal user=test --from-literal pass=pwd
```

This command generates a Secret equivalent to:

```yaml
apiVersion: v1
data:
  pass: cHdk
  user: dGVzdA==
kind: Secret
metadata:
  creationTimestamp: null
  name: secret1
  namespace: moon
```

**Step 2: Create Secret2 from existing yaml**

```bash
k -n moon -f /opt/course/14/secret2.yaml create
```

**Step 3: Verify both secrets exist**

```bash
k -n moon get secret
```

Expected output:
```
NAME                  TYPE                                  DATA   AGE
default-token-rvzcf   kubernetes.io/service-account-token   3      66m
secret1               Opaque                                2      4m3s
secret2               Opaque                                1      8s
```

**Step 4: Modify the Pod to use both Secrets**

Copy the original Pod yaml:

```bash
cp /opt/course/14/secret-handler.yaml /opt/course/14/secret-handler-new.yaml
vim /opt/course/14/secret-handler-new.yaml
```

Add the Secret configurations:

```yaml
# /opt/course/14/secret-handler-new.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    id: secret-handler
    uuid: 1428721e-8d1c-4c09-b5d6-afd79200c56a
    red_ident: 9cf7a7c0-fdb2-4c35-9c13-c2a0bb52b4a9
    type: automatic
  name: secret-handler
  namespace: moon
spec:
  volumes:
  - name: cache-volume1
    emptyDir: {}
  - name: cache-volume2
    emptyDir: {}
  - name: cache-volume3
    emptyDir: {}
  - name: secret2-volume              # add
    secret:                           # add
      secretName: secret2             # add
  containers:
  - name: secret-handler
    image: bash:5.0.11
    args: ['bash', '-c', 'sleep 2d']
    volumeMounts:
    - mountPath: /cache1
      name: cache-volume1
    - mountPath: /cache2
      name: cache-volume2
    - mountPath: /cache3
      name: cache-volume3
    - name: secret2-volume            # add
      mountPath: /tmp/secret2         # add
    env:
    - name: SECRET_KEY_1
      value: ">8$kH#kj..i8}HImQd{"
    - name: SECRET_KEY_2
      value: "IO=a4L/XkRdvN8jM=Y+"
    - name: SECRET_KEY_3
      value: "-7PA0_Z]>{pwa43r)__"
    - name: SECRET1_USER              # add
      valueFrom:                      # add
        secretKeyRef:                 # add
          name: secret1               # add
          key: user                   # add
    - name: SECRET1_PASS              # add
      valueFrom:                      # add
        secretKeyRef:                 # add
          name: secret1               # add
          key: pass                   # add
```

**Step 5: Apply the changes**

Delete the old pod and create the new one:

```bash
k -f /opt/course/14/secret-handler.yaml delete --force --grace-period=0
k -f /opt/course/14/secret-handler-new.yaml create
```

Alternative approach using replace:

```bash
k -f /opt/course/14/secret-handler-new.yaml replace --force --grace-period=0
```

**Step 6: Verify the configuration works**

Test environment variables:

```bash
k -n moon exec secret-handler -- env | grep SECRET1
```

Expected output:
```
SECRET1_USER=test
SECRET1_PASS=pwd
```

Test volume mount:

```bash
k -n moon exec secret-handler -- find /tmp/secret2
```

Expected output:
```
/tmp/secret2
/tmp/secret2/..data
/tmp/secret2/key
/tmp/secret2/..2019_09_11_09_03_08.147048594
/tmp/secret2/..2019_09_11_09_03_08.147048594/key
```

Check secret content:

```bash
k -n moon exec secret-handler -- cat /tmp/secret2/key
```

Expected output:
```
12345678
```

### Alternative Environment Variable Method

Instead of individual `secretKeyRef` entries, you can import all keys from a Secret:

```yaml
containers:
- name: secret-handler
  # ...
  envFrom:
  - secretRef:        # also works for configMapRef
      name: secret1
```

</details>