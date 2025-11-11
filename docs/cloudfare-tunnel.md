# 4️⃣ Cloudflare Tunnel Setup
# ------------------------------

# 4.1 cloudflared installieren auf allen Head Nodes
sudo apt install cloudflared -y
cloudflared --version

# 4.2 Tunnel erstellen auf Primary Head Node
cloudflared tunnel create home-lab-tunnel
# Tunnel-Konfigurationsdatei wird unter /root/.cloudflared/<tunnel-id>.json erstellt

# 4.3 Subdomain zuordnen
cloudflared tunnel route dns home-lab-tunnel ha.example.com
# Cloudflare erstellt einen CNAME auf den Tunnel

# 4.4 Tunnel starten
# Manuell testen:
cloudflared tunnel run home-lab-tunnel
# Automatisch beim Boot:
sudo cloudflared service install

# Hinweis: Für HA -> Tunnel-Konfiguration auf head2 und head3 kopieren und starten
# Cloudflare verteilt Traffic automatisch auf alle Tunnel-Endpunkte

# ------------------------------
# 5️⃣ Traefik Integration
# ------------------------------
# Traefik läuft standardmäßig als Ingress Controller im K3s Cluster
# Beispiel Ingress für HomeAssistant:

cat <<EOF > homeassistant-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: homeassistant
  namespace: homeassistant
spec:
  rules:
    - host: ha.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: homeassistant
                port:
                  number: 8123
EOF

# - Traefik leitet Anfragen vom Tunnel an den Service
# - TLS/HTTPS kann direkt über Cloudflare laufen (optional Traefik ACME)

# ------------------------------
# 6️⃣ Kontrolle
# ------------------------------
kubectl get nodes
kubectl get pods -A
kubectl get ingress -A

# Zugriff testen:
curl https://ha.example.com

