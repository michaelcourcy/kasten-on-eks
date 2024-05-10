# Goal 

This is the first part of the kasten on eks tutorial. We create the cluster and 
set up a csi driver with snapshot capabilities.


## Doc/Credits 
- https://eksctl.io/usage/creating-and-managing-clusters/
- https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html
- https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html
- https://docs.aws.amazon.com/eks/latest/userguide/csi-iam-role.html
- https://adil.medium.com/how-to-create-ebs-volume-snapshots-on-eks-e827b12b85c3


## Prerequisite 

Create this global variable
```
cluster_name=eks-mcourcy
region=eu-west-3

account_id=$(aws sts get-caller-identity --query Account --output text)
cluster_name_=$(echo $cluster_name | tr '-' '_')
role_name="${cluster_name_}_AmazonEKS_EBS_CSI_DriverRole"
```
## Create the cluster 

```
cat<<EOF > cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${cluster_name}
  region: ${region}
nodeGroups:
  - name: ng-1
    instanceType: m5.large
    desiredCapacity: 3
    volumeSize: 80
    ssh:
      allow: true 
  - name: ng-2
    instanceType: m5.xlarge
    desiredCapacity: 3
    volumeSize: 100
    ssh:
      allow: true
EOF
eksctl create cluster -f cluster.yaml
```

### Create an IAM OIDC provider for your cluster

It will allow kubernetes service account to authentify to IAM and assume roles to do the CSI operations.
```
# check you have an oidc issuer for this cluster 
oidc_id=$(aws eks describe-cluster --region $region --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4

# if ther is no output on the previous command then 
eksctl utils associate-iam-oidc-provider --region $region --cluster $cluster_name --approve
```

### Create the Amazon EBS CSI driver IAM role

```
eksctl create iamserviceaccount \
    --region $region \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster $cluster_name \
    --role-name $role_name \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```

### Add the ebs csi driver addon 

```
eksctl create addon --name aws-ebs-csi-driver \
    --region $region \
    --cluster $cluster_name \
    --service-account-role-arn "arn:aws:iam::${account_id}:role/${role_name}" \
    --force
```

The addon will execute with the service account `ebs-csi-controller-sa` which will assume the  IAM `${role_name}` role.

check the status 
```
eksctl get addon --name aws-ebs-csi-driver --cluster $cluster_name --region $region
```

## Test the csi driver  

connect to the cluster 
```
aws eks --region $region update-kubeconfig --name $cluster_name
```

check the csi pods are eplpyed in the kube-system namespace 
```
kubectl get pods -n kube-system
```

Deploy the csi storage class 
```
git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
cd aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/
kubectl apply -f manifests/
kubectl describe storageclass ebs-sc
```

check pvc is created 
```
kubectl get pvc
```


Make ebs-sc the default storage class
```
kubectl annotate storageclass gp2 storageclass.kubernetes.io/is-default-class=false --overwrite
kubectl annotate storageclass ebs-sc storageclass.kubernetes.io/is-default-class=true
```

## Deploy the snapshot controller

```
cd ../../../..
```

check the last version of the https://github.com/kubernetes-csi/external-snapshotter project.

In our case it's v7.0.2.

```
version=v7.0.2
```

Install the CRD : 
```
kubectl apply -k "github.com/kubernetes-csi/external-snapshotter/client/config/crd?ref=$version"
```

Install the snapshot controller 
```
kubectl apply -k "github.com/kubernetes-csi/external-snapshotter/deploy/kubernetes/snapshot-controller?ref=$version"
```

create a volumesnapshotclass 
```
cat <<EOF | kubectl create -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshotclass
driver: ebs.csi.aws.com
deletionPolicy: Delete
EOF
```

check you can create a volumesnapshot of the pvc we created for the test 

```
cat <<EOF |kubectl create -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: first-snapshot
spec:
  volumeSnapshotClassName: ebs-snapshotclass
  source:
    persistentVolumeClaimName: ebs-claim
EOF
```

Check the successful creation of the volumesnapshot 
```
kubectl get volumesnapshot
```

You should see READYTOUSE at true.
```
NAME             READYTOUSE   SOURCEPVC   SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS       SNAPSHOTCONTENT                                    CREATIONTIME   AGE
first-snapshot   true         ebs-claim                           4Gi           ebs-snapshotclass   snapcontent-8c4048e5-c0de-4660-8291-3dcb807165a8   60s            61s
```

# Conclusion 

You are now ready to go with a csi storage on EKS that support snapshot. 

Delete this resources only useful for the test. 
```
kubectl delete pod app
kubectl delete pvc ebs-claim
kubectl delete volumesnapshot first-snapshot
```





