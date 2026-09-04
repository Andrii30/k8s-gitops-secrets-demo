# k8s-gitops-secrets-demo

Encrypts a Gitea admin password client-side with Sealed Secrets so it's
safe to commit to git, decrypts it only in-cluster, and feeds it into a
real ArgoCD-managed Gitea deployment.

## Prerequisites
- A Kubernetes cluster with ArgoCD installed (shared setup, once per
  cluster).
- `kubeseal` CLI installed locally (`brew install kubeseal`).

## Deploy
```bash
kubectl apply -f argocd/sealed-secrets-controller.yaml
kubectl apply -f argocd/gitea-secrets.yaml
kubectl apply -f argocd/gitea-secure.yaml
```

## Rotate the secret
```bash
kubectl -n gitops-secrets-demo create secret generic gitea-admin-credentials \
  --dry-run=client \
  --from-literal=username=gitea_admin \
  --from-literal=password='<new-password>' \
  -o yaml | kubeseal --cert /tmp/pub-cert.pem --format yaml \
  > secrets/gitea-admin-sealedsecret.yaml
git add secrets/gitea-admin-sealedsecret.yaml
git commit -m "chore: rotate gitea admin password"
git push
```
ArgoCD picks up the change automatically; the controller re-decrypts and
Gitea picks up the new credential on its next restart.

## Why this is safe
`secrets/gitea-admin-sealedsecret.yaml` is the only secret-shaped file in
this repo, and it only ever contains ciphertext — decryptable solely by
the Sealed Secrets controller's private key, which never leaves the
cluster.
