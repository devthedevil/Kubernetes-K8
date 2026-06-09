# Cloud-Native Programming Language Voting App on Kubernetes

A production-style, multi-tier voting application I built and deployed on **Amazon EKS** to deepen my hands-on experience with Kubernetes primitives — StatefulSets, Services, Secrets, Persistent Volumes, and Replica Sets. Users vote for their favorite programming language out of six choices (C#, Python, JavaScript, Go, Java, Node.js), and votes are persisted to a 3-node MongoDB replica set running inside the cluster.

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.24+-326CE5?logo=kubernetes&logoColor=white)
![AWS EKS](https://img.shields.io/badge/AWS-EKS-FF9900?logo=amazon-aws&logoColor=white)
![Go](https://img.shields.io/badge/Backend-Go-00ADD8?logo=go&logoColor=white)
![React](https://img.shields.io/badge/Frontend-React-61DAFB?logo=react&logoColor=black)
![MongoDB](https://img.shields.io/badge/MongoDB-ReplicaSet-47A248?logo=mongodb&logoColor=white)

---

## Why I Built This

I wanted a realistic, multi-tier workload to practice end-to-end Kubernetes operations — not just `kubectl run nginx`. This project gave me hands-on experience with:

- Designing a **3-tier microservice architecture** (React frontend → Go API → MongoDB) where each tier scales independently.
- Running a **stateful workload** (MongoDB) on Kubernetes using a `StatefulSet`, headless service, and persistent volumes — including bootstrapping a 3-member replica set from inside the cluster.
- Exposing stateless workloads to the internet via **AWS ELB-backed `LoadBalancer` Services**.
- Securing application credentials with Kubernetes **Secrets**.
- Configuring **IAM-for-Service-Accounts** and `aws-auth` on EKS to let an EC2 jump host talk to the API server.

---

## Architecture

```
                       ┌────────────────────────────────────────────────┐
                       │              Amazon EKS Cluster                │
                       │                                                │
   user's browser ───▶ │  ELB ─▶ frontend Deployment (React, 2 replicas)│
                       │           │                                    │
                       │           │  AJAX                              │
                       │           ▼                                    │
                       │  ELB ─▶ api Deployment (Go, 2 replicas)        │
                       │           │                                    │
                       │           │  Mongo wire protocol               │
                       │           ▼                                    │
                       │       headless "mongo" Service                 │
                       │           │                                    │
                       │           ▼                                    │
                       │  StatefulSet: mongo-0, mongo-1, mongo-2        │
                       │       │         │         │                    │
                       │       ▼         ▼         ▼                    │
                       │     PVC       PVC       PVC                    │
                       └────────────────────────────────────────────────┘
```

### Stack

| Tier        | Technology                  | Kubernetes Resource           |
|-------------|-----------------------------|-------------------------------|
| Frontend    | React (JavaScript)          | Deployment + LoadBalancer Svc |
| API         | Go (Golang)                 | Deployment + LoadBalancer Svc |
| Database    | MongoDB (3-node replica set)| StatefulSet + Headless Svc + PVCs |
| Secrets     | Mongo credentials           | Secret                        |
| Isolation   | App boundary                | Namespace (`cloudchamp`)      |

---

## Kubernetes Concepts Exercised

- **Namespace** — isolates app workloads from system pods.
- **Deployment** — declarative rollout and scaling for the stateless React and Go tiers.
- **Service (LoadBalancer)** — provisions AWS ELBs to expose the frontend and API externally.
- **Service (Headless / ClusterIP None)** — stable DNS names (`mongo-0.mongo`, `mongo-1.mongo`, `mongo-2.mongo`) so each Mongo pod has a unique addressable identity.
- **StatefulSet** — ordered pod creation and stable network identities, essential for the MongoDB replica set election.
- **PersistentVolume / PersistentVolumeClaim** — durable storage per Mongo pod that survives rescheduling.
- **Secret** — Mongo credentials injected into the Go API as env vars.

---

## Repo Layout

```
Kubernetes-K8/
├── manifests/
│   ├── mongo-statefulset.yaml      # 3-node Mongo StatefulSet + PVCs
│   ├── mongo-service.yaml          # Headless service for stable Mongo DNS
│   ├── mongo-secret.yaml           # Mongo credentials
│   ├── api-deployment.yaml         # Go API Deployment
│   ├── api-service.yaml            # API LoadBalancer service
│   ├── frontend-deployment.yaml    # React frontend Deployment
│   └── frontend-service.yaml       # Frontend LoadBalancer service
└── README.md
```

---

## Prerequisites

- An **AWS account** with permissions to create EKS clusters
- `kubectl`, `aws` CLI v2, and `eksctl` (or AWS Console) installed locally
- (Optional) An EC2 t2.micro jump host inside the EKS VPC for cluster operations

### IAM policy for the jump host

If you use an EC2 jump host, attach this IAM policy so it can authenticate to the EKS API:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "eks:DescribeCluster",
      "eks:ListClusters",
      "eks:DescribeNodegroup",
      "eks:ListNodegroups",
      "eks:ListUpdates",
      "eks:AccessKubernetesApi"
    ],
    "Resource": "*"
  }]
}
```

---

## Deployment Walkthrough

### 1. Create the EKS cluster

Create an EKS cluster with a managed node group of **2 × t2.medium** instances. You can use `eksctl`, the AWS Console, or Terraform — anything that produces a working `kubeconfig`.

### 2. Install tooling on the jump host (optional)

```bash
# kubectl
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.11/2023-03-17/bin/linux/amd64/kubectl
chmod +x ./kubectl && sudo cp ./kubectl /usr/local/bin
export PATH=/usr/local/bin:$PATH

# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install
```

### 3. Wire up kubeconfig

```bash
aws eks update-kubeconfig --name <EKS_CLUSTER_NAME> --region us-west-2
kubectl get nodes
```

> Hitting `You must be logged in to the server (Unauthorized)`? Update the `aws-auth` ConfigMap so your IAM principal maps to a cluster role — see [AWS docs](https://repost.aws/knowledge-center/eks-api-server-unauthorized-error).

### 4. Clone and create the namespace

```bash
git clone https://github.com/devthedevil/Kubernetes-K8.git
cd Kubernetes-K8/manifests

kubectl create ns cloudchamp
kubectl config set-context --current --namespace cloudchamp
```

### 5. Deploy MongoDB (StatefulSet + headless service)

```bash
kubectl apply -f mongo-statefulset.yaml
kubectl apply -f mongo-service.yaml
```

Confirm each Mongo pod has its own DNS record via the headless service:

```bash
kubectl run --rm utils -it --image=praqma/network-multitool -- bash
# inside the utils pod:
for i in {0..2}; do nslookup mongo-$i.mongo; done
exit
```

### 6. Initialize the replica set

On `mongo-0`, initialize the replica set and add the other two members:

```bash
cat << 'EOF' | kubectl exec -it mongo-0 -- mongo
rs.initiate();
sleep(2000);
rs.add("mongo-1.mongo:27017");
sleep(2000);
rs.add("mongo-2.mongo:27017");
sleep(2000);
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
sleep(5000);
EOF
```

Takes about 10–15 seconds. Verify one PRIMARY and two SECONDARY members:

```bash
kubectl exec -it mongo-0 -- mongo --eval "rs.status()" | grep "PRIMARY\|SECONDARY"
```

### 7. Seed the database

```bash
cat << 'EOF' | kubectl exec -it mongo-0 -- mongo
use langdb;
db.languages.insert({ name: "csharp", codedetail: { usecase: "system, web, server-side", rank: 5, compiled: false, homepage: "https://dotnet.microsoft.com/learn/csharp", download: "https://dotnet.microsoft.com/download/", votes: 0 }});
db.languages.insert({ name: "python", codedetail: { usecase: "system, web, server-side", rank: 3, script: false, homepage: "https://www.python.org/", download: "https://www.python.org/downloads/", votes: 0 }});
db.languages.insert({ name: "javascript", codedetail: { usecase: "web, client-side", rank: 7, script: false, homepage: "https://en.wikipedia.org/wiki/JavaScript", download: "n/a", votes: 0 }});
db.languages.insert({ name: "go", codedetail: { usecase: "system, web, server-side", rank: 12, compiled: true, homepage: "https://golang.org", download: "https://golang.org/dl/", votes: 0 }});
db.languages.insert({ name: "java", codedetail: { usecase: "system, web, server-side", rank: 1, compiled: true, homepage: "https://www.java.com/en/", download: "https://www.java.com/en/download/", votes: 0 }});
db.languages.insert({ name: "nodejs", codedetail: { usecase: "system, web, server-side", rank: 20, script: false, homepage: "https://nodejs.org/en/", download: "https://nodejs.org/en/download/", votes: 0 }});
db.languages.find().pretty();
EOF
```

### 8. Apply the Mongo credentials secret

```bash
kubectl apply -f mongo-secret.yaml
```

### 9. Deploy the Go API

```bash
kubectl apply -f api-deployment.yaml

kubectl expose deploy api \
  --name=api \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8080
```

Wait for the ELB to become routable and test:

```bash
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $API_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl $API_ELB_PUBLIC_FQDN/ok
curl -s $API_ELB_PUBLIC_FQDN/languages | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/go | jq .
```

### 10. Deploy the React frontend

```bash
kubectl apply -f frontend-deployment.yaml

kubectl expose deploy frontend \
  --name=frontend \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8080
```

Grab the public URL:

```bash
FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc frontend -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
echo "App is live at: http://$FRONTEND_ELB_PUBLIC_FQDN"
```

### 11. Vote and verify

1. Open the frontend URL in your browser and cast votes by clicking **+1** next to each language.
2. Confirm votes were persisted back to MongoDB:

```bash
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```

You should see the `votes` counts incrementing on each language document.

---

## What I Took Away

Building this end-to-end gave me practical reps with:

- **StatefulSet ordering guarantees** — and why MongoDB's replica set bootstrap depends on them.
- **Headless services** for pod-addressable DNS, versus standard ClusterIP services.
- **ELB provisioning** via `Service.type=LoadBalancer` and the DNS propagation gotchas around it.
- **Operator-style commands** (`kubectl exec`, `rs.initiate()`, replica reconfig) for bringing up stateful workloads.
- **PVC lifecycle** — what stays, what goes when a pod is rescheduled.

---

## Teardown

When you're done, delete the namespace (this also cleans up the ELBs):

```bash
kubectl delete ns cloudchamp
```

Then delete the EKS cluster to stop billing.

---

## Author

**Dev Kumar** — [GitHub @devthedevil](https://github.com/devthedevil)
