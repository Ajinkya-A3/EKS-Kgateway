# kgateway on EKS with AWS NLB

End-to-end guide for installing **kgateway** (Envoy-based Kubernetes Gateway API
implementation) on Amazon EKS and exposing it through an **AWS Network Load
Balancer (NLB)** using the AWS Load Balancer Controller.

This repo uses **NLB target type `ip`** — the NLB registers Pod IPs directly,
bypassing NodePort and kube-proxy for lower latency and native client IP
preservation. See [NLB target type](#nlb-target-type-ip-vs-instance) for the
full explanation.

---

## Architecture

```
Internet
   │
   ▼
AWS NLB  (internet-facing, provisioned by AWS Load Balancer Controller)
   │  port 80  — target type: ip
   ▼
kgateway proxy Pod  (Envoy — Pod IP registered directly in NLB target group)
   │  HTTPRoute match
   ▼
httpbin Service  (test app, namespace: httpbin)
   │
   ▼
httpbin Pod
```

With `ip` target type there is **no NodePort hop and no kube-proxy NAT** —
the NLB sends packets straight to the Envoy pod's VPC IP.

---

## Prerequisites

| Tool | Version |
|------|---------|
| `kubectl` | ≥ 1.28 |
| `helm` | ≥ 3.12 |
| `eksctl` | ≥ 0.170 |
| `aws` CLI | v2 |
| EKS cluster | ≥ 1.28 |

Your cluster must use **AWS VPC CNI** (`aws-node`) — the default for EKS
managed node groups and Karpenter node pools. Pods must have routable VPC IPs.

Your EKS subnets must be tagged for the Load Balancer Controller:

```
kubernetes.io/role/elb: "1"          # public/internet-facing subnets
kubernetes.io/role/internal-elb: "1" # private subnets
```

---

## Repository layout

```
.
├── README.md
├── manifests/
│   ├── apps/
│   │   └── httpbin.yaml              # httpbin Namespace, SA, Service, Deployment
│   ├── gateway/
│   │   ├── gateway-parameters.yaml   # GatewayParameters — NLB annotations (ip target type)
│   │   └── gateway.yaml              # Gateway — HTTP listener on port 80
│   └── routes/
│       └── httproute-httpbin.yaml    # HTTPRoute → httpbin
```

---

## Step 1 — Install Kubernetes Gateway API CRDs v1.5.1

kgateway requires the **standard channel** Gateway API CRDs.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml
```

Verify:

```bash
kubectl get crd gateways.gateway.networking.k8s.io
# NAME                                  CREATED AT
# gateways.gateway.networking.k8s.io   2024-...
```

> **Need TCPRoute / UDPRoute / experimental features?** Use the experimental
> channel instead — kgateway 2.2+ enables experimental features by default:
> ```bash
> kubectl apply --server-side -f \
>   https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/experimental-install.yaml
> ```

---

## Step 2 — Install the AWS Load Balancer Controller

The AWS Load Balancer Controller watches Services annotated with
`aws-load-balancer-type: external` and provisions the NLB for you.

```bash
export CLUSTER_NAME="<your-cluster-name>"
export REGION="<your-region>"            # e.g. ap-south-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export IAM_POLICY_NAME=AWSLoadBalancerControllerIAMPolicy
export IAM_SA=aws-load-balancer-controller
```

### 2a — Associate OIDC provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region ${REGION} \
  --cluster ${CLUSTER_NAME} \
  --approve
```

### 2b — Create IAM policy and IRSA service account

```bash
curl -o iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.3/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name ${IAM_POLICY_NAME} \
  --policy-document file://iam-policy.json

eksctl create iamserviceaccount \
  --cluster=${CLUSTER_NAME} \
  --namespace=kube-system \
  --name=${IAM_SA} \
  --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${IAM_POLICY_NAME} \
  --override-existing-serviceaccounts \
  --approve \
  --region ${REGION}
```

### 2c — Install via Helm

```bash
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"

helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=${CLUSTER_NAME} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=${IAM_SA}
```

Verify:

```bash
kubectl -n kube-system get deployment aws-load-balancer-controller
# NAME                           READY   UP-TO-DATE   AVAILABLE
# aws-load-balancer-controller   2/2     2            2
```

---

## Step 3 — Install kgateway

### 3a — Install kgateway CRDs

```bash
helm upgrade -i --create-namespace \
  --namespace kgateway-system \
  --version v2.4.0-main \
  kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds
```

### 3b — Install the kgateway control plane

```bash
helm upgrade -i -n kgateway-system kgateway \
  oci://cr.kgateway.dev/kgateway-dev/charts/kgateway \
  --version v2.4.0-main
```

Verify:

```bash
kubectl get pods -n kgateway-system
# NAME                        READY   STATUS    RESTARTS   AGE
# kgateway-78658959cd-cz6jt   1/1     Running   0          30s

kubectl get gatewayclass kgateway
# NAME       CONTROLLER              ACCEPTED   AGE
# kgateway   kgateway.dev/kgateway   True       1m
```

---

## Step 4 — Deploy the Gateway (NLB-backed, ip target type)

### 4a — Apply GatewayParameters

`GatewayParameters` injects custom annotations onto the Service that kgateway
creates for the Gateway proxy.

```bash
kubectl apply -f manifests/gateway/gateway-parameters.yaml
```

The three annotations and what they do:

| Annotation | Value | Effect |
|---|---|---|
| `aws-load-balancer-type` | `external` | Delegates provisioning to the AWS LB Controller instead of the in-tree cloud provider |
| `aws-load-balancer-scheme` | `internet-facing` | Creates the NLB with public IPs reachable from the internet |
| `aws-load-balancer-nlb-target-type` | `ip` | Registers **Pod IPs** directly in the NLB target group — no NodePort, no kube-proxy |

### 4b — Apply the Gateway

```bash
kubectl apply -f manifests/gateway/gateway.yaml
```

### 4c — Verify the NLB is provisioned

```bash
kubectl get svc -n kgateway-system aws-cloud
```

Wait ~60–90 s for the NLB DNS hostname to appear:

```
NAME        TYPE           CLUSTER-IP      EXTERNAL-IP                                                  PORT(S)        AGE
aws-cloud   LoadBalancer   172.20.39.233   k8s-kgateway-awscloud-xxxx.elb.ap-south-1.amazonaws.com   80:30565/TCP   90s
```

Save the hostname for testing:

```bash
export NLB_HOST=$(kubectl get svc -n kgateway-system aws-cloud \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo $NLB_HOST
```

---

## Step 5 — Deploy the httpbin test app

```bash
kubectl apply -f manifests/apps/httpbin.yaml
```

Verify:

```bash
kubectl -n httpbin get pods
# NAME                       READY   STATUS    RESTARTS   AGE
# httpbin-d57c95548-nz98t   1/1     Running   0          20s
```

---

## Step 6 — Create the HTTPRoute

```bash
kubectl apply -f manifests/routes/httproute-httpbin.yaml
```

> The route is configured for hostname `www.example.com`. Edit
> `manifests/routes/httproute-httpbin.yaml` to set your own domain, or change
> it to `"*"` to match any host during testing.

Verify the route is accepted by the Gateway:

```bash
kubectl get httproute -n httpbin httpbin
# NAME      HOSTNAMES            AGE   ACCEPTED
# httpbin   ["www.example.com"]  30s   True
```

---

## Step 7 — Test end-to-end

```bash
curl -v http://${NLB_HOST}/headers -H "Host: www.example.com"
```

Expected response (`HTTP 200`):

```json
{
  "headers": {
    "Accept": "*/*",
    "Host": "www.example.com",
    "User-Agent": "curl/8.x",
    "X-Forwarded-Proto": "http",
    "X-Real-Ip": "203.0.113.x"
  }
}
```

`X-Real-Ip` shows the **true client IP** — possible because `ip` target type
preserves the source IP all the way from the NLB to the pod.

---

## NLB target type: `ip` vs `instance`

This repo uses `ip`. Here is why and when you would switch.

### How `ip` works

```
Client ──► NLB ──► Pod IP  (direct VPC routing, no NodePort)
```

The NLB target group contains **Pod IPs** registered directly. When a pod is
rescheduled, the LB controller deregisters the old IP and registers the new
one automatically.

**Benefits:**
- No extra kube-proxy hop — lower latency, fewer hops
- True client IP preserved at the pod without any `externalTrafficPolicy` tricks
- Works on AWS Fargate (instance mode does not)
- Cleaner connection tracking — the NLB talks directly to Envoy

**Requirements:**
- **AWS VPC CNI** (`aws-node`) must be the CNI — pods need real VPC-routable IPs
- Overlay CNIs (Flannel, Calico VXLAN, Weave) are **incompatible** — their pod
  IPs are not routable from the NLB
- Subnets must be tagged: `kubernetes.io/role/elb: "1"` (public) or
  `kubernetes.io/role/internal-elb: "1"` (private)
- EKS managed node groups and Karpenter with AWS VPC CNI satisfy this by default

### How `instance` works

```
Client ──► NLB ──► EC2 Node:NodePort ──► kube-proxy ──► Pod
```

The NLB target group contains EC2 **instance IDs**. Traffic enters on a
NodePort and kube-proxy NATs it to a pod (possibly on a different node).

**When to use `instance` instead:**
- You are running a non-AWS CNI (Flannel, Calico VXLAN, Cilium overlay)
- You cannot tag subnets for the LB controller
- You want simpler security group rules (only NodePort range needs opening)

To switch back to `instance`, change one line in `gateway-parameters.yaml`:

```yaml
service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance"
```

### Comparison table

| | `ip` (this repo) | `instance` |
|---|---|---|
| CNI requirement | AWS VPC CNI only | Any CNI |
| kube-proxy hop | None | Yes |
| Client IP | Preserved natively | Needs `externalTrafficPolicy: Local` |
| Fargate support | ✅ | ❌ |
| Subnet tag required | Yes | No |
| Latency | Lower | Higher (one extra hop) |

---

## NLB operational notes

**Idle timeout** — AWS NLB has a **fixed 350-second idle timeout** that cannot
be changed. For long-lived WebSocket or gRPC connections configure TCP
keepalives on both the client and Envoy to stay within this window, or you
will see spurious TCP RSTs.

**Health checks** — With `ip` target type the LB controller health-checks Pod
IPs directly on the container port. Ensure your security groups allow TCP from
the NLB's subnet CIDR to your pod's port (default: 8080 for kgateway's Envoy
admin/health port).

**Cross-zone load balancing** — Disabled by default. Enable to distribute
traffic evenly across AZs:

```yaml
service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
```

**Deregistration delay** — Default is 300 s. For faster rolling deploys you
can reduce it:

```yaml
service.beta.kubernetes.io/aws-load-balancer-target-group-attributes: deregistration_delay.timeout_seconds=30
```

---

## Cleanup

```bash
kubectl delete -f manifests/routes/httproute-httpbin.yaml
kubectl delete -f manifests/apps/httpbin.yaml
kubectl delete -f manifests/gateway/gateway.yaml
kubectl delete -f manifests/gateway/gateway-parameters.yaml

helm uninstall kgateway -n kgateway-system
helm uninstall kgateway-crds -n kgateway-system
kubectl delete namespace kgateway-system

helm uninstall aws-load-balancer-controller -n kube-system

aws iam delete-policy \
  --policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${IAM_POLICY_NAME}
eksctl delete iamserviceaccount \
  --name=${IAM_SA} \
  --cluster=${CLUSTER_NAME}
```

---

## References

- [kgateway docs](https://kgateway.dev/docs/)
- [kgateway AWS NLB guide](https://kgateway.dev/docs/envoy/main/operations/traffic-management/aws-nlb/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [NLB target type: ip mode](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/service/nlb/#ip-mode)
- [Kubernetes Gateway API v1.5.1](https://gateway-api.sigs.k8s.io/)
