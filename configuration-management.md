# Configuration Management - Practice Questions

This section covers Helm package management, application deployment, and configuration handling.

## Question 4 | Helm Management

**Solve this question on instance:** `ssh ckad7326`

Team Mercury asked you to perform some operations using Helm, all in Namespace `mercury`:

1. Delete release `internal-issue-report-apiv1`
2. Upgrade release `internal-issue-report-apiv2` to any newer version of chart `bitnami/nginx` available
3. Install a new release `internal-issue-report-apache` of chart `bitnami/apache`. The Deployment should have two replicas, set these via Helm-values during install
4. There seems to be a broken release, stuck in pending-install state. Find it and delete it

<details>
<summary>Answer</summary>

**Background Information:**
- **Helm Chart:** Kubernetes YAML template-files combined into a single package, Values allow customisation
- **Helm Release:** Installed instance of a Chart
- **Helm Values:** Allow to customise the YAML template-files in a Chart when creating a Release

### 1. Delete the required release:

```bash
➜ helm -n mercury ls
NAME                            NAMESPACE     STATUS          CHART           APP VERSION
internal-issue-report-apiv1     mercury       deployed        nginx-9.5.0     1.21.1     
internal-issue-report-apiv2     mercury       deployed        nginx-9.5.0     1.21.1     
internal-issue-report-app       mercury       deployed        nginx-9.5.0     1.21.1  

➜ helm -n mercury uninstall internal-issue-report-apiv1
release "internal-issue-report-apiv1" uninstalled
```

### 2. Upgrade the release to a newer chart version:

First, check for available chart versions:
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
bitnami/nginx         9.5.2           1.21.1          Chart for the nginx server
```

Upgrade to the newer version:
```bash
➜ helm -n mercury upgrade internal-issue-report-apiv2 bitnami/nginx
Release "internal-issue-report-apiv2" has been upgraded. Happy Helming!
NAME: internal-issue-report-apiv2
LAST DEPLOYED: Tue Aug 31 17:40:42 2021
NAMESPACE: mercury
STATUS: deployed
REVISION: 2

➜ helm -n mercury ls
NAME                            NAMESPACE     STATUS          CHART           APP VERSION
internal-issue-report-apiv2     mercury       deployed        nginx-9.5.2     1.21.1     
internal-issue-report-app       mercury       deployed        nginx-9.5.0     1.21.1 
```

**INFO:** Also check out `helm rollback` for undoing a helm rollout/upgrade

### 3. Install new release with custom replica count:

You can view all possible values with:
```bash
helm show values bitnami/apache # shows all configurable values
```

Install with custom replica count:
```bash
➜ helm -n mercury install internal-issue-report-apache bitnami/apache --set replicaCount=2
NAME: internal-issue-report-apache
LAST DEPLOYED: Tue Aug 31 17:57:23 2021
NAMESPACE: mercury
STATUS: deployed
REVISION: 1
```

For multiple values, you can chain them:
```bash
helm -n mercury install internal-issue-report-apache bitnami/apache \
  --set replicaCount=2 \
  --set image.debug=true
```

Verify the installation:
```bash
➜ helm -n mercury ls
NAME                            NAMESPACE     STATUS          CHART           APP VERSION
internal-issue-report-apache    mercury       deployed        apache-8.6.3    2.4.48

➜ k -n mercury get deploy internal-issue-report-apache
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
internal-issue-report-apache   2/2     2            2           96s
```

### 4. Find and delete the broken release:

List all releases including those in failed states:
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

</details>