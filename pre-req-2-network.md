# Goal 

This is the second part of the kasten on eks tutorial. We create an ingress controller and with certmanager 
each https ingress will automatically get a valid certificate so that Kasten GUI could be exposed with a proper
certificate.

## Prerequisite 

- An EKS cluster with a storage stack created from this [tutorial](./pre-req-1-storage.md)
- Access to route 53
- A hosted zone on route 53 for instance mydomain.com

## Initial values 

In the [previous tutorial](https://github.com/michaelcourcy/eks-storage-stack) you initiated those values 
and we'll need them again for the rest of this tutorial

```
cluster_name=eks-mcourcy
region=eu-west-3

account_id=$(aws sts get-caller-identity --query Account --output text)
cluster_name_=$(echo $cluster_name | tr '-' '_')
role_name="${cluster_name_}_AmazonEKS_EBS_CSI_DriverRole"
```

In this tutorial we're going to use a wildcard domain that will be handled by the ingress controller.
We add this extra variable `domain` and `subdomain`.

```
domain="mydomain.com"
subdomain="${cluster_name}.${domain}"
```

We use a logic of one subdomain wildcard DNS record per cluster (or per ingress controller if you prefer). 
This way you don't have to recreate a domain per cluster and pay each times the fee associated to domain registration.

We'll create a wilcard DNS record `*.${subdomain}` for any service we'll create on this cluster. 
For instance kasten will be accessible through `https://kasten.${subdomain}` or pacman example application 
through `https://pacman.${subdomain}`.


# Install the nginx ingress controller

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update 
helm upgrade --install ingress-nginx ingress-nginx \
             --repo https://kubernetes.github.io/ingress-nginx \
             --namespace ingress-nginx \
             --create-namespace \
             --set controller.allowSnippetAnnotations=true
```

Check the install
```
kubectl get po -n ingress-nginx 
```

you should see something like this 
```
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-6d9dfb8fd7-56h6j   1/1     Running   0          93s
```

Find the load balancer created by the install 
```
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

You should get an output like this one 
```
NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP                                                             PORT(S)                      AGE
ingress-nginx-controller   LoadBalancer   10.100.32.241   a6c144kdjk6cb4cb0ac88a2a20c51160-99240850.us-west-2.elb.amazonaws.com   80:30168/TCP,443:31133/TCP   10m
```

Set the external IP in a variable
```
externalIP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

## Change on route 53


In route 53 create a wildcard entry *.${subdomain} in the domain so that it points to the load balancer 

```
hosted_zone_id=$(aws route53 list-hosted-zones | jq --arg domain "${domain}." -r '.HostedZones[] | select(.Name==$domain) | .Id')
wildcard="*.${subdomain}"
aws route53 change-resource-record-sets --hosted-zone-id $hosted_zone_id --change-batch '{
    "Changes": [
        {
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "'$wildcard'",
                "Type": "CNAME",
                "TTL": 300,
                "ResourceRecords": [
                    {
                        "Value": "'$externalIP'"
                    }
                ]
            }
        }
    ]
}'
```

Now test the resolution 
```
dig foo-bar.${subdomain}
```
you should have this answer 
```
;; ANSWER SECTION:
foo-bar.${subdomain}. 60 IN CNAME  a6c144kdjk6cb4cb0ac88a2a20c51160-99240850.us-west-2.elb.amazonaws.com.
a6c144kdjk6cb4cb0ac88a2a20c51160-99240850.us-west-2.elb.amazonaws.com. 60 IN A 35.94.44.40
a6c144kdjk6cb4cb0ac88a2a20c51160-99240850.us-west-2.elb.amazonaws.com. 60 IN A 34.208.8.173
a6c144kdjk6cb4cb0ac88a2a20c51160-99240850.us-west-2.elb.amazonaws.com. 60 IN A 44.243.124.114
```

## install cert-manager 

```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

Check your installation work as expected 
```
kubectl get po -n cert-manager
```

You should get an ouput like 
```
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-6bdd6f4d65-5bn87              1/1     Running   0          84s
cert-manager-cainjector-5665b7b4d4-6n5tz   1/1     Running   0          84s
cert-manager-webhook-7bb4c697b8-5b892      1/1     Running   0          84s
```

Create a Certificate Issuer using lets encrypt replace `<YOUR_EMAIL_ADRESS>` by a valid email address
```
EMAIL=<YOUR_EMAIL_ADDRESS>
cat <<EOF | kubectl create -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: $EMAIL
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

check the status of the issuer 
```
kubectl get clusterissuers.cert-manager.io letsencrypt-prod -o jsonpath='{.status}'|jq
```

You should see accountRegistered in the output with the status ready
```
{
  "acme": {
    "lastPrivateKeyHash": "k8U8gMChoeLmoKAvL3/P/02+S4f5lB5wXO9PDTojsVM=",
    "lastRegisteredEmail": "<YOUR_EMAIL_ADDRESS>",
    "uri": "https://acme-v02.api.letsencrypt.org/acme/acct/1675297167"
  },
  "conditions": [
    {
      "lastTransitionTime": "2024-04-16T09:43:37Z",
      "message": "The ACME account was registered with the ACME server",
      "observedGeneration": 1,
      "reason": "ACMEAccountRegistered",
      "status": "True",
      "type": "Ready"
    }
  ]
}
```

## Test an ingress with TLS 

```
helm repo add pacman https://shuguet.github.io/pacman/
helm install pacman pacman/pacman -n pacman --create-namespace
```

This will create a pacman application check all is working 
```
kubectl get po -n pacman
```

you should see two pods running 
```
NAME                              READY   STATUS    RESTARTS   AGE
pacman-6dcf9dd76d-4j2rr           1/1     Running   0          38s
pacman-mongodb-58ffb5567b-6bx2s   1/1     Running   0          38s
```

Now you can expose the pacman service with an ingress 
```
cat<<EOF |kubectl create -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pacman
  namespace: pacman
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: "nginx"    
spec:
  tls:
  - hosts:
      - pacman.${subdomain}
    secretName: pacman-tls
  rules:
  - host: pacman.${subdomain}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: pacman
            port:
              number: 80
EOF
```

If you immediately run 
```
kubectl get ingress -n pacman
```

You'll see the acme resolver validating that you own the name pacman.${subdomain}
```
NAME                        CLASS    HOSTS                               ADDRESS   PORTS     AGE
cm-acme-http-solver-s4pmf   <none>   pacman.${subdomain}             80        16s
pacman                      <none>   pacman.${subdomain}             80, 443   17s
```

After few seconds cert-manager create the certificate and remove the ingress.
```
kubectl get ingress -n pacman
NAME     CLASS    HOSTS                               ADDRESS   PORTS     AGE
pacman   <none>   pacman.${subdomain}             80, 443   23s

kubectl get secret -n pacman
NAME                           TYPE                 DATA   AGE
pacman-mongodb                 Opaque               1      9m52s
pacman-tls                     kubernetes.io/tls    2      3m31s
sh.helm.release.v1.pacman.v1   helm.sh/release.v1   1      9m52s


kubectl get certificate -n pacman
NAME         READY   SECRET       AGE
pacman-tls   True    pacman-tls   4m19s
```

And you can access the pacman application with https : https://pacman.${subdomain}

# Conclusion

The certmanager + ingress controller part was necessary to now move to a complete and 
secure [install of kasten on the eks cluster](./readme.md).