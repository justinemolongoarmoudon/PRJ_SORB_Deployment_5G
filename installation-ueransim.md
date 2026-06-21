# Installation et configuration d'UERANSIM

**VM1 — 192.168.200.130**

## Objectif

Mettre en place la simulation de l'équipement utilisateur (UE) et de la station de base 5G (gNB).

## 1. Vérification réseau

```bash
hostname
ip -br address
# Vérifier : 192.168.200.130/24 sur ens37

ping -c 2 192.168.200.128   # VM CP
ping -c 2 192.168.200.129   # VM UPF
```

## 2. Installation des dépendances

```bash
sudo apt update && sudo apt install -y \
  cmake make gcc g++ libsctp-dev lksctp-tools iproute2 git
```

## 3. Compilation d'UERANSIM

```bash
cd /opt
sudo git clone https://github.com/aligungr/UERANSIM
cd UERANSIM

sudo cmake -B build
sudo cmake --build build

# Vérifier les exécutables
ls -lh /opt/UERANSIM/build/nr-gnb
ls -lh /opt/UERANSIM/build/nr-ue
```

## 4. Configuration

Copier les fichiers de config depuis ce dépôt :

```bash
sudo cp config/gnb/open5gs-gnb.yaml /opt/UERANSIM/config/
sudo cp config/ue/open5gs-ue.yaml /opt/UERANSIM/config/
```

## 5. Vérification de l'interface TUN

```bash
ls -l /dev/net/tun

# Si le fichier rt_tables est absent :
sudo mkdir -p /etc/iproute2
sudo tee /etc/iproute2/rt_tables >/dev/null <<'EOF'
255 local
254 main
253 default
0 unspec
EOF
```

## 6. Démarrage

```bash
# Terminal 1 — gNB
cd /opt/UERANSIM
sudo ./build/nr-gnb -c config/open5gs-gnb.yaml
# Attendre : "NG Setup procedure is successful"

# Terminal 2 — UE
sudo ./build/nr-ue -c config/open5gs-ue.yaml
```

## 7. Validation

```bash
ip -br address show uesimtun0
# → uesimtun0  UNKNOWN  10.45.0.3/24

ping -I uesimtun0 -c 4 10.45.0.1
# → 0% packet loss ✓
```
