# EMR on EKS — kro ResourceGraphDefinition

## Files

| File | Purpose |
|---|---|
| `rgd-emr-on-eks.yaml` | ResourceGraphDefinition — defines the platform API |
| `ri-emr-on-eks.yaml` | ResourceInstance — what you deploy per team / environment |

---

## Prerequisites

### 1. ACK EMR on EKS controller installed
```bash
# Verify it is running
kubectl get pods -n ack-system | grep emrcontainers
```

### 2. kro controller installed
```bash
kubectl get pods -n kro-system
```

### 3. EMR Job Execution Role

Create an IAM role that EMR on EKS can assume for Spark jobs:

```bash
export ROLE_NAME="EMRonEKS-JobExecutionRole"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export EKS_CLUSTER="my-cluster"
export EMR_NAMESPACE="emr-eks"

# 1. Create the role with a basic trust policy
aws iam create-role \
  --role-name $ROLE_NAME \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "emr-containers.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'

# 2. Attach S3 + CloudWatch permissions
aws iam attach-role-policy \
  --role-name $ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 3. Update trust policy for the specific cluster + namespace
aws emr-containers update-role-trust-policy \
  --cluster-name $EKS_CLUSTER \
  --namespace $EMR_NAMESPACE \
  --role-name $ROLE_NAME
```

Update `ri-emr-on-eks.yaml` → `spec.executionRoleARN` with the ARN:
`arn:aws:iam::<ACCOUNT_ID>:role/EMRonEKS-JobExecutionRole`

---

## Apply Order

```bash
# 1. Apply the RGD — kro registers a new CRD (EMRonEKS)
kubectl apply -f rgd-emr-on-eks.yaml

# 2. Wait for the RGD to be Ready
kubectl get resourcegraphdefinition emr-on-eks.kro.run -w
# Look for: SYNCED=True, STATE=Active

# 3. Apply the instance — kro creates all resources in dependency order:
#    Namespace → Role → RoleBinding → VirtualCluster → JobRun → Deployment → Service
kubectl apply -f ri-emr-on-eks.yaml

# 4. Watch instance status
kubectl get emroneks my-emr-cluster -w

# 5. Check individual resources
kubectl get virtualcluster -A
kubectl get jobrun -A
kubectl get deployment -n emr-eks
kubectl get pods -n emr-eks
```

---

## Resource Creation Order (resolved by kro)

```
emrNamespace
    └── emrClusterRole
    └── emrClusterRoleBinding
    └── virtualCluster
            └── sparkJob
    └── sampleDeployment
            └── sampleService
```

---

## Customisation Tips

| Field | What to change |
|---|---|
| `eksClusterName` | Your EKS cluster name (must already exist) |
| `emrNamespace` | The K8s namespace bound to the EMR virtual cluster |
| `executionRoleARN` | IAM role for Spark job execution |
| `emrReleaseLabel` | EMR version, e.g. `emr-7.1.0-latest` |
| `jobEntryPoint` | S3 path or `local://` path to your PySpark / JAR |
| `sparkSubmitParameters` | Tune executor count, memory, cores |
| `sampleReplicas` | Number of sample-app pods |

---

## Cleanup

```bash
# Delete instance first — kro removes all child resources in reverse order
kubectl delete -f ri-emr-on-eks.yaml

# Then remove the RGD
kubectl delete -f rgd-emr-on-eks.yaml
```

> **Note:** ACK JobRuns cannot be deleted while their VirtualCluster is active.
> kro handles the ordering automatically on instance deletion.
