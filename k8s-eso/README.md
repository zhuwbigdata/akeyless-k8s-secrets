# ESO
Reference: 

https://external-secrets.io/latest

https://tutorials.akeyless.io/docs/sync-secrets-to-k8s-with-external-secrets-operator



### Prerequisite for ESO operator
1. Akeyless k8s auth method on the k8s cluster (accessID)
2. Akeyless k8s conf name 
3. Akeyless access role associated with k8s auth method (role name) - can use namespace on subclaim
4. Akeyless secrets to be accessed by the access role (secret name)

### ESO resources
1. ESO deployment - synchronizes secrets from Akeyless into k8s 
1. SecretStore - defines how to access the secrets, namespace based.
2. ClusterSecretStore - defines how to access the secrets, cluster based.
3. ExternalSecret - specifies what to fetech per SecretStore



## Install ESO operator in its own namespace
```
$ helm repo add external-secrets https://charts.external-secrets.io
"external-secrets" has been added to your repositories

$ helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
NAME: external-secrets
LAST DEPLOYED: Fri Mar 21 09:56:24 2025
NAMESPACE: external-secrets
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
external-secrets has been deployed successfully in namespace external-secrets!

$ kubectl get all -n external-secrets
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/external-secrets-76c775559b-jgtsj                   1/1     Running   0          19m
pod/external-secrets-cert-controller-6fbfc58b8c-mjnhh   1/1     Running   0          19m
pod/external-secrets-webhook-5c7c967dfc-bqrtz           1/1     Running   0          19m

NAME                               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/external-secrets-webhook   ClusterIP   1.2.3.4   <none>        443/TCP   19m

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/external-secrets                   1/1     1            1           19m
deployment.apps/external-secrets-cert-controller   1/1     1            1           19m
deployment.apps/external-secrets-webhook           1/1     1            1           19m

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/external-secrets-76c775559b                   1         1         1       19m
replicaset.apps/external-secrets-cert-controller-6fbfc58b8c   1         1         1       19m
replicaset.apps/external-secrets-webhook-5c7c967dfc           1         1         1       19m
waynezhus-MacBook-Pro:k8s-eso waynezhu$ 

```

## Option 1 using secret store in a cluster

## Option 2 using secret store in a specific namespace
### Create a name space and a service account 
```
$ kubectl create ns waynez
namespace/waynez created
$ kubectl create sa devopsa -n waynez
serviceaccount/devopsa created
```

### Create SecretStore in the namespace
Notes: make sure that akeylessGWApiURL pointing to the gateway URL/api/v2 or SaaS https://api.akeyless.io.
```
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: akeyless-secret-store
spec:
  provider:
    akeyless:
      # URL of your akeyless API
      akeylessGWApiURL: "https://<AKL-GW-URL>"
      authSecretRef:
        kubernetesAuth:
          accessID: "p-xyz"
          k8sConfName: "/devops/gcp/k8s-conf" # Found in your gateway
          # Optional service account field containing the name
          # of a kubernetes ServiceAccount
          serviceAccountRef:
            name: "devopsa"
```

```
$ kubectl apply -f K8sSecretStore.yaml -n waynez
secretstore.external-secrets.io/akeyless-secret-store created

$ kubectl describe  SecretStore akeyless-secret-store  -n waynez
```

## Verify Akeyless access role with subclaim on the namespace
```
$ akeyless list-roles --filter "devops/k8s/k8s_eso_role" --profile devopsapi
{

      "role_name": "devops/k8s/k8s_eso_role",
      ...
      "comment": "eso ",
      "role_auth_methods_assoc": [
        {
          ...
          "auth_method_name": "devops/gcp/gke-k8s-auth",
          "auth_method_access_id": "p-xyz",
          "auth_method_sub_claims": {
            "namespace": [
              "waynez"
            ]
          },
        }
      ]
}

```

## Create external secret in the name space
```
$ kubectl apply -f externalsecret.yaml -n waynez
externalsecret.external-secrets.io/akeyless-external-secret-example created

$ kubectl get secret -n waynez
NAME                        TYPE     DATA   AGE
akeyless-secret-to-create   Opaque   1      17m

$ kubectl get secret akeyless-secret-to-create -n waynez -o jsonpath='{.data.secretKey}' | base64 -d
ESO controlled secret

```

### Debug
```
$ kubectl get secret akeyless-secret-to-create -n waynez -o jsonpath='{.data.secretKey}' | base64 -d
Error from server (NotFound): secrets "akeyless-secret-to-create" not found

$ kubectl describe externalsecret akeyless-external-secret-example -n waynez
Name:         akeyless-external-secret-example
Namespace:    waynez
Labels:       <none>
Annotations:  <none>
API Version:  external-secrets.io/v1beta1
Kind:         ExternalSecret
Metadata:
  ...
Spec:
  Data:
    Remote Ref:
      Conversion Strategy:  Default
      Decoding Strategy:    None
      Key:                  /devops/eso/k8s-eso-secret
      Metadata Policy:      None
    Secret Key:             secretKey
  Refresh Interval:         1h
  Secret Store Ref:
    Kind:  SecretStore
    Name:  akeyless-secret-store
  ...
Status:
  Binding:
    Name:  
  Conditions:
    Last Transition Time:  2025-03-21T15:21:42Z
    Message:               could not get secret data from provider
    Reason:                SecretSyncedError
    Status:                False
    Type:                  Ready
  Refresh Time:            <nil>
Events:
  Type     Reason        Age                  From              Message
  ----     ------        ----                 ----              -------
  Warning  UpdateFailed  6m8s (x10 over 14m)  external-secrets  error processing spec.data[0] (key: /devops/eso/k8s-eso-secret), err: item does not exist
```





API key based GW
```
$ helm install external-secrets external-secrets/external-secrets -n eso  --create-namespace
NAME: external-secrets
LAST DEPLOYED: Fri May  2 14:52:02 2025
NAMESPACE: eso
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
external-secrets has been deployed successfully in namespace eso!

In order to begin using ExternalSecrets, you will need to set up a SecretStore
or ClusterSecretStore resource (for example, by creating a 'vault' SecretStore).
More information on the different types of SecretStores and how to configure them
can be found in our Github: https://github.com/external-secrets/external-secrets
```


