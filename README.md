# k3s Gitops Setup with ArgoCD


This repository manages Kubernetes resources for a K3s cluster using **ArgoCD** and **GitOps**.


## Add More Applications
- Add new applications to apps/
- Commit & push changes to GitHub
- ArgoCD will automatically apply the new configurations!


## Features:
- GitOps-based Kubernetes Management
- Automatic Syncing with Git
- Declarative Infrastructure (except for secrets... will be fixed later though)


## Secrets
ArgoCD, and Kubernetes has some good documentation on how to properly set this up. But for the purpose of me not forgetting stuff, i will document my steps here.

### Private repositories (Github)
To authenticate via http, on github, you can create a secret to template all github repositories that your token can authenticate against:

store this in `base/argocd/secret.yaml` and make sure it's ignored by git, by adding it to `.gitignore`.

```
apiVersion: v1
kind: Secret
metadata:
  name: github-credentials
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
type: Opaque
stringData:
  url: https://github.com/
  username: <your-username>
  password: <your-github-access-token>
```
Then apply the secret to the argocd namespace.

```
kubectl apply -f base/argocd/secret.yaml
```

### Pulling private docker images from ghrc.io
To pull down private images from ghrc.io, you will need to add the following secret to your application's namespace:

```
apiVersion: v1
kind: Secret
metadata:
  name: ghrc-secret
  namespace: <your-applications-namespace>

type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-configuration-file>

```

To generate this secret, you have to do the following:

```
export CR_PAT=<your-github-access-token>
echo $CR_PAT | docker login ghcr.io -u <your-github-username> --password-stdin
```

This should generate, or update your existing `~/.docker/config.json` file.
You should see a `ghcr.io` entry with `"auth": <base64-encoded username:access-token>`

To generate the secret now, run the following:

```
kubectl create secret generic ghrc-secret --from-file=.dockerconfigjson=$HOME/.docker/config.json --type=kubernetes.io/dockerconfigjson --output "yaml"
```


You should now be able to specify this secret when pulling down private images:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: braum
  namespace: braum
spec:
    spec:
      imagePullSecrets:
      - name: ghrc-secret # <--- This is what you need to add.
      containers:
      - name: braum
        image: ghcr.io/dj-braum/braum:latest
        imagePullPolicy: Always
```
