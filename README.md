# Using Kubernetes Service Accounts for Google Workload Identity Federation

Or..how to use a plain-old [Kubernetes service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) for authentication to Google Cloud.

With this technique, you can enable an onprem k8s cluster to directly authenticate and access GCP Services without needing to distribute [GCP Service Account Keys](https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform#importing_credentials_as_a_secret) as a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/).  First off, don't do that...dont' download a GCP service account key if you can help it (see [Best practices for securing service accounts](https://cloud.google.com/iam/docs/best-practices-for-securing-service-accounts))

Please note we are talking about two different Service Accounts:

* [GCP Service Accounts](https://cloud.google.com/iam/docs/service-accounts#default)
* [Kubernetes Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)

This article focuses on using the Kubernetes Service account to authenticate to GCP and _optionally_ impersonate a GCP Service Account without needed to download a key.  Its basically federating the Kubernetes Cluster with GCP.

> This repo is NOT supported by Google


If your'e using GKE, don't bother reading this...just use the built in [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity).  If its a raw k8s service on AWS or Azure, see [Configuring workload identity federation](https://cloud.google.com/iam/docs/configuring-workload-identity-federation#aws) and [Anthos clusters](https://cloud.google.com/anthos/clusters/docs)

---

- How does this work?

  We will enable the k8s cluster you run onprem to use [GCP Workload Identity Federation](https://cloud.google.com/iam/docs/configuring-workload-identity-federation#oidc).  Essentially, we will establish trust with GCP that says: "trusts this [openid connect (OIDC)](https://openid.net/connect/)  token issued to a k8s service account by a given K8s API server and map it to this GCP `principal:// or principalSet://`.

- So can i login to kubernetes using an OIDC provider?

  No, thats not what this is about.  This tutorial is not about using an external OIDC provider to login and represent a k8s service.  This article is about the other way:  using the default k8s service identity to access GCP.   If you're interested in the former, you can stop reading and see some of the many tutorials about that

  - [Kubernetes Authentication with OpenID Connect](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)
  - and a tutorial here [Kubernetes RBAC with Google Cloud Identity Platform/Firebase Tokens](https://github.com/salrashid123/kubernetes_oidc_gcp_identity_platform)

- How does this really work?

  We will run a k8s server and specifically ask it to print its OIDC JWK public key.
  
  What that allows us to do is bind the kubernetes OIDC issuer for its service account with GCP and establish trust/federation between them.  We will setup a rule that basically says to GCP: "ok, trust an OIDC JWT token issued and digitally signed by this provider".  When a kubernetes service uses its ambient service account token, it can be used to authenticate to GCP since we setup this trust.

- Whats the catch?

  Well, for one you need to extract the JWK endpoint and upload it to GCP.  If the kuberntes public key every changes (though i don't know why), you would need to update the workload federation too.

  You could also expose the kubernetes server JWK via url also but that would mean GCP will have to find some way to contact the API server.  If you are interested in that flow, see the previous commit in this repo that uses `ngrok` to proxy the api server.


### Setup

First install the following on your laptop

* [minikube](https://minikube.sigs.k8s.io/docs/)
* optionally [jq](https://stedolan.github.io/jq/)


### Start Minikube

Now use that to setup Minikube and enable the `ServiceAccountIssuerDiscovery` feature gate

```bash
export DISCOVERY_URL="https://some-address"
minikube start --driver=kvm2  \
    --extra-config=apiserver.service-account-jwks-uri=$DISCOVERY_URL/openid/v1/jwks \
    --extra-config=apiserver.service-account-issuer=$DISCOVERY_URL
```



Test that you can see the discovery endpoint from the internet though ngrok

```bash
kubectl get --raw /.well-known/openid-configuration  | jq '.'

kubectl get --raw /openid/v1/jwks | jq '.' > jwk.json
```


```bash
$ gcloud organizations list
    DISPLAY_NAME               ID  DIRECTORY_CUSTOMER_ID
    esodemoapp2.com  673208786098              redacted

gcloud resource-manager org-policies allow constraints/iam.workloadIdentityPoolProviders \
   --organization=673208786098 https://some-address 
```


You can optionally view the traffic from the internet through ngrock by looking at the console [http://localhost:4040/](http://localhost:4040/)

### Deploy k8s Service Account 

Deploy a sample applicaton within this cluster.

The following will crate a new serviceAccount and a [Service Account Token Volume Projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection).   

k8s will mount the service account's token at `/var/run/secrets/iot-token` and will include an arbitrary audience we set `gcp-sts-audience`.  This value can be anything but we will use this specific one later when setting up the federation constraints.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: svc1-sa
---
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: svc1-sa
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    type: myapp-deployment-label
spec:
  replicas: 2
  selector:
    matchLabels:
      type: myapp
  template:
    metadata:
      labels:
        type: myapp
        tier: frontend
        version: v1
    spec:
      serviceAccountName: svc1-sa    
      containers:
      - name: frontend
        image: debian
        args:
        - sleep
        - "1000000"
        volumeMounts:
        - mountPath: /var/run/secrets/iot-token
          name: iot-token         
      volumes:
      - name: iot-token
        projected:
          sources:
          - serviceAccountToken:
              path: iot-token
              expirationSeconds: 7200
              audience: gcp-sts-audience
```

Apply the configuration and view the token

```bash
# we will need these later
export PROJECT_ID=`gcloud config get-value core/project`
export PROJECT_NUMBER=`gcloud projects describe $PROJECT_ID --format='value(projectNumber)'`


$ envsubst < "my-deployment.yaml.tmpl" > "my-deployment.yaml"
$ kubectl apply -f my-deployment.yaml
```

Note, the deployment has some extra configmaps and volumes...those aren't used here but i'll explain those later

```bash
$ kubectl get po
NAME                                READY   STATUS    RESTARTS   AGE
myapp-deployment-654945df77-8vw97   1/1     Running   0          24s
myapp-deployment-654945df77-9bz92   1/1     Running   0          24s

$ kubectl exec -ti myapp-deployment-86d84cff8f-ckljb  cat /var/run/secrets/iot-token/iot-token

eyJhbGciOiJSUzI1NiIsImtpZCI6ImZic3ozODQwLVNBTHVtanlrZlE5S3lHSnduNGRuTmc2MmdIejRES3k1MW8ifQ.eyJhdWQiOlsiZ2NwLXN0cy1hdWRpZW5jZSJdLCJleHAiOjE3MzA0MTc1NTUsImlhdCI6MTczMDQxMDM1NSwiaXNzIjoiaHR0cHM6Ly9zb21lLWFkZHJlc3MiLCJrdWJlcm5ldGVzLmlvIjp7Im5hbWVzcGFjZSI6ImRlZmF1bHQiLCJwb2QiOnsibmFtZSI6Im15YXBwLWRlcGxveW1lbnQtNjU0OTQ1ZGY3Ny04dnc5NyIsInVpZCI6ImQ0NTk5MWEzLTdhYWUtNGYwNi05MjY2LTQ4OWYwNzRkMjUxNCJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoic3ZjMS1zYSIsInVpZCI6ImUwNGQzOGRjLTZhZTQtNDc5ZS05MDM4LTNmM2YyNjQ4MmMxMiJ9fSwibmJmIjoxNzMwNDEwMzU1LCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpzdmMxLXNhIn0.sfDdQRc-ewVsXVDDBLEhp1PhMOGW3S5IK49oZeBCxs3rwUqB5ur6-q9BbdgWwppFVNnzS7lQYst8l8N59LAMv9VD4N3-tZ8hUaJbvqXmhyAYHCyfL6q0VAQUuRq1z4jdsYOn551yJMsE5oElHWMflK5rlVdpi_f46G89IQULBVt_MQeHki7bqG2Tj5Fvv39jzMsn61nVw9wezzFyPMrwEvNXeK-7yxV1fhBzBzsmuzj6zZocb4dpqFCb1zLpAo0UezGNo2pkO1PHBK2BhaJvaabOl6Oyfod1DcGtbH3-1EgBnPGeD_uYyA4VVMripx1KKFjD9Uj2OqpWt-1PWWCCaw
```

You can decode that JWT token at [jwt.io](jwt.io).  For the example above

```json
{
  "alg": "RS256",
  "kid": "fbsz3840-SALumjykfQ9KyGJwn4dnNg62gHz4DKy51o"
}
{
  "aud": [
    "gcp-sts-audience"
  ],
  "exp": 1730417555,
  "iat": 1730410355,
  "iss": "https://some-address",
  "kubernetes.io": {
    "namespace": "default",
    "pod": {
      "name": "myapp-deployment-654945df77-8vw97",
      "uid": "d45991a3-7aae-4f06-9266-489f074d2514"
    },
    "serviceaccount": {
      "name": "svc1-sa",
      "uid": "e04d38dc-6ae4-479e-9038-3f3f26482c12"
    }
  },
  "nbf": 1730410355,
  "sub": "system:serviceaccount:default:svc1-sa"
}
```

Note the `issuer`, `aud` and `sub` fields.  We will later configure GCP federation to use these claims to map to an identity

### Configure Workload Identity Federation


  - [Exchange Generic OIDC Credentials for GCP Credentials using GCP STS Service](https://github.com/salrashid123/gcpcompat-oidc)


```bash
export PROJECT_ID=`gcloud config get-value core/project`
export PROJECT_NUMBER=`gcloud projects describe $PROJECT_ID --format='value(projectNumber)'`

gcloud iam workload-identity-pools create pool-k8s-minikube \
    --location="global" \
    --description="k8s OIDC Pool" \
    --display-name="k8s OIDC Pool" --project $PROJECT_ID

# note, we're using a static JWK file, if the oidc server is public, you can specify the external URL (which must be accessible from the kubernetes server)
## see https://github.com/salrashid123/k8s_federation_with_gcp/tree/bc5579b00639c5a79bc7640e849ebf68b2c4c5b7
gcloud iam workload-identity-pools providers create-oidc oidc-provider-k8s-1 \
    --workload-identity-pool=pool-k8s-minikube \
    --issuer-uri="$DISCOVERY_URL" --jwk-json-path=jwk.json \
    --location="global" \
    --attribute-mapping="google.subject=assertion.sub,attribute.aud=assertion.aud[0]" \
    --allowed-audiences="gcp-sts-audience" \
    --project $PROJECT_ID
```


At this moment we are saying, "an OIDC token issued by the provider at `$DISCOVERY_URL` is trusted but as long as its audience is `gcp-sts-audience` and map the sub field to the google `Subject` and also map the audience values."

So, in our case an OIDC token issued by the service we just deployed will be identified as

`"principal://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/oidc-pool-k8s/subject/system:serviceaccount:default:svc1-sa"`

(i'm using just `principal://` here but in reality you'd use groups, etc `principalSet://`)

For now create a gcs bucket and allow this principal access to it.   Only certain services like IAM and GCS support using the `principal://` federated accounts.  If you want to access all other GCP services, you need to use that principal:// and make it exchange its token for another GCP service account that has access (yeah, its a bit confusing)

*Note* that we are giving it full project level admin right..do not do this in reality...i just think our GCS services and cloud console do not allow setting it at the bucket level  (it'm sure its possible, i just don't know how to)

```bash
gsutil mb gs://$PROJECT_ID-test
echo fooooo > foo.txt
gsutil cp foo.txt gs://$PROJECT_ID-test

gcloud projects add-iam-policy-binding $PROJECT_ID  \
 --member "principal://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/pool-k8s-minikube/subject/system:serviceaccount:default:svc1-sa" \
 --role roles/storage.objectAdmin
```

Most GCP services allow you to bind permissions directly with the `principal://` or `principalSet://` federated value.  All this means you can directly access the resource with the binding above.

(optional) However, if you really want to do service account impersonation, create one 

```bash
gcloud iam service-accounts create oidc-federated
gcloud iam service-accounts add-iam-policy-binding oidc-federated@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "principal://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/pool-k8s-minikube/subject/system:serviceaccount:default:svc1-sa"

## add permissions
gcloud projects add-iam-policy-binding $PROJECT_ID  \
 --member "serviceAccount:oidc-federated@$PROJECT_ID.iam.gserviceaccount.com" \
 --role roles/storage.objectAdmin    
```

and in the `sts-creds.json` file below add the stanza

```json
 "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/oidc-federated@core-eso.iam.gserviceaccount.com:generateAccessToken"
```


Anyway, We're now ready to test the federation as-is

```bash

kubectl apply -f my-deployment.yaml
export POD=$(kubectl get pod  -l "type=myapp" -o jsonpath="{.items[0].metadata.name}")
echo $POD
export OIDC_TOKEN=`kubectl exec -ti $POD  cat /var/run/secrets/iot-token/iot-token`
echo $OIDC_TOKEN

curl -s -X POST    -d "grant_type=urn:ietf:params:oauth:grant-type:token-exchange"  \
   -d "audience=//iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/pool-k8s-minikube/providers/oidc-provider-k8s-1" \
   -d "subject_token_type=urn:ietf:params:oauth:token-type:jwt" \
   -d "requested_token_type=urn:ietf:params:oauth:token-type:access_token" \
   -d "scope=https://www.googleapis.com/auth/cloud-platform" \
   -d "subject_token=$OIDC_TOKEN"     https://sts.googleapis.com/v1/token | jq '.'
```

with a little bit of luck you should see

```
{
  "access_token": "ya29.d..",
  "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

Now use that access token to read the GCS bucket

```bash
export ACCESS_TOKEN=....

$ curl -s -H "Authorization: Bearer $ACCESS_TOKEN" https://storage.googleapis.com/storage/v1/b/$PROJECT_ID-test/o/foo.txt
```

If you had GCS data access logs enabled, you'd see:

![images/data_access.png](images/data_access.png)


### Using ADC from the POD

Finally, we just tested this out manually. How would you access this from inside the pod transparently?

First we need to create an [Application Default Credentials file](https://cloud.google.com/iam/docs/configuring-workload-identity-federation#oidc) for OIDC Federation.

Please note that we are specifying the `--credential-source-file`  to use the secret volume mount on the pod `/var/run/secrets/iot-token/iot-token`.  That way, ADC will automatically know where to look.

```bash

gcloud iam workload-identity-pools create-cred-config  \
  projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/pool-k8s-minikube/providers/oidc-provider-k8s-1   \
  --service-account=oidc-federated@$PROJECT_ID.iam.gserviceaccount.com   \
  --output-file=sts-creds.json  \
  --credential-source-file=/var/run/secrets/iot-token/iot-token


```

for me the `sts-cred.json` file looked like this

```json
$ more sts-creds.json 
{
  "type": "external_account",
  "audience": "//iam.googleapis.com/projects/995081019036/locations/global/workloadIdentityPools/pool-k8s-minikube/providers/oidc-provider-k8s-1",
  "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
  "token_url": "https://sts.googleapis.com/v1/token",
  "credential_source": {
    "file": "/var/run/secrets/iot-token/iot-token"
  }
}
```

What we are going to do is mount this file into the deployment as a configmap

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: svc1-sa
---
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: svc1-sa
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    type: myapp-deployment-label
spec:
  replicas: 2
  selector:
    matchLabels:
      type: myapp
  template:
    metadata:
      labels:
        type: myapp
        tier: frontend
        version: v1
    spec:
      serviceAccountName: svc1-sa    
      containers:
      - name: frontend
        image: debian
        args:
        - sleep
        - "1000000"
        env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: "/adc/creds/sts-creds.json"         
        volumeMounts:
        - mountPath: /var/run/secrets/iot-token
          name: iot-token
        - mountPath: /adc/creds
          name: adc-config-vol             
      volumes:
      - name: iot-token
        projected:
          sources:
          - serviceAccountToken:
              path: iot-token
              expirationSeconds: 7200
              audience: gcp-sts-audience
      - name: adc-config-vol
        configMap:
          name: adc-config             
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: adc-config
data:
  sts-creds.json: |
    {
      "type": "external_account",
      "audience": "//iam.googleapis.com/projects/12345678/locations/global/workloadIdentityPools/pool-k8s-minikube/providers/oidc-provider-k8s-1",
      "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
      "token_url": "https://sts.googleapis.com/v1/token",
      "credential_source": {
        "file": "/var/run/secrets/iot-token/iot-token"
      }
    }
```

```bash
kubectl delete -f my-deployment.yaml
kubectl apply -f my-deployment.yaml
export POD=$(kubectl get pod  -l "type=myapp" -o jsonpath="{.items[0].metadata.name}")
echo $POD
```

now verify everything is there:

```bash
$ kubectl exec -ti $POD /bin/bash

root@myapp-deployment-548bb79f55-brddj:/# echo $GOOGLE_APPLICATION_CREDENTIALS
/adc/creds/sts-creds.json

root@myapp-deployment-548bb79f55-brddj:/# cat /adc/creds/sts-creds.json
{
  "type": "external_account",
  "audience": "//iam.googleapis.com/projects/12345678/locations/global/workloadIdentityPools/pool-k8s-minikube/providers/oidc-provider-k8s-1",
  "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
  "token_url": "https://sts.googleapis.com/v1/token",
  "credential_source": {
    "file": "/var/run/secrets/iot-token/iot-token"
  }
}

root@myapp-deployment-548bb79f55-brddj:/# cat /var/run/secrets/iot-token/iot-token
eyJhbGciOiJSUzI1NiIsImtpZCI6IkFUaUdaN2Y2ZTRfMlFtOG5lQWhQeFlEVnlmRkpEQzNTUV9JNFFIdFgzbjgifQ.eyJhdWQiOlsiZ2NwLXN0cy1hdWRpZW5jZSJdLCJleHAiOjE2MzQ5MjE2ODUsImlhdCI6MTYzNDkxNDQ4NSwiaXNzIjoiaHR0cHM6Ly9lNzgyLTcyLTgzLTY3LTE3NC5uZ3Jvay5pbyIsImt1YmVybmV0ZXMuaW8iOnsibmFtZXNwYWNlIjoiZGVmYXVsdCIsInBvZCI6eyJuYW1lIjoibXlhcHAtZGVwbG95bWVudC01NDhiYjc5ZjU1LWJyZGRqIiwidWlkIjoiYmZmYWM5ZGYtZTM2ZS00YTdmLTljZDQtNTUxNmVjZTYzNmQ5In0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJzdmMxLXNhIiwidWlkIjoiYWFkMGJiNzAtYWJlZi00MjBmLTg4NDMtMTM5ZjVkNDBkMWE2In19LCJuYmYiOjE2MzQ5MTQ0ODUsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OnN2YzEtc2EifQ.jkpdNcoNmnNvNSHPox55cmLL6Gma3JkcPsXf1env2y9TUIVVyK0d-A_QDFDO5U5kNuUh21Z0RvjXDqbLriYdUbluP9H0d7nxNSuB0JFw2PpZnncnZ1LGp-2C84-2zv6WYpkinFYR2kS4Q6PkTM4r6_8sEEhT7kp_MSfzchmi--I63IrN6w39XSDLbWQSp7vAv-4exhPNPrpeF6SpSCa5ueRBMdY8uqFLm-1itNRyaozcqiQbZnGZPyahbdmQUoiXSTTcmZTHWo_CKUQ1JS0LbeyutVZSybr5j2rRCO27GYbQ9WnEfaPWdzCid8TX58ptvclijNFE_ZO4KZmEKrU1sAroot@myapp-deployment-548bb79f55-brddj
```

Now install a GCP library set:

```bash
apt-get update
apt-get install virtualenv vim python3-pip

virtualenv env
source env/bin/activate
pip3 install google-cloud-storage 
```

create `main.py` and enter in your `$PROJECT_ID`

```python
from google.cloud import storage
import google.auth

storage_client = storage.Client('$PROJECT_ID')

bucket = storage_client.bucket('$PROJECT_ID-test')
blob = bucket.blob('foo.txt')
out = blob.download_as_string()
print(out.decode('utf-8'))
```

now run it

```bash
root@myapp-deployment-548bb79f55-brddj:/# python3 main.py 
fooooo
```

magic..it used the id token and ADC to auth.

