# Akeyless k8s secret injector
Reference: https://docs.akeyless.io/docs/how-to-provision-secret-to-your-k8s 

## Prerequistie
k8s auth method: /devops/gcp/gke-k8s-auth (access ID)
k8s auth config: /devops/gcp/k8s-conf
Token Reviewer SA pre-installed 

## Create secret for K8s pod acccess
```
$ akeyless create-secret --name /devops/k8s/k8s_secret --value '{"aws_access_key":"1234","aws_k
ey_id":"abcd"}' --profile devopsapi
A new secret named /devops/k8s/k8s_secret was successfully created
```

## Create access role, can use subclaim on namespace for difference roles with --sub-claims
```
$ akeyless create-role --name /devops/k8s/k8s_role --profile devopsapi
$ akeyless assoc-role-am --role-name /devops/k8s/k8s_role --am-name /devops/gcp/gke-k8s-auth --profile devopsapi
Association ass-f5jcio0dtveo84mik8b8 was successfully created


$ akeyless set-role-rule --role-name /devops/k8s/k8s_role --path /devops/k8s/'*' --capability read --capability list --profile devopsapi
The requested rule was successfully set to the role /devops/k8s/k8s_role 

# In case to get asscoation ID
$ akeyless get-auth-method -n /devops/gcp/gke-k8s-auth --profile devopsapi -h
$ akeyless update-assoc --assoc-id ass-f5jcio0dtveo84mik8b8 --sub-claims namespace=waynez --profile devopsapi
Association ass-f5jcio0dtveo84mik8b8 was successfully updated
```

## Install Akeyless Injector
```
helm repo add akeyless https://akeylesslabs.github.io/helm-charts
helm repo update
helm show values akeyless/akeyless-secrets-injection > values.yaml

AKEYLESS_ACCESS_ID: p-xyz
AKEYLESS_ACCESS_TYPE: k8s
AKEYLESS_API_GW_URL: https://<AKL_GW_URL>/api/v1
AKEYLESS_K8S_AUTH_CONF_NAME: /devops/gcp/k8s-conf
AKEYLESS_URL: https://vault.akeyless.io
```

## Create Akeyless Secret Injector 

Note: AKEYLESS_API_GW_URL with the URL of your Gateway API v1 endpoint: /api/v1
```
$ kubectl create namespace akeyless
$ kubectl label namespace akeyless name=akeyless
$ helm upgrade --install injector akeyless/akeyless-secrets-injection --namespace akeyless -f values.yaml

$ kubectl get all  -n akeyless
NAME                                                       READY   STATUS    RESTARTS   AGE
pod/injector-akeyless-secrets-injection-596b895749-8d4dv   1/1     Running   0          65s
pod/injector-akeyless-secrets-injection-596b895749-ngl42   1/1     Running   0          65s

NAME                                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/injector-akeyless-secrets-injection   ClusterIP   34.118.226.113   <none>        443/TCP   66s

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/injector-akeyless-secrets-injection   2/2     2            2           66s

NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/injector-akeyless-secrets-injection-596b895749   2         2         2       66s
```

## Secret as ENV
Change the k8s secret name to what you just created.
```
kubectl apply -f env.yaml -n waynez
```

Check on ENV at pod level - not injected.
```
kubectl exec --stdin=true --namespace default  --tty=true  test-<pod-name> -- /bin/sh
# env | grep MY_SECRET
MY_SECRET=akeyless:/devops/k8s/k8s_secret
```
Check on logs 
```
$ kubectl logs test-<pod-name>
Defaulted container "alpine" out of: alpine, akeyless-init (init)
{"aws_access_key":"1234","aws_key_id":"abcd"}
going to sleep...
```

## Secret as File
Change the k8s secret name to what you just created.
```
kubectl apply -f env.yaml sidecar.yaml -n waynez
```
Check on ENV at pod level - not injected.
```
kubectl exec --stdin=true --namespace waynez  --tty=true  test-file-<pod-name> -- /bin/sh
# $ cat /secrets/secretsVersion.json
[
 {
  "version": 1,
  "secret_value": "{\"aws_access_key\":\"1234\",\"aws_key_id\":\"abcd\"}",
  "creation_date": 1742388385
 }
]
```
Check on logs 
```
kubectl logs test-file-<pod-name> -n waynez
Defaulted container "akeyless-sidecar" out of: akeyless-sidecar, alpine, akeyless-init (init)
2025/04/28 18:13:42 [INFO] Secret /devops/k8s/k8s_secret was successfully written to: /secrets/secretsVersion.json
going to sleep...
```