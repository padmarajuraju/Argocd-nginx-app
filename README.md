📘 END-TO-END DOCUMENTATION: Multi-VPC ArgoCD Cluster Registration Using VPC Peering + K3d Kubernetes Cluster
🟦 1. Overview

This document explains how to connect an ArgoCD instance running in VPC-A to a Kubernetes (k3d) cluster running in VPC-B, using:

AWS VPC Peering

Private IP communication only

ServiceAccount-based kubeconfig

Admin kubeconfig (optional alternative method)

Secure multi-cluster GitOps

The goal is to register the external Kubernetes cluster into ArgoCD without public IPs, while allowing secure API server access across different AWS VPCs.

🟦 2. Architecture Diagram
                        AWS Cloud
────────────────────────────────────────────────────────────────────
                 VPC-A (ArgoCD VPC)        ⇄        VPC-B (Cluster3 VPC)
             CIDR: 172.31.0.0/16                     CIDR: 10.0.0.0/16
────────────────────────────────────────────────────────────────────
   EC2: argocd-server                          EC2: cluster3-server
   Private IP: 172.31.31.148                   Private IP: 10.0.1.86
   ArgoCD UI + CLI                             k3d Kubernetes cluster
────────────────────────────────────────────────────────────────────
                     VPC Peering + Routes (Private Only)
                           No Public Exposure Needed
────────────────────────────────────────────────────────────────────

🟦 3. Prerequisites
EC2 1: ArgoCD Server

OS: Ubuntu 22.04

Security Groups: Allow outbound to VPC-B

ArgoCD installed via manifests

NodePort exposed on 30080

EC2 2: Cluster3 Kubernetes Node

OS: Ubuntu 22.04

k3d / k3s installed

Kubernetes API listening on 42477

Security Groups: Allow inbound 42477 from VPC-A

🟦 4. Create k3d Cluster on EC2 in VPC-B

Login to your cluster3-server EC2:

k3d cluster create cluster3 \
  --api-port 42477 \
  --servers 1 \
  --agents 1


Verify:

kubectl cluster-info


Output:

Kubernetes control plane is running at https://0.0.0.0:42477

Fix server address to private IP

Find private IP:

hostname -I
# Example: 10.0.1.86

🟦 5. Configure Kubernetes API to Use Private IP

Replace 0.0.0.0 with private IP in kubeconfig:

kubectl config view --raw > cluster3-admin-kubeconfig
nano cluster3-admin-kubeconfig


Update:

server: https://10.0.1.86:42477

🟦 6. Create VPC Peering Between VPC-A and VPC-B
6.1 Create Peering Connection

AWS Console → VPC → Peering Connections → Create Peering Connection

Requester: ArgoCD VPC (172.31.0.0/16)

Accepter: Cluster3 VPC (10.0.0.0/16)

Click Create, then Accept Request.

🟦 7. Update Route Tables

VPC Peering does nothing unless route tables are updated.

7.1 ArgoCD VPC Route Table (VPC-A)

Add route:

Destination	Target
10.0.0.0/16	pcx-xxxx

This lets ArgoCD reach cluster3 API.

7.2 Cluster3 VPC Route Table (VPC-B)

Add route:

Destination	Target
172.31.0.0/16	pcx-xxxx

This lets cluster3 respond back to ArgoCD.

🟦 8. Security Group Changes

On cluster3-server EC2 security group, add inbound:

Type	Port	Source
Custom TCP	42477	172.31.31.148/32 (ArgoCD EC2 private IP)

This allows private API access directly.

🟦 9. Validate Inter-VPC Connectivity (Critical Step)

From ArgoCD EC2, run:

curl -k https://10.0.1.86:42477


Expected output:

{
  "status": "Failure",
  "reason": "Unauthorized"
}


This means:

✔ Routing works
✔ SG works
✔ Peering works
✔ Kubernetes API reachable

🟦 10. Method 1 — Add Cluster Using ServiceAccount (Enterprise Recommended)
10.1 Create SA + RBAC inside cluster3

On cluster3 EC2:

kubectl create namespace argocd-sa

kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argocd-manager
  namespace: argocd-sa
EOF


Create ClusterRoleBinding:

kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-manager-binding
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: argocd-manager
  namespace: argocd-sa
EOF

10.2 Extract token
SECRET=$(kubectl -n argocd-sa get secret | grep argocd-manager | awk '{print $1}')
TOKEN=$(kubectl -n argocd-sa get secret $SECRET -o jsonpath="{.data.token}" | base64 -d)
CA=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')

10.3 Build kubeconfig

Create:

apiVersion: v1
kind: Config
clusters:
- name: cluster3
  cluster:
    server: https://10.0.1.86:42477
    certificate-authority-data: <CA_DATA>
users:
- name: argocd-manager
  user:
    token: <TOKEN>
contexts:
- name: cluster3-context
  context:
    user: argocd-manager
    cluster: cluster3
current-context: cluster3-context


Copy this file to ArgoCD EC2.

🟦 11. Register Cluster Using ServiceAccount Kubeconfig

On ArgoCD EC2:

argocd cluster add cluster3-context --kubeconfig ./cluster3-sa-kubeconfig


If ArgoCD tries to create a ServiceAccount and fails:

Use safe registration:

argocd cluster set cluster3 --kubeconfig ./cluster3-sa-kubeconfig


Now verify:

argocd cluster list


Expected:

https://10.0.1.86:42477     cluster3     Successful

🟦 12. Method 2 — Add Cluster Using Admin Kubeconfig (Easy Mode)

If you prefer simpler registration (not secure):

On ArgoCD EC2:

argocd cluster add k3d-cluster3 --kubeconfig ./cluster3-admin-kubeconfig


ArgoCD will automatically:

✔ Create its own ServiceAccount
✔ Bind RBAC
✔ Add the cluster

🟦 13. Validate in ArgoCD UI

Open ArgoCD UI:

http://<ARGOCD_PUBLIC_IP>:30080


Go to:

Settings → Clusters

Cluster3 (private IP) should appear.

🟦 14. Deploy Nginx via ArgoCD (Test)

Create an app:

Repository: GitHub repo with nginx manifest

Destination Cluster: cluster3

Namespace: default

Sync Policy: Manual or Auto

Once synced, check:

kubectl get pods -o wide
kubectl get svc -o wide
