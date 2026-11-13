# Grafana GitOps add-on — Dell OptiPlex / K3s

This package adds a Grafana Application to the existing GitOps repository.

## 1. Copy into your Git repository

Copy the `gitops/` directory onto the existing repository's `gitops/` directory.
Edit `gitops/bootstrap/grafana-application.yaml` and replace `REPLACE_GIT_REPO_URL` with the same Git repository URL used for Nginx. The branch is `main`; change if needed.

## 2. Create admin credentials outside Git

Create the namespace and a secret on the Dell OptiPlex before the first Argo CD synchronization:

```bash
sudo k3s kubectl create namespace monitoring --dry-run=client -o yaml | sudo k3s kubectl apply -f -
read -rsp 'Grafana admin password: ' GRAFANA_PASSWORD; echo
sudo k3s kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$GRAFANA_PASSWORD" \
  --dry-run=client -o yaml | sudo k3s kubectl apply -f -
unset GRAFANA_PASSWORD
```

Do not commit the secret or password to Git.

## 3. Commit and push

```bash
git add gitops/apps/grafana gitops/bootstrap/grafana-application.yaml
git commit -m "feat: deploy grafana with Argo CD"
git push origin main
```

## 4. Bootstrap

After the commit is pushed, run from the repo's root on a system with K3s admin credentials:

```bash
sudo k3s kubectl apply -f gitops/bootstrap/grafana-application.yaml
```

Alternatively submit the same Application manifest in the Argo CD UI.

## 5. Access

Add to the workstation's hosts file (or internal DNS):

```
192.168.3.230 grafana.home.arpa
```

Open http://grafana.home.arpa, user `admin` and the password you created above.

## 6. Validate

```bash
sudo k3s kubectl get application grafana -n argocd
sudo k3s kubectl get pods,svc,ingress,pvc -n monitoring
sudo k3s kubectl get storageclass
curl -I -H 'Host: grafana.home.arpa' http://192.168.3.230/login
```

## Notes

- Chart pinned to 12.8.0 from grafana-community (verify compatibility before upgrading).
- Argo CD multi-source Application: chart from official Helm repo, `values.yaml` from Git.
- `local-path` stores data on the Dell node; there is no high availability or off-node backup.
- This example uses HTTP only on the trusted LAN. For production or untrusted networks, configure TLS, authentication hardening, and backups.
- The admin Secret is deliberately not managed by this Argo CD Application.
- Grafana is a visualization platform. Prometheus metrics are not yet configured.
