# k8s auth method
Reference: https://docs.akeyless.io/docs/dedicated-k8s-auth-service-accounts

## Create ServiceAccount with permission to access token review API. 
```
$ kubectl apply -f akl_gw_token_reviewer.yaml
serviceaccount/gateway-token-reviewer created
clusterrolebinding.rbac.authorization.k8s.io/role-tokenreview-binding created

```

## Bearer Token Extraction for K8s Server v1.24 or higher using secret
Note: this will be used by the gateway to call TokenReview API
```
$ kubectl apply -f akl_gw_token_reviewer_token.yaml
secret/gateway-token-reviewer-token created

SA_JWT_TOKEN=$(kubectl get secret gateway-token-reviewer-token \
  --output 'go-template={{.data.token | base64decode}}')
```

## Extract K8s Cluster CA Certificate
```
CA_CERT=$(kubectl config view --raw --minify --flatten  \
    --output 'jsonpath={.clusters[].cluster.certificate-authority-data}')
```

## Create k8s auth method 
```
$ akeyless create-auth-method-k8s -n /devops/gcp/gke-k8s-auth --json --profile devopsapi
{
  "access_id": "p-cyzzfc44hst9km",
  "prv_key": "LS0tLh..Lo="
}

PRV_KEY="LS0tLh..Lo="
ACCESS_ID="p-xyz"
```

## Create K8s Gateway Auth Config Using Bearer Tokens
Note: access ID in Akeyless profile must be an admin in the gateway.
```
kubectl proxy --api-prefix=/k8s-api
```
```
$ K8S_ISSUER=$(curl -s http://localhost:8001/k8s-api/.well-known/openid-configuration | jq -r .issuer)
waynezhus-MacBook-Pro:k8s-auth waynezhu$ echo $K8S_ISSUER
https://container.googleapis.com/v1/projects/my-project/locations/us-central1/clusters/waynez-k8s-demo
```

Get k8s cluster IP
Note: This will be used by the gateway to call TokenReview API
```
K8S_Cluster_IP=$(gcloud container clusters describe waynez-k8s-demo --region us-central1 --format="value(endpoint)")
```

```
$ akeyless gateway-create-k8s-auth-config --name /devops/gcp/k8s-conf --gateway-url $AKL_GW_URL --access-id $ACCESS_ID --signing-key $PRV_KEY --k8s-host https://$K8S_Cluster_IP --token-reviewer-jwt $SA_JWT_TOKEN --k8s-ca-cert $CA_CERT --k8s-issuer $K8S_ISSUER --profile devopsapi
K8S Auth config /devops/gcp/k8s-conf successfully created. [ /devops/gcp/k8s-conf]
```
Question: how to get the details on gateway-create-k8s-auth-config in Akeyless CLI


## Verify with a pod
Create a pod
```
kubectl run mypod1 --image=nginx 
kubectl exec --stdin=true  --tty=true mypod1 -- /bin/sh
```
In the pod
```
./akeyless auth --access-id $ACCESS_ID \
    --acc> ess-type k8s \
    --gateway-url  $AKL_GW_URL \
    --k8s-auth-config-name /devops/gcp/k8s-conf
Authentication succeeded.
Token: t-80...d9

# ./akeyless describe-sub-claims
{
  "sub_claims": {
    "client_ip": [
      "1.2.3.4"
    ],
    "client_unique_id": [
      "p-...."
    ],
    "config_id": [
      "r2ehFquWVXqJVBbYaUSDvTE6sdvRp3hV"
    ],
    "config_name": [
      "/devops/gcp/k8s-conf"
    ],
    "k8s_id": [
      "default"
    ],
    "namespace": [
      "default"
    ],
    "pod_name": [
      "mypod1"
    ],
    "pod_uid": [
      "44d0ffa3-889c-447c-baf6-297e178638cc"
    ],
    "service_account_name": [
      "default"
    ],
    "service_account_secret_name": [
      ""
    ],
    "service_account_uid": [
      "54cb5579-205f-4f8d-90ec-525e5b3bf601"
    ],
    "use_pod_id_in_unique_id": [
      "false"
    ]
  }
}
```

## How does the gateway validate the JWT token of service account via TokenReview API
Create a token for a service account (default) to validated
```
TOKEN=$(kubectl create token default -n default)
```

Call TokenReview API with kubectl proxy without authentication
```
$ kubectl proxy
$ curl -X POST http://localhost:8001/apis/authentication.k8s.io/v1/tokenreviews -H "Content-Type: application/json" -d "{
>   \"apiVersion\": \"authentication.k8s.io/v1\",
>   \"kind\": \"TokenReview\",
>   \"spec\": {
>     \"token\": \"$TOKEN\"
>   }
> }"
{
  "kind": "TokenReview",
  "apiVersion": "authentication.k8s.io/v1",
  "metadata": {
    ...
  },
  "spec": {
    "token": "ey...0w"
  },
  "status": {
    "authenticated": true,
    "user": {
      "username": "system:serviceaccount:default:default",
      "uid": "54cb5579-205f-4f8d-90ec-525e5b3bf601",
      "groups": [
        "system:serviceaccounts",
        "system:serviceaccounts:default",
        "system:authenticated"
      ],
      "extra": {
        "authentication.kubernetes.io/credential-id": [
          "JTI=56a374a5-..."
        ]
      }
    },
    ...
  }
}
```

Call TokenReview API with SA_JWT_TOKEN authentication
```
K8S_API_SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')

$ curl -k -X POST $K8S_API_SERVER/apis/authentication.k8s.io/v1/tokenreviews \
   -H "Authorization: Bearer $SA_JWT_TOKEN" \
   -H "Content-Type: application/json" \
   -d "{
     \"apiVersion\": \"authentication.k8s.io/v1\",
     \"kind\": \"TokenReview\",
     \"spec\": {
       \"token\": \"$TOKEN\"
     }
   }"
```