# Goal 

This guide is providing an end-to-end example installation of Kasten on EKS : 
- CSI Storage that support snapshot for crash consistent backup 
- Kasten GUI exposed behind HTTPS with a valid certificate provided by letsencrypt through certmanager.
- Access with kubernetes token authentication (that was the simplest approach but we could improve with an OIDC provider)
- Create Immutable backup on AWS S3 bucket

A terraform could be provided that would do the same thing but the goal of this guide 
is to help the reader understand all the steps and let him hack or deviate consciously from it.

Everything is done through CLI and API so that you can reuse this work easily in your CI/CD pipeline or in your script.

# Prerequisite 

Follow this 2 tutorials in this order 
1. [Create an eks cluster with a CSI Driver that supports snapshot](./pre-req-1-storage.md)
2. [Install an NGINX ingress controller with cert manager](./pre-req-2-network.md)

The first tutorial will create the EKS cluster with the storage stack properly set up. The second tutorial will install the 
nginx ingress controller with cert manager to deliver valid certificates.

The second tutorial will also install the pacman example application that we'll use later to test that we can do valid backup.

## Initial values 

In the [previous tutorial](./pre-req-2-network.md) you initiated those values 
and we'll need them again for the rest of this tutorial.

```
cluster_name=eks-mcourcy
region=eu-west-3
domain="mydomain.com"
account_id=$(aws sts get-caller-identity --query Account --output text)
cluster_name_=$(echo "eks-mcourcy" | tr '-' '_')
role_name="${cluster_name_}_AmazonEKS_EBS_CSI_DriverRole"
subdomain="${cluster_name}.${domain}"
```

Make sure those variable are available for the commands in this tutorial.

# install kasten 

First create the namespace kasten-io 
```
kubectl create ns kasten-io 
```

Create a networkpolicy to allow certmanager and the acme protocol to check the url and deliver the certificate.
If you don't do that the helm installer will create the Kasten network policies that will by default 
deny any access to the temporary ingress that cert manager create to run the ACME protocol. 
```
cat <<EOF |kubectl create -f - 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-lets-encrypt
  namespace: kasten-io
spec:
  ingress:
  - ports:
    - port: 8089
      protocol: TCP
    - port: 30642
      protocol: TCP
  podSelector:
    matchLabels:
      acme.cert-manager.io/http01-solver: "true"
  policyTypes:
  - Ingress
EOF
```

Now we can install kasten the annotation on the ingress section will have certmanager generate a certificate 
with letsencrypt using the ACME protocol.
```
helm repo add kasten https://charts.kasten.io/ 
helm repo update 
cat <<EOF |helm install k10 kasten/k10 -n kasten-io -f -
auth:
  tokenAuth:
    enabled: true
ingress:
  create: true
  host: kasten.${subdomain}  
  urlPath: k10
  annotations: 
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt
  tls:
    enabled: true 
    secretName: kasten-gateway-tls-secret
EOF
```

You should see all the pods in kasten-io up and running 
```
kubectl get pods -n kasten-io 
```

# Access kasten 

We use the token authentication so we'll create a token secret of the k10-k10 service account.

```
cat <<EOF |kubectl create -n kasten-io -f -
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: k10-k10-token
  annotations:
    kubernetes.io/service-account.name: k10-k10
EOF
```

Now you can grab the token with this command 
```
kubectl get secret -n kasten-io k10-k10-token -o jsonpath='{.data.token}' | base64 -d
```

Use the ingress to access the kasten dashboard and use the token to authenticate 
```
echo "https://kasten.${subdomain}/k10/"
```

## Configure the snapshotclass to be used by Kasten 

```
kubectl annotate volumesnapshotclass ebs-snapshotclass k10.kasten.io/is-snapshot-class=true
```

# create a policy for the pacman application 

With the GUI or the API
```
cat <<EOF |kubectl create -f -
apiVersion: config.kio.kasten.io/v1alpha1
kind: Policy
metadata:
  name: pacman-backup
  namespace: kasten-io
spec:
  comment: ""
  frequency: "@onDemand"
  paused: false
  actions:
    - action: backup
  retention: null
  selector:
    matchExpressions:
      - key: k10.kasten.io/appNamespace
        operator: In
        values:
          - pacman
  subFrequency: null
EOF
```

and run it with the GUI or the API
```
cat<<EOF |kubectl create -f -
kind: RunAction
apiVersion: actions.kio.kasten.io/v1alpha1
metadata:
  name: pacman-run
  namespace: kasten-io  
  labels:
    k10.kasten.io/doNotRetire: "true"
    k10.kasten.io/policyName: pacman-backup
    k10.kasten.io/policyNamespace: kasten-io
spec:
  subject:
    apiVersion: config.kio.kasten.io/v1alpha1
    kind: Policy
    name: pacman-backup
    namespace: kasten-io
EOF
```

Check it succeeded with the GUI or the API
```
kubectl get runaction -n kasten-io pacman-run -o jsonpath='{.status.state}' 
```

The value should be `Complete`.


# Add a backup location profile 

So far we did local backup using snapshot and storing the collection of manifests in the internal kasten catalog (backup inventory).
But if you follow the 3-2-1 rules of backup you need an external location. Let's do it by creating a S3 bucket and a unique 
IAM user to access this bucket.

To reinforce the security we're going to use objectLocking on the S3 to create immutable backup so that even a rogue admin won't be able to delete your backups.

## Create the bucket with object lock retention 

Let's create the bucket 
```
# Create the bucket with object lock enabled
aws s3api create-bucket \
    --bucket $cluster_name \
    --region $region \
    --create-bucket-configuration LocationConstraint=$region \
    --object-lock-enabled-for-bucket

# Put object lock configuration
aws s3api put-object-lock-configuration \
    --bucket $cluster_name \
    --object-lock-configuration '{
        "ObjectLockEnabled": "Enabled",
        "Rule": {
            "DefaultRetention": {
                "Mode": "COMPLIANCE",
                "Days": 60
            }
        }
    }'
```

## Create a user and a policy for this bucket 

Let's create the IAM user that will use this bucket 
```
aws iam create-user --user-name ${cluster_name}-bucket
```

Create a policy json file 
```
cat <<EOF > ${cluster_name}-bucket-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetBucketLocation",
                "s3:ListAllMyBuckets"
            ],
            "Resource": "arn:aws:s3:::*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:PutBucketPolicy",
                "s3:ListBucket",
                "s3:DeleteObject",
                "s3:DeleteBucketPolicy",
                "s3:GetBucketLocation",
                "s3:GetBucketPolicy",
                "s3:ListBucketVersions",
                "s3:GetObjectRetention",
                "s3:PutObjectRetention",
                "s3:GetBucketObjectLockConfiguration",
                "s3:GetBucketVersioning",
                "s3:GetObjectVersion",
                "s3:DeleteObjectVersion"

            ],
            "Resource": [
                "arn:aws:s3:::${cluster_name}",
                "arn:aws:s3:::${cluster_name}/*"
            ]
        }
    ]
}
EOF
```

Create the policy 
```
aws iam create-policy --policy-name ${cluster_name}-bucket --policy-document file://${cluster_name}-bucket-policy.json
```

and attach it to the user 
```
aws iam attach-user-policy --user-name ${cluster_name}-bucket --policy-arn arn:aws:iam::${account_id}:policy/${cluster_name}-bucket
```

## Create the secret and the location profile


This will generate an access key a,d a secret key for this user 
and store them in a secret on your cluster. 
```
aws iam create-access-key --user-name ${cluster_name}-bucket| jq '
  {
    "apiVersion": "v1",
    "stringData": {
      "aws_access_key_id": "\(.AccessKey.AccessKeyId)",
      "aws_secret_access_key": "\(.AccessKey.SecretAccessKey)"
    },
    "kind": "Secret",
    "metadata": {
      "name": "\(.AccessKey.UserName)",
      "namespace": "kasten-io",      
    },
    "type": "secrets.kanister.io/aws"
  }
  '| kubectl apply -f - 
```

Now we can create the profile with a protection period of 10 days (240h): 
```
cat <<EOF |kubectl create -f -
apiVersion: config.kio.kasten.io/v1alpha1
kind: Profile
metadata:
  name: ${cluster_name}
  namespace: kasten-io
spec:
  type: Location
  locationSpec:
    credential:
      secretType: AwsAccessKey
      secret:
        apiVersion: v1
        kind: Secret
        name: ${cluster_name}-bucket
        namespace: kasten-io
    type: ObjectStore
    objectStore:
      name: ${cluster_name}
      objectStoreType: S3
      region: $region
      protectionPeriod: 240h
EOF
```

## Udpate the policy to include the profile 


Let's apply this new manifest for the policy
```
cat <<EOF |kubectl apply -f -
apiVersion: config.kio.kasten.io/v1alpha1
kind: Policy
metadata:
  name: pacman-backup
  namespace: kasten-io
spec:
  comment: ""
  frequency: "@onDemand"
  paused: false
  actions:
    - action: backup
    - action: export
      exportParameters:
        frequency: "@onDemand"
        migrationToken:
          name: ""
          namespace: ""
        profile:
          name: ${cluster_name}
          namespace: kasten-io
        receiveString: ""
        exportData:
          enabled: true
  retention: null
  selector:
    matchExpressions:
      - key: k10.kasten.io/appNamespace
        operator: In
        values:
          - pacman
  subFrequency: null
EOF
```

Test it by running another backup through the GUI or with CLI through a runaction 

```
cat<<EOF |kubectl create -f -
kind: RunAction
apiVersion: actions.kio.kasten.io/v1alpha1
metadata:
  name: pacman-run-with-export-to-s3
  namespace: kasten-io  
  labels:
    k10.kasten.io/doNotRetire: "true"
    k10.kasten.io/policyName: pacman-backup
    k10.kasten.io/policyNamespace: kasten-io
spec:
  subject:
    apiVersion: config.kio.kasten.io/v1alpha1
    kind: Policy
    name: pacman-backup
    namespace: kasten-io
EOF
```

# Conclusion 

In this tutorial you installed Kasten to run on EKS and create immutable backup of your applications.
