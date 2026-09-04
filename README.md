# k8s-gitops-secrets-demo

Encrypts a Gitea admin password client-side with Sealed Secrets so it's
safe to commit to git, decrypts it only in-cluster, and feeds it into a
real ArgoCD-managed Gitea deployment.
