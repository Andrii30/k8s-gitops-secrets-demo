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

## Re-seal after a cluster rebuild
The Sealed Secrets controller generates a fresh private key per cluster
on first start. Every `SealedSecret` in this repo was encrypted against
the *old* cluster's public cert, so a rebuilt cluster can't decrypt them
— the controller logs `no key could decrypt secret` and the Secret never
materializes. Re-fetch the new cert and re-run the rotate command above:
```bash
kubectl -n sealed-secrets get secret -o jsonpath='{.items[0].data.tls\.crt}' \
  | base64 -d > /tmp/pub-cert.pem
```
then redo the `kubeseal ... > secrets/gitea-admin-sealedsecret.yaml` step.

## Secret scopes
This repo's SealedSecret uses the default `strict` scope — it only
decrypts into the exact `name`/`namespace` it was sealed for, so copying
the ciphertext into a different namespace's manifest silently fails to
decrypt. To intentionally share one sealed secret across namespaces,
seal with `--scope namespace-wide` (decrypts into any name in the
sealed namespace) or `--scope cluster-wide` (decrypts anywhere), e.g.:
```bash
kubeseal --cert /tmp/pub-cert.pem --scope cluster-wide --format yaml \
  < my-secret.yaml > secrets/my-secret-sealedsecret.yaml
```
Prefer `strict` unless you have a real cross-namespace consumer — wider
scopes mean ciphertext decrypts in more places than you may expect.

## Why this is safe
`secrets/gitea-admin-sealedsecret.yaml` is the only secret-shaped file in
this repo, and it only ever contains ciphertext — decryptable solely by
the Sealed Secrets controller's private key, which never leaves the
cluster.
