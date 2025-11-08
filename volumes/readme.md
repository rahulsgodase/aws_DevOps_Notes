# Integrate Amazon EBS with your EKS cluster so pods can dynamically get and mount EBS volumes securely using IAM Roles for Service Accounts (IRSA) — all automated by eksctl.

 # 1 Prerequisites

Make sure you have:

EKS cluster already created.

aws, kubectl, and eksctl installed and configured.

You can run:

kubectl get nodes


and see your nodes.

Note down your:

Cluster name

AWS region

Set them as environment variables for convenience:
```
export CLUSTER=my-eks-cluster
export REGION=ap-south-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```
# 2 Create IAM Policy for EBS CSI Driver

This policy lets the driver create, attach, delete, and describe EBS volumes.

Create file ebs-csi-policy.json:
```
cat > ebs-csi-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:CreateVolume",
        "ec2:DeleteVolume",
        "ec2:DescribeVolumes",
        "ec2:DescribeSnapshots",
        "ec2:DescribeInstances",
        "ec2:ModifyVolume",
        "ec2:CreateTags",
        "ec2:DeleteTags",
        "ec2:DescribeAvailabilityZones"
      ],
      "Resource": "*"
    }
  ]
}
EOF

```
Now create the IAM policy:
```
aws iam create-policy \
  --policy-name AmazonEKS_EBS_CSI_Policy \
  --policy-document file://ebs-csi-policy.json

```
Copy the policy ARN from the output or store it automatically:
```
export POLICY_ARN=arn:aws:iam::${ACCOUNT_ID}:policy/AmazonEKS_EBS_CSI_Policy
```
# 3 Create IAM Role + K8s Service Account (IRSA)

This command does everything for you — creates:

an IAM Role,

links it with EKS’s OIDC provider,

creates a Kubernetes ServiceAccount,

and attaches the IAM policy to it.

Run:
```
eksctl create iamserviceaccount \
  --cluster ${CLUSTER} \
  --region ${REGION} \
  --namespace kube-system \
  --name ebs-csi-controller-sa \
  --attach-policy-arn ${POLICY_ARN} \
  --override-existing-serviceaccounts \
  --approve
```

✅ This creates a ServiceAccount in kube-system namespace with IRSA.

# 4 Install EBS CSI Driver Using Helm

Add and install the Helm chart:
```
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=ebs-csi-controller-sa

```
✅ This installs the EBS CSI controller & node components using the IRSA role you just created.

# 5 Create a StorageClass, PVC, and Test Pod
StorageClass
```
cat > storageclass-ebs.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
reclaimPolicy: Delete
EOF
kubectl apply -f storageclass-ebs.yaml
```
PVC
```
cat > pvc-ebs.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 4Gi
EOF
kubectl apply -f pvc-ebs.yaml
```
Pod
```
cat > pod-test.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ebs-test
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: ebs-vol
      mountPath: /data
  volumes:
  - name: ebs-vol
    persistentVolumeClaim:
      claimName: ebs-pvc
EOF
kubectl apply -f pod-test.yaml

```
# 6 Verify Setup
######## Check CSI driver pods ##########
```
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```
########Check PVC/PV ############
```
kubectl get pvc
kubectl get pv
```
####### See details #########
```
kubectl describe pvc ebs-pvc
kubectl describe pod ebs-test
```
########Test inside pod #######
```
kubectl exec -it ebs-test -- sh
echo "Hello from EBS!" > /data/test.txt
cat /data/test.txt
exit

```
In AWS Console → EC2 → Volumes, you’ll see a new EBS volume tagged with your PVC name.

# 7 Cleanup (optional)
```
kubectl delete -f pod-test.yaml
kubectl delete -f pvc-ebs.yaml
kubectl delete -f storageclass-ebs.yaml

helm uninstall aws-ebs-csi-driver -n kube-system

eksctl delete iamserviceaccount \
  --cluster ${CLUSTER} \
  --namespace kube-system \
  --name ebs-csi-controller-sa
```
# 🧩 8️⃣ How It Works (Internally)

The EBS CSI controller (running in kube-system) creates and manages EBS volumes via AWS API.

It authenticates securely using the IAM Role via IRSA.

When a PVC is created, the driver provisions an EBS volume in the same AZ as the node where the pod runs (WaitForFirstConsumer ensures that).

The EBS volume is attached automatically to that node.

The pod mounts it under /data.
