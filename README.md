# THIS IS TEMPORARILY SET TO RUN ENTERPRISE METRICS. SET ENTERPRISE TO FALSE AND REMOVE LICENSE IF RUNNING IN OSS 


# This repo was created to deploy the Grafana Stack via Helm with custom values from ArgoCD utilizing multiple sources. This demo utilizes a GKE Autopilot cluster. 
To note: These repos were developed for rapid redeployment of the Grafana based on the repo maintainers knowledge and experience around Kubernetes, this is not official Grafana Documentation. This repo is a base for the Grafana OSS stack, there are sister repos for the enterprise versions. 
A rapidly redployable stack should be present in any enterprise environment, these examples can be used as a foundation but should not be the be all end all of MTTR. This is in no way for production use nor is it official Grafana Labs documentation. 

### OpenShift Users. There are multiple changes needed to allow these charts to work on OpenShift, from skipping CRDs to user/security policies. 


### Intermediate or above Kubernetes knowledge is assumed, if you've made it this far. 


## The Why
The core Helm charts are already created, we simply point the ACD app to the main charts and to our values file. Efficiency, repeatability and accountability. Utilizing GitOps approaches, one can (theoretically) lock down a cluster with all actions that take place doing so only after a peer review and merge request has been completed and approved. I have worked with large enterprise customers where only a handful of people had direct cluster access, all changes took place via GitOps; Branch, edit, peer review, merge request and move forward. The "oops" moments were not completely gone but were reduced. Changes (if not done with direct cluster/machine access) were auditable in a single place (Git). 

## End Goals
In the end, I would like to setup 3 demo paths, the stack, the datasources and the dashboards. As will all things OpenSource, there are many ways these can be done. The end objective is to be sure you learn and come away from this with more confidence and knowledge for your day to day computing experiences. 

The demos rely on the following:
- A working Kubernetes cluster of your choice.
- ArgoCD installed and working properly.
  
## For a minimal Grafana functioning setup, you will usually need various manifests:

- A secret for your admin user and password (utilize the External Secret Operator for full workflows), this demo assumes manual creation of the secrets.
  - Note that when you base64 your user/pass for the secret you should use `-n`
- The ArgoCD application utilizing mulltiple sources (one pointing to the Grafana Helm chart, one to the directory with overrides)
- Your Helm overrides file

Examples of the manifests can be found below. [Secret storage in Kubernetes is not natively secure](https://kubernetes.io/docs/concepts/configuration/secret/#:~:text=%23%20values%20are%20base64,level%20of%20confidentiality), secrets are merely base64 encoded and should not be stored in Git. 
[The ExternalSecretOperator](https://external-secrets.io/latest/) paired with a Vault back end (or other secure secrets manager) is recommended. There are multiple approaches to external secrets, [this](https://www.redhat.com/en/blog/external-secrets-with-hashicorp-vault) linked article is an aging but good foundational knowledge for this approach. For this base tutorial, secrets are created by the user  prior to deploying the stack but this brings me to another point; [Kubernetes is a declarative system](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/), imperative approaches can be used for testing and learning but ultimately declarative approaches are desired. The focus of these repos is not the ESO or Vault or imperative vs declarative approaches but it is important to call these out for general knowledge. 

### The admin user secret (Grafana namespace)
```bash
apiVersion: v1
data:
  adminPassword: <your pass base 64 encoded>
  adminUser: <your admin user base64 encoded>
kind: Secret
metadata:
  name: admin-user
  namespace: grafana-prod
type: Opaque
```

### The minio bucket secret (each namespace, mimir, tempo, loki - name accordingly)
```
apiVersion: v1
data:
  MIMIRS3KEY: <>
kind: Secret
metadata:
  name: mimir-bucket-secret
  namespace: loki
type: Opaque
```
### The ArgoCD Application
```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mimir-distributed 
  namespace: argocd
spec:
  destination:
    namespace: mimir    
    server: https://kubernetes.default.svc
  sources:
    - repoURL: https://grafana.github.io/helm-charts
      targetRevision: 5.4.1 
      chart: mimir-distributed 
      helm:
        valueFiles:
          - $values/values/mimir-overrides.yaml
    - repoURL: https://github.com/pbmoses/grafana-stack-argo.git 
      targetRevision: HEAD 
      ref: values
  project: default
```
### The overrides file
[Values that you have determined are best for your environment](./values)

Deploy each of the secrets `kubectl create -f <manifest>`. You can either apply the ArgoCD application imperatively `kubectl create -f application.yaml`or also place this in a repo to allow ArgoCD to sync from it, beginning an app of apps pattern. Your values file should be present in the source that ArgoCD is pointing to. 

Upon successful syncronization and deployment of the Grafana front end, you can retrieve the initial admin password with the following command: `kubectl get secret admin-user -o jsonpath="{.data.adminPassword}" | base64 -d`

### An ArgoCD base application deployed and sync'd
<img width="1253" alt="Screenshot 2024-10-12 at 9 48 37 AM" src="https://github.com/user-attachments/assets/d403af90-8af5-4151-8091-a693f0dd4257">


### A  Grafana application deployed and sync'd
<img width="1294" alt="Screenshot 2024-10-12 at 9 33 41 AM" src="https://github.com/user-attachments/assets/3b381886-ee6b-4a5d-9b0b-244c90cedb20">


### A deployed Grafana front end with successful login 
<img width="1444" alt="Screenshot 2024-10-12 at 9 08 44 AM" src="https://github.com/user-attachments/assets/a7242f70-2649-4e50-9518-01307dcefb1c">

### Following this, check in for similar demos for Mimir, Loki, Tempo, Datasources and Dashboards

* Disclaimer. Best efforts are approached and we're all human. Use as a foundation, deploy, verify and approve in your own environment with developed best practices  similar to a pattern of sandbox/dev, staging and prod (or similar). 
