# Installation et configuration d'Open5GS UPF

**VM3 — 192.168.200.129**

## Objectif

Mettre en place le plan utilisateur : transport des données entre le gNB et le réseau de données.

## 1. Vérification réseau

```bash
hostname
ip -br address
# Vérifier : 192.168.200.129/24 sur ens37

ping -c 2 192.168.200.128   # VM CP
ping -c 2 192.168.200.130   # VM UERANSIM
```

## 2. Installation des dépendances

```bash
sudo apt update
sudo apt install -y python3-pip python3-setuptools python3-wheel \
  ninja-build build-essential flex bison git cmake libsctp-dev \
  libgnutls28-dev libgcrypt-dev libssl-dev libidn11-dev libmongoc-dev \
  libbson-dev libyaml-dev libnghttp2-dev libmicrohttpd-dev \
  libcurl4-gnutls-dev libtins-dev libtalloc-dev meson
```

## 3. Compilation d'Open5GS

```bash
cd ~/open5gs-2.7.7
meson setup build --prefix="$(pwd)/install"
ninja -C build
ninja -C build install

ls -lh ~/open5gs-2.7.7/install/bin/open5gs-upfd
```

## 4. Application de la configuration

```bash
cp config/upf/upf.yaml ~/open5gs-2.7.7/install/etc/open5gs/
```

## 5. Création de l'interface ogstun

```bash
# Charger le module TUN
sudo modprobe tun
ls -l /dev/net/tun

# Créer l'interface
sudo ip tuntap add name ogstun mode tun

# Configurer l'adresse passerelle
sudo ip addr replace 10.45.0.1/16 dev ogstun

# Activer l'interface
sudo ip link set ogstun up

# Vérification
ip -br address show ogstun
# → ogstun  UNKNOWN  10.45.0.1/16  (UNKNOWN est normal pour une interface TUN)

ip route | grep 10.45
# → 10.45.0.0/16 dev ogstun
```

## 6. Activation du transfert IPv4

```bash
sudo sysctl -w net.ipv4.ip_forward=1

# Rendre persistant
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-open5gs-upf.conf
sudo sysctl --system

# Vérifier
sysctl net.ipv4.ip_forward
# → net.ipv4.ip_forward = 1
```

## 7. Démarrage de l'UPF

```bash
cd ~/open5gs-2.7.7
sudo ./install/bin/open5gs-upfd -c ./install/etc/open5gs/upf.yaml
```

## 8. Vérification des ports

```bash
sudo ss -lunp | grep -E ':8805|:2152'
# → 192.168.200.129:8805  (PFCP)
# → 192.168.200.129:2152  (GTP-U)
```

## 9. Vérification de l'association PFCP

```bash
# Observer les Heartbeats entre SMF et UPF
sudo tcpdump -ni ens37 udp port 8805
# → Échanges entre 192.168.200.128:8805 et 192.168.200.129:8805
```
