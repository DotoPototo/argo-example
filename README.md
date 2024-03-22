# ArgoCD Boilerplate

Self-managed ArgoCD example to get ArgoCD to monitor and update itself and other applications.

## Setting up ArgoCD

### Prerequisites

- A server with kubernetes installed (for this example, [Microk8s](https://microk8s.io/) with dns, ingress and hostpath-storage addons enabled)
- Some helpful aliases (in this case for Microk8s)
  - kubectl="microk8s.kubectl"
  - helm="microk8s.helm3"
  - mk8s="microk8s"

### Install App Required Packages

First, if helm isn't installed then install it following the latest [online instructions](https://helm.sh/)

If the kubernetes instance doesn't have an ingress controller then install the [NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/) using helm.

Finally install [sealed secrets](https://github.com/bitnami-labs/sealed-secrets#installation) via helm, following the suggested overrides for namespace and controller name.

Additionally, install the accompanying [CLI tool Kubeseal](https://github.com/bitnami-labs/sealed-secrets#kubeseal) to encrypt secrets on the cluster. If you plan to pull the public key from the cluster and encrypt secrets locally then you should also install `kubeseal` on your local machine.

Some kubernetes instances supply their own kubeconfig (i.e. Microk8s) so you may have to update/create the host kubeconfig with a copy so `kubeseal` can access it for sealed secrets. An example of this is [here](https://microk8s.io/docs/working-with-kubectl) for Microk8s.

To pull the public key to encrypt secrets locally, output the secret and then reference it when running future `kubeseal` commands

```bash
# On the cluster
kubeseal --fetch-cert > mycert.pem

# Locally
kubeseal --cert mycert.pem
```

Public keys renew every 30 days on the cluster. **You should routinely fetch the latest key**. For more details and explentation see [here](https://github.com/bitnami-labs/sealed-secrets#common-misconceptions-about-key-renewal)

### Install ArgoCD and Cert-Manager

Clone the repo onto the server (can set up an access token in GitLab and remove it afterwards)

```bash
git clone ...
cd ...
```

Install the initial ArgoCD instance via helm into an `argocd` namespace

```bash
helm repo add argo-cd https://argoproj.github.io/argo-helm
helm dep update charts/argocd/
kubectl create namespace argocd
helm install -n argocd argocd charts/argocd
```

Retrieve the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

In a new terminal port forward localhost when connecting with SSH to the k8s server

```bash
ssh xxx -L 8080:127.0.0.1:8080
```

Port forward the ArgoCD server service

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Visit 127.0.0.01:8080 locally and log in with admin and the retrieved password. Change the password.

Next install [cert-manager using helm](https://cert-manager.io/docs/installation/helm/)

```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
```

Applications can then take advantage of the [ClusterIssuer](https://microk8s.io/docs/addon-cert-manager) `apps/templates/cert-manager.yaml` for certificate management.

To finish, running `kubectl get all -A` should show the ouput of argo, cert-manager and others and they should all have a status of `running`.

### Setup the App of Apps

For ease of managing several apps in one cluster, we'll setup Argo to use the [App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern) pattern

First apply the root application, to watch for helm charts under `apps/` and synchronise any changes.

```bash
kubectl apply -n argocd -f apps/templates/root.yaml
```

At this point any applications (and other kubernetes configs i.e. Projects, ClusterIssuers etc) declared in `apps/templates/` should show in ArgoCD.

Then, inform helm that it no longer manages ArgoCD, as ArgoCD will be managing itself

```bash
kubectl delete secret -n argocd -l owner=helm,name=argocd
```

From here you can stop port forwarding argocd and accessing it on localhost and instead access it from the url defined in its values file `charts/argocd/values.yaml`.

You should additionally finish setting up the user accounts ([see below](#user-accounts)).

Finally, clean up the pulled repo you started with

```
rm -r repo_folder-name
```

## Adding Repo Connections

Any connections Argo makes should be defined ArgoCDs values file `charts/values.yaml` under `argo-cd.configs.repositories`

If an application Argo is managing has a private repository then an access token will also have to be defined under` argo-cd.configs.credentialTemplates.https-creds`

## Updating ArgoCD

To update Argo, update the version number in Argos Chart file `charts/Chart.yaml`. Available ArgoCD helm versions can be found on their [GitHub page](https://github.com/argoproj/argo-helm/releases). Commit the update Chart file to the repository and Argo will see the change and update itself accordingly.

## User Accounts

To create an ArgoCD user account, add the new user to the ArgoCD values file `charts/values.yaml` under `argo-cd.configs.cm` with the `login` parameter;

`accounts.ACCOUNT_NAME: login`

Then, under the RBAC policy rules in the same values file, add the user to any roles they need to be a part of

`g, ACCOUNT_NAME, role:some-defined-role`

Push these changes so Argo can pick them up and apply them to itself.

From here, initial passwords need to be setup by the initial admin user or a user with account update priviledges. This is done by using the argocd cli tool ([installed locally](https://argo-cd.readthedocs.io/en/stable/cli_installation/#installation)), where `CURRENT_USER_PASSWORD` is the password of the admin setting the users intial password

```
argocd login argocd.DOMAIN_HERE.com --grpc-web
argocd account list --grpc-web
argocd account update-password --account ACCOUNT_NAME --current-password CURRENT_USER_PASSWORD --new-password INITIAL_USER_PASSWORD --grpc-web
```

See [here](https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/user-management/index.md#manage-users) for more information on manging users.

## Adding / Updating RBAC

To create or modify roles and their access you need to amend the rules defined in ArgoCD values file `charts/values.yaml` under `argo-cd.configs.rbac`.

See [here](https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/rbac.md) for more information on RBAC.
