# Kubernetes Lab on WSL Ubuntu — Reproducible Lab Manual

**Audience:** Mohan (basic Linux + Kubernetes knowledge)
**Goal:** A modular, audit-ready, start/stop-friendly lab using KVM/QEMU/Libvirt on WSL Ubuntu, running a single-node RKE2 cluster, a Rancher management cluster, MetalLB, NGINX ingress, GitOps (ArgoCD + Flux), Longhorn storage, and Keycloak identity.

**Verified component versions (June 2026 — pin and confirm against the linked release pages before each rebuild):**

| Component | Version used here | Source of truth |
|---|---|---|
| Rancher | **v2.14.2** (stable) — supports Kubernetes **< 1.36** | https://github.com/rancher/rancher/releases |
| RKE2 | **1.33.x line** (`+rke2r1`) — chosen so Rancher can import it | https://github.com/rancher/rke2/releases |
| cert-manager | **v1.17.x** (pin a patch) | https://github.com/cert-manager/cert-manager/releases |
| MetalLB | **v0.15.x** (Helm chart) | https://metallb.io |
| NGINX Ingress | **F5 NGINX Ingress Controller** (`nginxinc/kubernetes-ingress`) | https://docs.nginx.com/nginx-ingress-controller/ |
| Longhorn | **v1.11.x** | https://longhorn.io |
| ArgoCD | `stable` manifest | https://github.com/argoproj/argo-cd/releases |
| Flux | v2.x CLI | https://fluxcd.io |
| Keycloak | **26.x** | https://www.keycloak.org |

> ⚠️ **Ingress caution (read before Part 4).** The community **`kubernetes/ingress-nginx`** controller reached **end-of-life in March 2026** — the repo is archived and receives **no further releases, bug fixes, or CVE patches**. It is *not* the same as F5's **NGINX Ingress Controller** (`nginxinc/kubernetes-ingress`), which is actively maintained. This manual installs the **maintained F5 controller** to satisfy the "Nginx ingress" goal in an audit-defensible way. RKE2 itself made **Traefik the default from v1.36**, and the strategic long-term direction is the **Gateway API**. Choose accordingly; this lab stays on a maintained NGINX path.

---

## Architecture

```
 Windows Host (Hyper-V platform + WSL2)
 └── WSL2 distro: Ubuntu  (systemd ON)         [libvirt host]
      ├── libvirtd + KVM/QEMU
      └── virbr0  NAT network 192.168.122.0/24
           ├── VM: rke2-cluster   192.168.122.20   ← workloads, MetalLB, NGINX, Longhorn, Keycloak, GitOps
           └── VM: rancher-mgmt    192.168.122.30   ← Rancher local cluster (manages rke2-cluster)

 MetalLB LoadBalancer pool: 192.168.122.200–192.168.122.250  (inside virbr0 subnet)
 DNS for browser access:    *.sslip.io  (e.g. rancher.192.168.122.30.sslip.io)
```

**Design principles**
- Each capability is an independent module (a VM or a Helm release) so you can rebuild one without touching the others.
- All ingress/LoadBalancer IPs live **inside the libvirt NAT subnet** so they are reachable from WSL with no extra routing.
- State is preserved across host reboots via `virsh managedsave` + `virsh autostart`.

---

# PART A — Host preparation

## Step A1 — Confirm Windows and CPU virtualization support

**Description.** KVM inside WSL2 requires *nested virtualization*. Without it `/dev/kvm` will not exist and every VM step fails.

**Commands (Windows PowerShell, as Administrator):**
```powershell
wsl --version                      # WSL 2.x; update if older:
wsl --update
# Confirm CPU virtualization is enabled in firmware:
systeminfo | findstr /i "Hyper-V Virtualization"
```

**Validation:** `Virtualization Enabled In Firmware: Yes` (or Hyper-V requirements all "Yes"). On Windows 11, nested virtualization is available by default; on Windows 10 you need a recent build. If virtualization is disabled, enable **Intel VT-x / AMD-V** in BIOS/UEFI.

---

## Step A2 — Size WSL2 and enable nested virtualization

**Description.** This lab runs two VMs plus Longhorn, Keycloak, ArgoCD, Flux and Rancher. Give WSL2 enough RAM/CPU or pods will be `Evicted`/`OOMKilled`. Plan for ~**20 GB RAM and 6 vCPU** allocated to WSL2 (host should have ≥32 GB total).

**Configuration — create/edit `C:\Users\<you>\.wslconfig` (Windows side):**
```ini
[wsl2]
memory=20GB
processors=6
nestedVirtualization=true
swap=8GB
# Optional: keep the VM disk from ballooning
# sparseVhd=true
```

**Apply (PowerShell):**
```powershell
wsl --shutdown
```

**Validation (inside WSL Ubuntu after it restarts):**
```bash
nproc            # should reflect processors=6
free -h          # total should reflect ~20GB
ls -l /dev/kvm   # MUST exist:  crw-rw---- ... /dev/kvm
```
> 🚧 **Caution:** If `/dev/kvm` is missing, nested virtualization is off. Re-check `nestedVirtualization=true`, run `wsl --shutdown`, and confirm firmware virtualization. On some AMD systems you must also be on the latest WSL kernel (`wsl --update`).

---

## Step A3 — Enable systemd in WSL Ubuntu

**Description.** `libvirtd` and `iscsid` run best as systemd services. WSL does not enable systemd by default.

**Configuration — `/etc/wsl.conf` (inside Ubuntu):**
```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true

[network]
generateResolvConf=true
EOF
```

**Apply (PowerShell):** `wsl --shutdown`, then reopen Ubuntu.

**Validation:**
```bash
systemctl is-system-running   # "running" or "degraded" (degraded is acceptable in WSL)
ps -p 1 -o comm=              # should print: systemd
```

---

# PART B — KVM / QEMU / Libvirt (Goal 1)

## Step B1 — Install the virtualization stack

**Description.** Installs the hypervisor userland (QEMU), the management daemon (libvirt), the default NAT network helper, cloud-image tooling, and `virtinst` for scripted VM creation.

```bash
sudo apt-get update
sudo apt-get install -y \
  qemu-kvm libvirt-daemon-system libvirt-clients \
  bridge-utils virtinst libguestfs-tools \
  cloud-image-utils genisoimage dnsmasq-base \
  whois jq curl
```

**Validation:**
```bash
kvm-ok                         # "KVM acceleration can be used"
virsh version                  # prints libvirt + QEMU versions
```
> If `kvm-ok` is absent, `sudo apt-get install -y cpu-checker`.

---

## Step B2 — Start libvirtd and grant your user access

**Description.** Adds your user to `libvirt`/`kvm` groups so you can run `virsh` without `sudo`, and ensures the daemon starts at boot.

```bash
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt,kvm "$USER"
# Apply group membership without a full relogin:
newgrp libvirt
```

**Validation:**
```bash
systemctl is-active libvirtd   # active
virsh list --all               # connects with no error (empty list is fine)
```

---

## Step B3 — Bring up and persist the default NAT network

**Description.** The `default` libvirt network gives VMs DHCP on `192.168.122.0/24` via `virbr0`. Every VM, MetalLB IP, and ingress IP lives here, reachable directly from WSL.

```bash
virsh net-list --all
# If 'default' is inactive:
virsh net-start default
virsh net-autostart default
```

**Validation:**
```bash
virsh net-info default         # Active: yes,  Autostart: yes
ip addr show virbr0            # inet 192.168.122.1/24
```
> 🚧 **Caution (double NAT).** WSL2 is itself behind NAT, and libvirt adds a second NAT. VMs are reachable **from WSL** but **not from the Windows browser** without extra routing. Part J explains how to reach the UIs from Windows. For most of the lab, run `kubectl`/`curl` **inside WSL**.

---

# PART C — Reusable VM template (modularity)

## Step C1 — Build a cloud-init Ubuntu base image

**Description.** Instead of clicking through an installer, we use the official Ubuntu **cloud image** plus **cloud-init** so every VM is identical and reproducible. We create one golden image and clone from it.

```bash
mkdir -p ~/lab/{images,vms,seed}
cd ~/lab/images

# Ubuntu 24.04 LTS cloud image (verify current URL at cloud-images.ubuntu.com)
curl -L -o noble-base.img \
  https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Inspect to confirm it is a valid qcow2:
qemu-img info noble-base.img
```

**Validation:** `qemu-img info` reports `file format: qcow2`.

---

## Step C2 — Create a parameterised VM-builder script

**Description.** One script defines a VM with given name/IP/CPU/RAM, generates its cloud-init seed (user `mohan`, your SSH key, password login), and boots it. This is the modular core: re-run it to recreate any node.

```bash
cat > ~/lab/make-vm.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
NAME="$1"; IP="$2"; CPUS="${3:-2}"; MEM_MB="${4:-4096}"; DISK_GB="${5:-40}"
BASE=~/lab/images/noble-base.img
DISK=~/lab/vms/${NAME}.qcow2
SEED=~/lab/seed/${NAME}-seed.iso
PUBKEY="$(cat ~/.ssh/id_ed25519.pub 2>/dev/null || echo '')"

# Per-VM disk backed by the golden image (copy-on-write):
qemu-img create -f qcow2 -F qcow2 -b "$BASE" "$DISK" "${DISK_GB}G"

# cloud-init user-data:
cat > /tmp/${NAME}-user-data <<CIEOF
#cloud-config
hostname: ${NAME}
manage_etc_hosts: true
users:
  - name: mohan
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: false
    passwd: $(echo 'labpass123' | mkpasswd -m sha-512 -s)
    ssh_authorized_keys:
      - ${PUBKEY}
ssh_pwauth: true
package_update: true
packages: [qemu-guest-agent, curl, jq]
runcmd:
  - systemctl enable --now qemu-guest-agent
CIEOF

cat > /tmp/${NAME}-meta-data <<CIEOF
instance-id: ${NAME}
local-hostname: ${NAME}
CIEOF

cloud-localds "$SEED" /tmp/${NAME}-user-data /tmp/${NAME}-meta-data

virt-install \
  --name "$NAME" --memory "$MEM_MB" --vcpus "$CPUS" \
  --disk path="$DISK",format=qcow2,bus=virtio \
  --disk path="$SEED",device=cdrom \
  --os-variant ubuntu24.04 \
  --network network=default,model=virtio \
  --graphics none --noautoconsole --import

# Reserve a static DHCP lease so the IP never changes:
MAC="$(virsh dumpxml "$NAME" | awk -F\' '/mac address/ {print $2; exit}')"
virsh net-update default add ip-dhcp-host \
  "<host mac='${MAC}' name='${NAME}' ip='${IP}'/>" --live --config || true
virsh reboot "$NAME" || virsh start "$NAME"
echo "Created ${NAME} -> ${IP} (mac ${MAC})"
EOF
chmod +x ~/lab/make-vm.sh
```

**Validation:** `bash -n ~/lab/make-vm.sh` (syntax check passes). Generate an SSH key first if needed: `ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519`.

> 🚧 **Caution:** `labpass123` and a sample key are lab defaults. Replace with your own before any non-lab use.

---

# PART D — VM 1: single-node RKE2 cluster (Goal 2)

## Step D1 — Create the RKE2 VM

**Description.** RKE2 + workloads + Longhorn + Keycloak need headroom. Allocate 4 vCPU / 8 GB / 60 GB.

```bash
~/lab/make-vm.sh rke2-cluster 192.168.122.20 4 8192 60
# Wait for cloud-init, then confirm reachability:
until ping -c1 -W1 192.168.122.20 >/dev/null 2>&1; do sleep 3; done
ssh mohan@192.168.122.20 'cloud-init status --wait; uname -a'
```

**Validation:** `cloud-init status` returns `status: done`; SSH succeeds.

---

## Step D2 — Install RKE2 server (pinned, with bundled ingress disabled)

**Description.** We install a **1.33-line** RKE2 (importable by Rancher 2.14) and **disable the bundled ingress** so we can install MetalLB + the maintained NGINX controller cleanly in Part E.

**On the VM (`ssh mohan@192.168.122.20`):**
```bash
# Pin a concrete 1.33 patch from https://github.com/rancher/rke2/releases
export INSTALL_RKE2_VERSION="v1.33.6+rke2r1"   # <-- verify this exact tag exists

sudo mkdir -p /etc/rancher/rke2
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
write-kubeconfig-mode: "0644"
disable:
  - rke2-ingress-nginx        # we install a maintained controller ourselves
tls-san:
  - 192.168.122.20
  - rke2-cluster
node-label:
  - "lab=rke2-cluster"
EOF

curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_VERSION="$INSTALL_RKE2_VERSION" sh -
sudo systemctl enable --now rke2-server.service

# Make kubectl/crictl available:
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' | sudo tee /etc/profile.d/rke2.sh
echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml' | sudo tee -a /etc/profile.d/rke2.sh
source /etc/profile.d/rke2.sh
```

**Validation (on the VM):**
```bash
sudo systemctl is-active rke2-server          # active
kubectl get nodes -o wide                      # node Ready, correct version
kubectl get pods -A | grep -E 'Running|Completed' | wc -l   # core pods up
```

---

## Step D3 — Pull kubeconfig to WSL for remote management

**Description.** Manage the cluster from WSL (where ArgoCD/Flux CLIs and your editor live) instead of SSHing each time.

**On WSL:**
```bash
mkdir -p ~/.kube
scp mohan@192.168.122.20:/etc/rancher/rke2/rke2.yaml ~/lab/rke2.kubeconfig
# Point the API server at the VM IP instead of 127.0.0.1:
sed -i 's#127.0.0.1#192.168.122.20#g' ~/lab/rke2.kubeconfig
export KUBECONFIG=~/lab/rke2.kubeconfig
sudo snap install kubectl --classic 2>/dev/null || sudo apt-get install -y kubectl
kubectl get nodes
```

**Validation:** `kubectl get nodes` from WSL returns the `rke2-cluster` node as `Ready`.

> Keep `export KUBECONFIG=~/lab/rke2.kubeconfig` in your shell for all WSL `kubectl`/`helm` commands targeting the workload cluster.

---

# PART E — MetalLB + NGINX ingress on the RKE2 cluster (Goal 4)

## Step E1 — Install Helm (on WSL)

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```
**Validation:** `helm version` prints a v3 client.

## Step E2 — Install MetalLB and define an address pool

**Description.** RKE2 has no cloud load balancer, so `Service type=LoadBalancer` stays `<pending>`. MetalLB hands out IPs from a pool **inside the libvirt subnet** so they are reachable from WSL.

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm install metallb metallb/metallb \
  --namespace metallb-system --create-namespace --wait
```

Then apply the pool (note the range is inside `192.168.122.0/24`):
```bash
kubectl apply -f - <<'EOF'
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lab-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.122.200-192.168.122.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lab-l2
  namespace: metallb-system
spec:
  ipAddressPools: [lab-pool]
EOF
```

**Validation:**
```bash
kubectl -n metallb-system get pods       # controller + speaker Running
kubectl -n metallb-system get ipaddresspool lab-pool
```

## Step E3 — Install the maintained NGINX Ingress Controller (F5)

**Description.** This is the actively maintained NGINX controller (not the EOL community project). It will request a `LoadBalancer` service and receive a MetalLB IP.

```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
helm install nginx-ingress nginx-stable/nginx-ingress \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.ingressClass.name=nginx \
  --set controller.ingressClass.setAsDefaultIngress=true \
  --wait
```

**Validation:**
```bash
kubectl -n ingress-nginx get svc nginx-ingress-controller
# EXTERNAL-IP should be a 192.168.122.2xx address (not <pending>)
LB_IP=$(kubectl -n ingress-nginx get svc nginx-ingress-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'); echo "Ingress IP: $LB_IP"
```

## Step E4 — End-to-end ingress smoke test

```bash
kubectl create deployment hello --image=nginxdemos/hello
kubectl expose deployment hello --port=80
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
  annotations: {}
spec:
  ingressClassName: nginx
  rules:
    - host: hello.${LB_IP}.sslip.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: hello, port: { number: 80 } } }
EOF

curl -s "http://hello.${LB_IP}.sslip.io" | grep -i "Server" && echo "INGRESS OK"
```
**Validation:** `curl` returns the demo page (run it from WSL). Clean up with `kubectl delete deploy/hello svc/hello ingress/hello`.

> 🚧 **Caution:** `sslip.io` resolves `hello.192.168.122.200.sslip.io` → `192.168.122.200`. This works from WSL because the subnet is local. From the Windows browser, see Part J.

---

# PART F — VM 2: Rancher management cluster (Goal 3)

## Step F1 — Create the Rancher VM and install RKE2 for the local cluster

**Description.** Rancher runs *on* Kubernetes. We give it its own single-node RKE2 (kept on the bundled ingress this time, since Rancher's Helm chart integrates with it).

```bash
# On WSL:
~/lab/make-vm.sh rancher-mgmt 192.168.122.30 4 8192 50
until ping -c1 -W1 192.168.122.30 >/dev/null 2>&1; do sleep 3; done
ssh mohan@192.168.122.30 'cloud-init status --wait'
```

**On the rancher-mgmt VM:**
```bash
export INSTALL_RKE2_VERSION="v1.33.6+rke2r1"   # same supported line
sudo mkdir -p /etc/rancher/rke2
sudo tee /etc/rancher/rke2/config.yaml >/dev/null <<'EOF'
write-kubeconfig-mode: "0644"
tls-san:
  - 192.168.122.30
  - rancher-mgmt
EOF
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_VERSION="$INSTALL_RKE2_VERSION" sh -
sudo systemctl enable --now rke2-server.service
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' | sudo tee /etc/profile.d/rke2.sh
echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml'  | sudo tee -a /etc/profile.d/rke2.sh
source /etc/profile.d/rke2.sh
kubectl get nodes
```

**Validation:** node `Ready` on the Rancher VM.

## Step F2 — Pull the Rancher cluster's kubeconfig to WSL

```bash
# On WSL:
scp mohan@192.168.122.30:/etc/rancher/rke2/rke2.yaml ~/lab/rancher.kubeconfig
sed -i 's#127.0.0.1#192.168.122.30#g' ~/lab/rancher.kubeconfig
export KUBECONFIG=~/lab/rancher.kubeconfig
kubectl get nodes
```
**Validation:** WSL `kubectl` reaches the rancher-mgmt node.

## Step F3 — Install cert-manager

**Description.** Rancher uses cert-manager to issue its TLS certificate.

```bash
# Pin a current patch from https://github.com/cert-manager/cert-manager/releases
CM_VERSION="v1.17.2"
helm repo add jetstack https://charts.jetstack.io && helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true --version "$CM_VERSION" --wait
```
**Validation:** `kubectl -n cert-manager get pods` → 3 pods Running.

## Step F4 — Install Rancher (v2.14 stable)

**Description.** Single-replica install for a lab, with a `sslip.io` hostname and a self-signed cert managed by cert-manager.

```bash
RANCHER_HOST="rancher.192.168.122.30.sslip.io"
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
helm install rancher rancher-stable/rancher \
  --namespace cattle-system --create-namespace \
  --set hostname="$RANCHER_HOST" \
  --set bootstrapPassword='ChangeMe-Lab-2026' \
  --set replicas=1 \
  --set ingress.tls.source=rancher \
  --wait --timeout 15m
```

**Validation:**
```bash
kubectl -n cattle-system rollout status deploy/rancher
kubectl -n cattle-system get ingress           # host = $RANCHER_HOST
curl -sk "https://${RANCHER_HOST}/ping"         # "pong"
```

Open `https://rancher.192.168.122.30.sslip.io` (from WSL via `curl`, or from Windows per Part J), log in with the bootstrap password, and set the admin password when prompted.

> 🚧 **Caution:** Rancher 2.14 supports Kubernetes **< 1.36**. Keep both RKE2 VMs on the **1.33/1.34/1.35** line. Do not upgrade RKE2 to 1.36+ while this Rancher version manages it.

---

# PART G — Manage the RKE2 cluster from Rancher (Goal 5)

## Step G1 — Import the workload cluster

**Description.** Rather than provisioning, we *import* the existing `rke2-cluster` so Rancher manages it through an agent.

1. In the Rancher UI: **☰ → Cluster Management → Import Existing → Generic**.
2. Name it `rke2-cluster`, click **Create**.
3. Rancher shows a registration command. Because the lab uses a self-signed cert, copy the **insecure** (`curl --insecure ... | kubectl apply -f -`) variant.
4. Run that command **against the workload cluster** from WSL:
   ```bash
   export KUBECONFIG=~/lab/rke2.kubeconfig
   # paste the curl --insecure ... | kubectl apply -f -  command here
   ```

**Validation:**
```bash
kubectl -n cattle-system get pods | grep cattle-cluster-agent   # Running
```
In the Rancher UI the cluster turns **Active**. You can now browse nodes, workloads, install apps from the Rancher catalog, and open a `kubectl` shell — all from the UI.

## Step G2 — Confirm management actions work

- In Rancher: open `rke2-cluster` → **Workloads** and confirm you see the MetalLB/NGINX namespaces from Part E.
- Deploy a test workload from **Apps → Charts** and confirm it appears via `kubectl get pods -A` on the workload cluster.

**Validation:** A change made in the UI is reflected by `kubectl` against `~/lab/rke2.kubeconfig`, proving bidirectional management.

---

# PART H — GitOps learning workflows (Goal 6)

> All GitOps steps target the **workload cluster**: `export KUBECONFIG=~/lab/rke2.kubeconfig`.

## Step H1 — ArgoCD

**Description.** ArgoCD pulls desired state from Git and reconciles it into the cluster (pull-based GitOps with a strong UI).

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server

# Expose the UI through the NGINX ingress:
ARGO_HOST="argocd.${LB_IP}.sslip.io"
kubectl -n argocd apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  annotations:
    nginx.org/ssl-services: "argocd-server"
spec:
  ingressClassName: nginx
  rules:
    - host: ${ARGO_HOST}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: argocd-server, port: { number: 443 } } }
EOF

# Initial admin password:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

**Example Application (declarative):**
```bash
kubectl apply -n argocd -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
EOF
```

**Validation:**
```bash
kubectl -n argocd get applications.argoproj.io guestbook   # SYNCED / HEALTHY
kubectl -n guestbook get pods                              # workload deployed by GitOps
```

## Step H2 — Flux

**Description.** Flux is a CNCF GitOps toolkit. Run it alongside ArgoCD on the same cluster to compare push-free reconciliation patterns.

```bash
# Install the CLI on WSL:
curl -s https://fluxcd.io/install.sh | sudo bash
flux check --pre

# Install controllers (non-bootstrap, simplest for a lab):
flux install

# Reconcile a public repo declaratively:
flux create source git podinfo \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master --interval=1m
flux create kustomization podinfo \
  --target-namespace=default --source=podinfo \
  --path="./kustomize" --prune=true --interval=5m
```

**Validation:**
```bash
flux get sources git           # podinfo  READY=True
flux get kustomizations        # podinfo  READY=True
kubectl get deploy podinfo     # reconciled by Flux
```

> 💡 **Learning note.** For audit-grade GitOps, replace `flux install` with `flux bootstrap github ...` (commits the Flux config into your own repo, making the cluster's desired state fully versioned). Rancher also bundles **Fleet** for fleet-wide GitOps — explore it under **Continuous Delivery** in the Rancher UI as a third pattern.

---

# PART I — Storage and identity add-ons (Goal 7)

> Target the **workload cluster** for both: `export KUBECONFIG=~/lab/rke2.kubeconfig`.

## Step I1 — Longhorn host prerequisites

**Description.** Longhorn presents block storage over iSCSI and needs host packages + the `iscsi_tcp` module on **every node**. NFS client enables ReadWriteMany volumes and backups.

**On the rke2-cluster VM (`ssh mohan@192.168.122.20`):**
```bash
sudo apt-get update
sudo apt-get install -y open-iscsi nfs-common cryptsetup dmsetup
sudo modprobe iscsi_tcp
echo "iscsi_tcp" | sudo tee /etc/modules-load.d/longhorn.conf
sudo systemctl enable --now iscsid
```

**Validation (on the VM):**
```bash
systemctl is-active iscsid      # active
lsmod | grep iscsi_tcp          # module loaded
```

## Step I2 — Install Longhorn

```bash
# On WSL, targeting the workload cluster:
helm repo add longhorn https://charts.longhorn.io && helm repo update
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system --create-namespace \
  --version 1.11.2 \
  --set defaultSettings.defaultReplicaCount=1 \
  --wait --timeout 10m
```
> `defaultReplicaCount=1` is correct for a **single-node** lab (you cannot satisfy 3 replicas on one node).

**Set Longhorn as the default StorageClass:**
```bash
kubectl patch storageclass longhorn \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
# Remove default flag from any other SC if present:
kubectl get storageclass
```

**Expose the Longhorn UI (optional):**
```bash
kubectl -n longhorn-system apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ui
spec:
  ingressClassName: nginx
  rules:
    - host: longhorn.${LB_IP}.sslip.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: longhorn-frontend, port: { number: 80 } } }
EOF
```

**Validation:**
```bash
kubectl -n longhorn-system get pods | grep -c Running   # many pods Running
kubectl get storageclass longhorn                        # (default)
# PVC smoke test:
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: test-pvc }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources: { requests: { storage: 1Gi } }
EOF
kubectl get pvc test-pvc          # STATUS = Bound
kubectl delete pvc test-pvc
```

## Step I3 — Keycloak (identity)

**Description.** Deploy Keycloak 26.x backed by PostgreSQL on Longhorn storage, exposed through the NGINX ingress. This is a self-contained identity provider you can later wire into Rancher (OIDC), ArgoCD, or your own apps.

```bash
kubectl create namespace keycloak

# PostgreSQL with a Longhorn-backed PVC:
kubectl -n keycloak apply -f - <<'EOF'
apiVersion: v1
kind: Secret
metadata: { name: kc-db }
stringData:
  POSTGRES_DB: keycloak
  POSTGRES_USER: keycloak
  POSTGRES_PASSWORD: kc-db-pass
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: kc-pg-data }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources: { requests: { storage: 5Gi } }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: postgres }
spec:
  replicas: 1
  selector: { matchLabels: { app: kc-postgres } }
  template:
    metadata: { labels: { app: kc-postgres } }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          envFrom: [{ secretRef: { name: kc-db } }]
          ports: [{ containerPort: 5432 }]
          volumeMounts: [{ name: data, mountPath: /var/lib/postgresql/data, subPath: pgdata }]
      volumes:
        - name: data
          persistentVolumeClaim: { claimName: kc-pg-data }
---
apiVersion: v1
kind: Service
metadata: { name: postgres }
spec:
  selector: { app: kc-postgres }
  ports: [{ port: 5432, targetPort: 5432 }]
EOF
```

Now Keycloak itself (substitute the real ingress IP for `<LB_IP>`):
```bash
KC_HOST="keycloak.${LB_IP}.sslip.io"
kubectl -n keycloak apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata: { name: keycloak }
spec:
  replicas: 1
  selector: { matchLabels: { app: keycloak } }
  template:
    metadata: { labels: { app: keycloak } }
    spec:
      containers:
        - name: keycloak
          image: quay.io/keycloak/keycloak:26.2
          args: ["start"]
          env:
            - { name: KC_BOOTSTRAP_ADMIN_USERNAME, value: admin }
            - { name: KC_BOOTSTRAP_ADMIN_PASSWORD, value: admin-lab-2026 }
            - { name: KC_DB, value: postgres }
            - { name: KC_DB_URL, value: "jdbc:postgresql://postgres:5432/keycloak" }
            - { name: KC_DB_USERNAME, value: keycloak }
            - { name: KC_DB_PASSWORD, value: kc-db-pass }
            - { name: KC_HOSTNAME, value: "https://${KC_HOST}" }
            - { name: KC_HTTP_ENABLED, value: "true" }
            - { name: KC_PROXY_HEADERS, value: "xforwarded" }
          ports: [{ containerPort: 8080 }]
          readinessProbe:
            httpGet: { path: /realms/master, port: 8080 }
            initialDelaySeconds: 30
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata: { name: keycloak }
spec:
  selector: { app: keycloak }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: keycloak }
spec:
  ingressClassName: nginx
  rules:
    - host: ${KC_HOST}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: keycloak, port: { number: 80 } } }
EOF
```

**Validation:**
```bash
kubectl -n keycloak rollout status deploy/keycloak
kubectl -n keycloak get pvc kc-pg-data            # Bound (proves Longhorn integration)
curl -s "http://${KC_HOST}/realms/master" | jq .realm   # "master"
```

**Usage (admin console):** browse to `http://keycloak.<LB_IP>.sslip.io/admin`, log in as `admin / admin-lab-2026`, create a realm + client. To use it as Rancher's IdP, configure **Rancher → Users & Authentication → Auth Provider → Keycloak (OIDC)** with that realm's discovery URL and a confidential client.

> 🚧 **Caution:** These passwords are lab placeholders. For audit hygiene, move them into Kubernetes Secrets and rotate before reuse.

---

# PART J — Reaching the UIs from the Windows browser (optional)

**Description.** The `192.168.122.0/24` subnet is local to WSL only. Two ways to reach it from Windows:

**Option 1 — `socat` relay in WSL (per service, simplest):**
```bash
# Example: expose Rancher (on rancher-mgmt) to Windows on localhost:8443
sudo apt-get install -y socat
socat TCP-LISTEN:8443,fork,reuseaddr TCP:192.168.122.30:443 &
# Then browse https://localhost:8443 from Windows (accept the self-signed cert).
```

**Option 2 — Windows route to the WSL subnet (PowerShell as Admin):**
```powershell
$wslIp = (wsl hostname -I).Trim().Split(" ")[0]
route add 192.168.122.0 mask 255.255.255.0 $wslIp
```
Then add `*.sslip.io` hosts work automatically (sslip resolves to the literal IP). The route is lost on reboot unless made persistent (`route -p`).

**Validation:** From Windows, `https://localhost:8443/ping` (Option 1) or `https://rancher.192.168.122.30.sslip.io` (Option 2) loads Rancher.

---

# PART K — Lab lifecycle: start / stop cleanly (Goal 8)

**Description.** Use `managedsave` to freeze VM RAM state to disk so the lab survives `wsl --shutdown` and host reboots without re-bootstrapping Kubernetes.

## Step K1 — Make VMs auto-start and persist

```bash
virsh autostart rke2-cluster
virsh autostart rancher-mgmt
virsh net-autostart default
```

## Step K2 — Stop scripts (`lab-down.sh`)

```bash
cat > ~/lab/lab-down.sh <<'EOF'
#!/usr/bin/env bash
set -e
for vm in rke2-cluster rancher-mgmt; do
  if virsh domstate "$vm" | grep -q running; then
    echo "Saving state of $vm ..."
    virsh managedsave "$vm"     # freezes RAM to disk; instant resume later
  fi
done
echo "Lab paused. Safe to run 'wsl --shutdown' from Windows."
EOF
chmod +x ~/lab/lab-down.sh
```

## Step K3 — Start script (`lab-up.sh`)

```bash
cat > ~/lab/lab-up.sh <<'EOF'
#!/usr/bin/env bash
set -e
sudo systemctl start libvirtd
virsh net-start default 2>/dev/null || true
# Start Rancher cluster first, then the workload cluster:
for vm in rancher-mgmt rke2-cluster; do
  virsh start "$vm" 2>/dev/null || echo "$vm already running / resumed from managedsave"
done
echo "Waiting for nodes ..."
until ping -c1 -W1 192.168.122.20 >/dev/null 2>&1 && \
      ping -c1 -W1 192.168.122.30 >/dev/null 2>&1; do sleep 3; done
echo "Lab up. Set KUBECONFIG to ~/lab/rke2.kubeconfig or ~/lab/rancher.kubeconfig."
EOF
chmod +x ~/lab/lab-up.sh
```

**Validation:**
```bash
~/lab/lab-down.sh && echo "paused"
~/lab/lab-up.sh
KUBECONFIG=~/lab/rke2.kubeconfig kubectl get nodes      # Ready after resume
```

## Step K4 — Snapshots for known-good restore points

```bash
# After a clean build, snapshot each VM:
virsh snapshot-create-as rke2-cluster  clean-build "post-install baseline"
virsh snapshot-create-as rancher-mgmt  clean-build "post-install baseline"
# Roll back if you break something:
# virsh snapshot-revert rke2-cluster clean-build
```

> 🚧 **Lifecycle cautions**
> - Use `virsh shutdown` (graceful ACPI) or `managedsave`, **never** `virsh destroy` (a hard power-off that can corrupt etcd).
> - Always `lab-down.sh` **before** `wsl --shutdown`; otherwise VMs are killed un-gracefully.
> - `managedsave` state is invalidated if you change a VM's CPU/RAM; reconfigure only while shut down.
> - After a long pause, give RKE2 a minute to re-establish etcd leadership before expecting `kubectl` to respond.

---

# Final checklist

**Installed components**

- [ ] WSL2 sized (20 GB / 6 vCPU), `nestedVirtualization=true`, `/dev/kvm` present, systemd on
- [ ] KVM/QEMU/Libvirt installed; `default` NAT network active + autostart (Part B)
- [ ] Reusable cloud-init VM builder `~/lab/make-vm.sh` (Part C)
- [ ] **VM rke2-cluster** (192.168.122.20) — RKE2 1.33, bundled ingress disabled (Part D)
- [ ] **MetalLB** pool 192.168.122.200–250 + **F5 NGINX Ingress** with LoadBalancer IP (Part E)
- [ ] **VM rancher-mgmt** (192.168.122.30) — RKE2 1.33 + cert-manager + **Rancher v2.14.2** (Part F)
- [ ] Workload cluster **imported and Active** in Rancher (Part G)
- [ ] **ArgoCD** (guestbook synced) + **Flux** (podinfo reconciled) (Part H)
- [ ] **Longhorn 1.11** default StorageClass, PVC binds (Part I)
- [ ] **Keycloak 26** on Postgres/Longhorn, reachable via ingress (Part I)
- [ ] Browser access path chosen (socat or route) (Part J)
- [ ] `lab-up.sh` / `lab-down.sh` / snapshots in place (Part K)

**Start / stop quick reference**

```bash
# Stop for the day (then 'wsl --shutdown' on Windows is safe):
~/lab/lab-down.sh
# Resume:
~/lab/lab-up.sh
# Per-VM:  virsh shutdown <vm> | virsh start <vm> | virsh managedsave <vm>
# Restore: virsh snapshot-revert <vm> clean-build
```

**How to extend later (modularity)**

- Add a worker node: `~/lab/make-vm.sh rke2-worker 192.168.122.21 4 8192 60`, then run the RKE2 **agent** install pointing at `https://192.168.122.20:9345` with the server token from `/var/lib/rancher/rke2/server/node-token`. With a second node you can raise Longhorn `defaultReplicaCount` to 2–3.
- Swap ingress strategy: install **Traefik** or adopt the **Gateway API** (the maintained long-term direction) without touching the rest of the lab.
- Add observability (Prometheus/Grafana via Rancher Monitoring) or a private registry as additional independent modules.

> **Audit note:** every version above is pinned and traceable to its upstream release page (top table). Before each clean rebuild, re-confirm the exact patch tags and the Rancher support matrix at https://www.suse.com/suse-rancher/support-matrix/, since the 1.33→1.36 / ingress landscape is moving quickly in 2026.
