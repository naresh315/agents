# Real GKE Java Microservice Deployment Example

This repository demonstrates how to implement a production-style Java/Spring Boot service on GKE using:

- Regional GKE cluster across multiple zones
- Private nodes + VPC-native networking
- Workload Identity
- Artifact Registry
- Kubernetes Deployment with 3 replicas
- Topology spread constraints across zones
- Pod anti-affinity across nodes
- Startup, readiness, and liveness probes
- CPU/memory requests and limits
- HPA for Pod scaling
- GKE node-pool autoscaling for node capacity
- ClusterIP Service for stable service discovery
- GCE Ingress for north-south HTTP routing
- PodDisruptionBudget for voluntary-disruption protection
- Optional node taints/tolerations + node affinity for specialized workloads
- Secret Manager + External Secrets Operator using Workload Identity
- Basic NetworkPolicy

## Architecture

```text
Internet
   |
   v
GCE HTTP(S) Load Balancer
   |
   v
Ingress
   |
   v
ClusterIP Service
   |
   +------------------------------+
   |              |               |
   v              v               v
Pod zone-a     Pod zone-b       Pod zone-c
Order v1      Order v1         Order v1
   |              |               |
   +--------------+---------------+
                  |
             Backend services

Autoscaling:
  HPA -> changes Pod count
  GKE Cluster/Node autoscaler -> changes node capacity

Secrets:
  Secret Manager -> External Secrets Operator -> Kubernetes Secret -> Pod
                         |
                   Workload Identity
```

## 1. Provision the infrastructure

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars and set project_id

gcloud auth application-default login

gcloud services enable \
  container.googleapis.com \
  compute.googleapis.com \
  artifactregistry.googleapis.com \
  secretmanager.googleapis.com \
  iamcredentials.googleapis.com

terraform init
terraform plan
terraform apply
```

## 2. Authenticate to the cluster

```bash
gcloud container clusters get-credentials java-platform-gke \
  --region us-central1 \
  --project YOUR_GCP_PROJECT_ID
```

Check the zones/nodes:

```bash
kubectl get nodes -L topology.kubernetes.io/zone,workload
```

## 3. Build and push the Java image

Assume your Spring Boot build creates `target/orders-service.jar`.

```bash
export PROJECT_ID=YOUR_GCP_PROJECT_ID
export REGION=us-central1
export IMAGE="$REGION-docker.pkg.dev/$PROJECT_ID/java-services/java-orders:1.0.0"

gcloud auth configure-docker "$REGION-docker.pkg.dev"
docker build -t "$IMAGE" .
docker push "$IMAGE"
```

Update `k8s/deployment.yaml` with the image URI.

## 4. Create the application secret

Create the Secret Manager version outside Terraform so the secret value is not written into Terraform state:

```bash
echo -n 'replace-with-real-secret' | gcloud secrets versions add orders-api-key \
  --data-file=- \
  --project "$PROJECT_ID"
```

## 5. Install External Secrets Operator

Pin the operator version approved by your organization rather than using a floating latest version. For example:

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets-system \
  --create-namespace
```

Verify:

```bash
kubectl get pods -n external-secrets-system
kubectl get crd | grep external-secrets
```

## 6. Deploy the application

Replace `YOUR_GCP_PROJECT_ID` in the manifests first.

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/serviceaccount.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secretstore.yaml
kubectl apply -f k8s/externalsecret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml
kubectl apply -f k8s/pdb.yaml
kubectl apply -f k8s/ingress.yaml
kubectl apply -f k8s/networkpolicy.yaml
```

Check rollout and secret synchronization:

```bash
kubectl -n orders get pods -o wide
kubectl -n orders get externalsecret
kubectl -n orders get secret java-orders-secrets
kubectl -n orders rollout status deployment/java-orders
```

## 7. Why each configuration exists

### Deployment / replicas

```yaml
replicas: 3
```

Three replicas provide basic redundancy. In production, the HPA can increase this to 20 based on resource utilization.

### Topology spread

```yaml
topologyKey: topology.kubernetes.io/zone
maxSkew: 1
whenUnsatisfiable: DoNotSchedule
```

Kubernetes attempts to distribute replicas evenly across zones. This protects against a zone-level failure.

### Pod anti-affinity

```yaml
topologyKey: kubernetes.io/hostname
```

This discourages two replicas from landing on the same node.

A useful combination is:

```text
Topology spread -> across zones
Pod anti-affinity -> across nodes
```

### Resource requests and limits

```yaml
requests:
  cpu: "500m"
  memory: "512Mi"
limits:
  cpu: "1"
  memory: "1Gi"
```

Requests influence scheduling and HPA calculations. Limits provide an upper resource boundary for the container.

### Probes

```text
Startup probe
    |
    v
Allows slow JVM startup without premature liveness failures

Readiness probe
    |
    v
Controls whether the Service should send traffic

Liveness probe
    |
    v
Restarts a container that is unhealthy/stuck
```

### HPA

The HPA changes the number of Pods:

```text
3 Pods -> high CPU -> 6 Pods -> 10 Pods
```

It does not create VM/node capacity itself.

### GKE node autoscaling

The node pool autoscaler changes node capacity:

```text
3 nodes -> Pods no longer fit -> 5 nodes
```

This is why HPA and node autoscaling are complementary.

### Service

The `ClusterIP` Service provides a stable virtual endpoint:

```text
http://java-orders.orders.svc.cluster.local
```

Pods can be recreated and IP addresses can change without clients needing to know.

### Ingress

Ingress exposes HTTP traffic north-south:

```text
Internet -> GCE Load Balancer -> Ingress -> Service -> Pods
```

For a new platform you can replace Ingress with GKE Gateway API when you need Gateway resources and more advanced traffic-management patterns.

### Workload Identity

No JSON service-account key is stored in the Pod.

```text
Kubernetes ServiceAccount
        |
        | Workload Identity
        v
Google Service Account
        |
        v
Secret Manager
```

This is preferable to baking credentials into the image or mounting long-lived service-account keys.

### Node affinity + taints/tolerations

The specialized pool is tainted:

```text
workload=specialized:NoSchedule
```

The specialized workload has both:

```yaml
tolerations:
  - key: workload
    value: specialized
    effect: NoSchedule
```

and:

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
```

The toleration allows the Pod onto the tainted node, while node affinity actually directs it toward the intended nodes.

## 8. What happens during failures?

### Pod failure

```text
Pod A crashes
   |
   v
Kubernetes restarts/replaces it
   |
   v
Service continues using healthy replicas
```

### Node failure

With replicas distributed across nodes, Kubernetes reschedules Pods onto healthy capacity when possible.

### Zone failure

With regional GKE and topology spreading, replicas can be distributed across zones so the service is not dependent on a single zone.

### Traffic during startup

Readiness remains false until the application is ready:

```text
New Pod
  |
  v
Startup probe
  |
  v
Readiness = false
  |
  v
No Service traffic
  |
  v
Application ready
  |
  v
Readiness = true
  |
  v
Traffic begins
```

## 9. Interview explanation

A strong interview explanation based on this implementation is:

> I would deploy the Spring Boot service on a regional GKE cluster with multiple replicas distributed across zones using topology spread constraints and Pod anti-affinity. A ClusterIP Service provides stable service discovery, and GCE Ingress provides north-south HTTP routing. I would use startup, readiness, and liveness probes to control startup, traffic eligibility, and recovery from unhealthy containers. CPU and memory requests and limits provide predictable scheduling and resource governance. HPA scales the Pod count based on utilization, while GKE node-pool autoscaling provides additional node capacity when Pods cannot be scheduled. For specialized workloads, I would use node affinity together with node taints and matching tolerations. Secrets would be stored in Secret Manager and accessed through Workload Identity rather than static service-account keys. A PodDisruptionBudget and multi-zone placement improve availability during planned maintenance and infrastructure disruptions.

## 10. Production hardening to add next

For a real production platform, add:

- Cloud Armor/WAF in front of the external load balancer or use Apigee for API-management requirements.
- TLS and managed certificates.
- NetworkPolicy rules restricted to known ingress/egress dependencies.
- Pod Security Admission / organization policy.
- Separate node pools for system and workload classes where appropriate.
- OpenTelemetry + Managed Service for Prometheus / Cloud Monitoring.
- Workload-specific SLOs and alerting.
- Multi-cluster or multi-region DR if the availability target requires protection beyond a regional GKE cluster.
- GitOps with Kustomize/Helm and CI/CD promotion between environments.

## Important note

This example intentionally separates secret metadata creation from secret-value creation. Do not put production secret values directly into `terraform.tfvars`, Kubernetes YAML, Git, or container images.
