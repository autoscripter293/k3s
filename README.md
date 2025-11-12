#!/bin/bash
# =============================================================================
# HomeLab K3s Cluster Setup – Vollständiges Bash-Dokument
# Mit VLAN10/20/30, Cloudflare Managed Tunnel + GitOps Deployment
# =============================================================================

# =============================================================================
# 1️⃣ Raspberry Pi OS vorbereiten
# =============================================================================
# 64-bit Raspberry Pi OS Lite flashen
# SSH aktivieren, Hostnames vergeben: master1, master2, master3, worker1, worker2

sudo apt update && sudo apt full-upgrade -y
sudo reboot

# =============================================================================
# 2️⃣ VLAN-Setup auf Raspberry Pi (Master + Worker)
# =============================================================================
# VLAN10 = Management/Admin
# VLAN20 = Cluster/K3s Nodes
# VLAN30 = Services / Guest / IoT

# --- MASTER (head1) ---
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip addr add 10.0.10.1/24 dev eth0.10
sudo ip link set dev eth0.10 up

sudo ip link add link eth0 name eth0.20 type vlan id 20
sudo ip addr add 10.0.20.1/24 dev eth0.20
sudo ip link set dev eth0.20 up

sudo ip link add link eth0 name eth0.30 type vlan id 30
sudo ip addr add 10.0.30.1/24 dev eth0.30
sudo ip link set dev eth0.30 up

# --- WORKER (node1) ---
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip addr add 10.0.10.2/24 dev eth0.10
sudo ip link set dev eth0.10 up

sudo ip link add link eth0 name eth0.20 type vlan id 20
sudo ip addr add 10.0.20.2/24 dev eth0.20
sudo ip link set dev eth0.20 up

sudo ip link add link eth0 name eth0.30 type vlan id 30
sudo ip addr add 10.0.30.2/24 dev eth0.30
sudo ip link set dev eth0.30 up

# --- VLAN-Verbindung testen ---
echo "Ping Master -> Worker VLAN10:"
ping -c 3 10.0.10.2

echo "Ping Master -> Worker VLAN20:"
ping -c 3 10.0.20.2

echo "Ping Master -> Worker VLAN30:"
ping -c 3 10.0.30.2

# =============================================================================
# 3️⃣ K3s Installation (HA Cluster mit etcd)
# =============================================================================

# -------------------------
# 3.1 Primary Master (head1)
# -------------------------
curl -sfL https://get.k3s.io | sh -
sudo cat /var/lib/rancher/k3s/server/node-token
# IP von head1 merken (z. B. 10.0.20.1 für Cluster VLAN)

# -------------------------
# 3.2 Weitere Master (head2, head3)
# -------------------------

hostname -I        #Ip rausfinden

curl -sfL https://get.k3s.io | \
  K3S_URL=https://10.0.20.1:6443 \
  K3S_TOKEN=<TOKEN> \
  sh -

# -------------------------
# 3.3 Worker Nodes (node1, node2)
# -------------------------
curl -sfL https://get.k3s.io | \
  K3S_URL=https://10.0.20.1:6443 \
  K3S_TOKEN=<TOKEN> \
  sh -

# Kontrolle
sudo kubectl get nodes -A

# =============================================================================
# 4️⃣ Cloudflare Tunnel – Service Token (Managed Mode)
# =============================================================================
# KEINE cert.pem, KEINE JSON Credentials – alles über Dashboard.

# -------------------------
# 4.1 cloudflared installieren (arm64)
# -------------------------
cd /tmp
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
sudo apt install ./cloudflared-linux-arm64.deb -y
cloudflared --version

# -------------------------
# 4.2 Tunnel im Cloudflare Dashboard anlegen
# -------------------------
# Cloudflare Zero Trust → Access → Tunnels → Create Tunnel
# → "Cloudflared" wählen → Managed Mode
# → cloudflared service install <SERVICE_TOKEN>

# -------------------------
# 4.3 Service Token auf jedem head-node ausführen
# -------------------------
sudo cloudflared service install <SERVICE_TOKEN>
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared

# =============================================================================
# 5️⃣ Cloudflare Hostnames (Subdomains) konfigurieren – Dashboard
# =============================================================================
# Beispiele:
# web.example.com      → HTTP → http://localhost:80
# ssh.example.com      → SSH  → ssh://localhost:22
# notes.example.com    → HTTP → http://localhost:80

# =============================================================================
# 6️⃣ Traefik & Ingress (K3s integriert)
# =============================================================================
# Traefik kommt vorinstalliert, EntryPoints: :80 und :443
kubectl get pods -n kube-system
kubectl get ingress -A

# =============================================================================
# 7️⃣ GitHub GitOps Workflow
# =============================================================================
cd ~
git clone git@github.com:autoscripter293/k3s.git
cd ~/k3s

# Neue YAMLs hinzufügen
git add projects/web/*
git commit -m "Add web project YAMLs"
git push origin main

# Deployment ins Cluster übernehmen
git pull
sudo kubectl apply -f projects/web/

# QUICK & DIRTY Deployment
git add projects/privatebin/*
git commit -m "Add PrivateBin deployment + ingress"
git push origin main
git pull
sudo kubectl apply -f projects/privatebin/

# =============================================================================
# 8️⃣ Kontrolle
# =============================================================================
sudo kubectl get nodes
sudo kubectl get pods -A
sudo kubectl get ingress -A

# =============================================================================
# Ende des Bash-Dokuments
# =============================================================================
