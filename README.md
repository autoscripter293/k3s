#!/bin/bash
# =============================================================================
# HomeLab K3s Cluster Setup – Bash-Dokumentation
# =============================================================================
# Dieses Dokument beschreibt die Installation von K3s auf einem Cluster aus
# 3 Head Nodes (Master) und 2 Compute Nodes (Worker) sowie die Einrichtung
# eines Cloudflare Tunnels für sicheren Zugriff ohne öffentliche IP.
# =============================================================================

# ------------------------------
# 1️⃣ Raspberry Pi vorbereiten
# ------------------------------

# 1. Flash Raspberry Pi OS Lite (64-bit)
# 2. SSH aktivieren
# 3. Hostnamen vergeben:
#    head1, head2, head3, node1, node2
# 4. System aktualisieren:
sudo apt update && sudo apt full-upgrade -y
sudo reboot

# ------------------------------
# 2️⃣ K3s Installation
# ------------------------------

# 2.1 Installation auf Primary Head Node (head1)
# curl Befehl lädt und installiert K3s automatisch
curl -sfL https://get.k3s.io | sh -
# Token für weitere Nodes abrufen
sudo cat /var/lib/rancher/k3s/server/node-token
# IP von head1 merken

# 2.2 Installation auf weiteren Head Nodes (head2, head3)
# Dies verbindet die Nodes zu einem HA etcd Cluster
curl -sfL https://get.k3s.io | K3S_URL=https://<HEAD1_IP>:6443 K3S_TOKEN=<TOKEN> sh -
# Kontrolle: kubectl get nodes

# 2.3 Installation auf Worker Nodes (node1, node2)
curl -sfL https://get.k3s.io | K3S_URL=https://<HEAD1_IP>:6443 K3S_TOKEN=<TOKEN> sh -
# Kontrolle: kubectl get nodes

# ------------------------------
# 3️⃣ Cloudflare Tunnel Setup
# ------------------------------

# 3.1 cloudflared installieren auf allen Head Nodes
sudo apt install cloudflared

# 3.2 Tunnel erstellen auf Primary Master (head1)
cloudflared tunnel create home-lab-tunnel
cloudflared tunnel route dns home-lab-tunnel ha.example.com

# 3.3 Tunnel-Konfiguration auf alle Head Nodes kopieren
# (z. B. /root/.cloudflared/<tunnel-config.json>)
# Auf head2 und head3:
cloudflared tunnel run home-lab-tunnel

# Optional als Systemservice starten:
sudo cloudflared service install

# 3.4 HA und Load Balancing
# Cloudflare verteilt Anfragen automatisch auf alle Tunnel-Endpunkte
# Traefik auf Head Nodes empfängt den Traffic und routet zu Services

# ------------------------------
# 4️⃣ Traefik & Ingress
# ------------------------------

# Traefik ist standardmäßig in K3s installiert
# Deployment + Service YAML für eigene Services vorbereiten
# Ingress YAML:
# - host: ha.example.com
# - path / → zu HomeAssistant oder anderen Services
# - TLS / HTTPS über Cloudflare oder Traefik ACME möglich

# ------------------------------
# 5️⃣ Kontrolle
# ------------------------------

# Knoten prüfen
kubectl get nodes
# Pods prüfen
kubectl get pods -A
# Ingress prüfen
kubectl get ingress -A

# =============================================================================
# Ende des Bash-Dokuments – alle Befehle dienen als Dokumentation / Anleitung
# =============================================================================
