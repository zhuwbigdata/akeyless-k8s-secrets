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

##
```
$ akeyless create-role --name /devops/k8s/k8s_role --profile devopsapi

$ akeyless assoc-role-am --role-name /devops/k8s/k8s_role --am-name /devops/gcp/gke-k8s-auth --profile devopsapi
Association ass-f5jcio0dtveo84mik8b8 was successfully created

$ akeyless set-role-rule --role-name /devops/k8s/k8s_role --path /devops/k8s/'*' --capability read --capability list --profile devopsapi
The requested rule was successfully set to the role /devops/k8s/k8s_role 
```

## Install Akeyless Injector
```
helm repo add akeyless https://akeylesslabs.github.io/helm-charts
helm repo update
helm show values akeyless/akeyless-secrets-injection > values.yaml
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
waynezhus-MacBook-Pro:gcp waynezhu$
```

## Verify
Change the k8s secret name to what you just created.
```
kubectl apply -f env.yaml
```

Check on ENV at pod level - not injected.

```
kubectl exec --stdin=true --namespace default  --tty=true  test -- /bin/sh
# env | grep MY_SECRET
MY_SECRET=akeyless:/devops/k8s/k8s_secret
$ kubectl logs test-c8c69f54c-wgv2h
Defaulted container "alpine" out of: alpine, akeyless-init (init)
{"aws_access_key":"1234","aws_key_id":"abcd"}
going to sleep...
```
./cli-linux-amd64 auth --access-id p-cyzzfc44hst9km \
    --access-type k8s \
    --gateway-url https://gw-gke.wz.cs.akeyless.fans/api/v1 \
    --k8s-auth-config-name /devops/gcp/k8s-conf

curl -o akeyless https://akeyless-cli.s3.us-east-2.amazonaws.com/cli/latest/production/cli-linux-amd64
chmod +x akeyless