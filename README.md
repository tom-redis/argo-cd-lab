# Installing Argo CD in the RSOT for K8s Lab

These instructions assume you are using the **RSOT for Kubernetes** lab environment in **PS Portal** and that the cluster is accessed via `sudo kubectl` from the lab host.

---

## 1. Access the lab and verify cluster connectivity

1. In **PS Portal**, create or open an **RSOT for K8s** lab environment.
2. From the lab UI, open the **terminal/SSH** link for the environment.
3. In the terminal, confirm you can talk to the cluster:

   ```bash
   sudo kubectl get nodes
   sudo kubectl get pods -A
   ```

4. For convenience, define an alias:

   ```bash
   alias k='sudo kubectl'
   ```

All subsequent commands in this document assume `k` is defined as above.

---

## 2. Create the Argo CD namespace

Install Argo CD into its own namespace:

```bash
k create namespace argocd
```

You can re-run this safely; it will be a no-op if the namespace already exists.

---

## 3. Install Argo CD components

Use the official “all-in-one” installation manifest:

```bash
k apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all Argo CD pods to become ready:

```bash
k get pods -n argocd
```

You should see pods such as:

- `argocd-server`
- `argocd-repo-server`
- `argocd-application-controller`
- `argocd-dex-server`
- `argocd-notifications-controller`
- `argocd-redis-ha-*`

All should be in `Running` and `READY` status before continuing.

---

## 4. (Optional) Install the Argo CD CLI in the lab

Installing the CLI on the lab host makes it easier to work with Argo CD.

```bash
curl -sSL -o argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

Verify:

```bash
argocd version
```

(If this fails, make sure `/usr/local/bin` is in your `$PATH`.)

---

## 5. Retrieve the initial admin password

Argo CD initializes an `admin` account and stores the password in a Kubernetes Secret.

```bash
k -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Copy this value; you will need it to log in as `admin`.

For convenience:

```bash
export ARGOCD_ADMIN_PASSWORD="$(k -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d)"
```

---

## 6. Expose the Argo CD API server (port-forward)

In the RSOT for K8s lab, the simplest way to access Argo CD is port-forwarding from the lab host:

```bash
k -n argocd port-forward svc/argocd-server 8080:80
```

Leave this running in a dedicated terminal.

### 6.1. Web UI access

Depending on how PS Portal exposes forwarded ports, you can usually:

- Use the IDE/terminal “open port 8080” feature, or
- Open a browser from your local machine to the lab’s forwarded URL (if provided by the lab UI).


In RSOT labs, HTTP(S) access goes through the `*.labs.ps-redis.com` domain. If your instance name is `instanceName` and **the lab template exposes port 8080**, a URL like the following may work:

```text
https://8080-dot-<instanceName>.labs.ps-redis.com
```

where &lt;instanceName&gt; is the unique 6-character string in the URL, "rl-s-labs-*xxxxxx*.ps-redis.com"

If that URL returns **502 Bad Gateway**, it usually means PS Portal is **not** forwarding that port for this lab template. In that case, use the CLI flow in §6.2 (port-forward + `argocd login localhost:8080`) instead of trying to reach Argo CD from your local browser.

When (if) the browser URL does work, you can log in with:

- **Username:** `admin`  
- **Password:** value from `ARGOCD_ADMIN_PASSWORD`

If you see a TLS warning, accept it for the lab environment.

### 6.2. CLI login

If you prefer to work only from the terminal:

```bash
argocd login localhost:8080 \
  --username admin \
  --password "$ARGOCD_ADMIN_PASSWORD" \
  --insecure
```

You can now use `argocd app ...` and other CLI commands.

---

## 7. Register the in-cluster Kubernetes target (optional)

For the simplest use case, you can reference the in-cluster API server directly in your `Application` manifests (`server: https://kubernetes.default.svc`) without any extra steps.

If you want to register the current cluster explicitly, first copy the kubeconfig that `sudo kubectl` uses into your own home directory (so `argocd` can see the same context):

```bash
# Create kube dir for labuser
mkdir -p ~/.kube

# Copy root's kubeconfig into your home
sudo cp /root/.kube/config ~/.kube/config

# Fix ownership and permissions
sudo chown labuser:labuser ~/.kube/config
chmod 600 ~/.kube/config
```

Now use **plain** `kubectl` (no sudo) to confirm the context and add the cluster:

```bash
kubectl config current-context    # should print kind-kind (or similar)
CURRENT_CONTEXT="$(kubectl config current-context)"
argocd cluster add "$CURRENT_CONTEXT"
```

Follow any prompts. This will create a ServiceAccount and RBAC bindings for Argo CD in the cluster.

---

## 8. Example: Argo CD Application for RSOT workloads

The following example assumes:

- You have a Git repository with example RSOT K8s manifests, and
- You want Argo CD to deploy them into a namespace in the *same* cluster.

Create a file named `rsot-demo-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rsot-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:tom-redis/argo-cd-lab.git
    targetRevision: HEAD
    path: k8s/rsot-demo   # adjust to the actual path in your repo
  destination:
    server: https://kubernetes.default.svc
    namespace: rsot-demo   # or an existing namespace such as redis-enterprise
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply the Application:

```bash
k apply -f rsot-demo-app.yaml
```

Sync and inspect it with the CLI:

```bash
argocd app sync rsot-demo
argocd app get rsot-demo
```

Or use the Web UI to visualize the Application tree, health, and sync status.

---

## 9. Cleanup

To remove Argo CD from the RSOT for K8s lab:

```bash
k delete -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

k delete namespace argocd
```

If you created any Argo-managed workloads (Applications, namespaces, etc.), delete those separately.

---

## 10. Notes and best practices for RSOT for K8s

- **Scope:** These steps are intended for **your own testing/dry runs**. For customer-facing RSOT deliveries, avoid modifying the shared lab image beyond the official exercises unless you explicitly need to demo Argo CD.
- **Persistence:** RSOT labs are ephemeral. When the lab is torn down, any Argo CD state in the cluster is lost. Keep your manifests and configuration in Git so you can recreate Argo CD quickly in a fresh lab.
- **Security:** This setup uses default Argo CD credentials and `--insecure` CLI options for convenience in an isolated training environment. Do **not** reuse this pattern as-is in production clusters.
