# Azure Active Directory Kubernetes OIDC Example

## Overview

The guide will configure Kubernetes by:
* enabling [OpenID Connect](http://openid.net/connect/) Authentication (with Azure Active Directory)
* enabling RBAC Authorization Plugin
* locking it down by default
* enabling a super user (so that we can create role assignments)
* creating a [`ClusterRole`](http://kubernetes.io/docs/admin/authorization/#rbac-mode) (named `cluster-read-only`) which grants read-only (`"get", "list"`) access to all api objects
* creating a [`ClusterRoleBinding`](http://kubernetes.io/docs/admin/authorization/#rbac-mode) granting your AAD user the `cluster-read-only` role

Finally, we will prove that the Azure Active Directory user is authenticated and has limited access by retrieving
an OIDC `id_token` from Active Directory and showing that the user can list pods (but not create them).


### Configure `kube-apiserver`

SSH to your master node and edit `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```
{... other flags ...}
--authorization-mode=AlwaysAllow,RBAC
--authorization-rbac-super-user=client
--oidc-client-id=49c61316-a48b-4e58-81b3-020ab2cab9dc
--oidc-issuer-url=https://login.microsoftonline.com/72f988bf-86f1-41af-91ab-2d7cd011db47/v2.0
--oidc-username-claim=preferred_username
--v=10
```

Ensure you use the correct values:
* `--oidc-client-id`: use your own (create one [here](https://apps.dev.microsoft.com))
* `--oidc-issuer-url`: use your own tenant ID
* `--authorization-rbac-super-user`: a SAN from the client cert you normally use to auth to apiserver

### Create a `ClusterRole` and `ClusterRoleBinding` for your user

1. Edit [`./rbac.yaml`](https://github.com/colemickens/azure-ad-k8s-oidc-example/blob/master/rbac.yaml) to put your `{issuer_url}#{preferred_username}` in as the
subject of the `ClusterRoleBinding`.

2. Apply it: `kubectl apply -f ./rbac.yaml`

### Manually obtain an `id_token` from Azure Active Directory

1. Launch Fiddler
2. [Login to the App with AAD](https://login.microsoftonline.com/common/oauth2/v2.0/authorize?client_id=49c61316-a48b-4e58-81b3-020ab2cab9dc&response_type=id_token&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F&scope=openid%20profile%20email%20offline_access&response_mode=form_post&state=12345&nonce=678910)
3. In Fiddler, retrieve the `id_token` from the POST body. (Look for the red 502.)

### Configure kubectl with the new user

```
kubectl config set-credentials "oidc-user" --token="${AAD_ID_TOKEN}"
```

(We won't modify the current context, just add the new user and then specify
it manually to `kubectl` when we want to use that limited user for testing.)

### Prove it works!

```
$ kubectl --user=oidc-user get pods

NAME                         READY     STATUS    RESTARTS   AGE
nginx-791583134-1dvmc        1/1       Running   0          1d
nginx-791583134-g3ajy        1/1       Running   0          1d
```

```
$ kubectl --user=oidc-user run testpod --image=busybox --restart=Never

Error from server: the server does not allow access to the requested resource (post pods)
```
## Unknowns

* Why are there RSA errors in the `kube-apiserver` logs?
  This is output whenever I make a request with the `oidc-user`:
  ```
  I1030 01:48:58.530111       1 jwt.go:149] Signature error (key 0): crypto/rsa: verification error
  ```
