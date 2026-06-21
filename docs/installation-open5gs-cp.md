# Installation et configuration d'Open5GS CP

**VM2 — 192.168.200.128**

## Objectif

Mettre en place le plan de contrôle 5G : AMF, SMF, NRF, AUSF, UDM, UDR, PCF, NSSF et MongoDB.

## 1. Vérification réseau

```bash
hostname
ip -br address
# Vérifier : 192.168.200.128/24 sur ens37

ping -c 2 192.168.200.129   # VM UPF
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

# Vérifier les exécutables
ls -lh ~/open5gs-2.7.7/install/bin/open5gs-*
```

> ⚠️ Les fichiers de config à modifier se trouvent dans `~/open5gs-2.7.7/install/etc/open5gs/`  
> Ne pas modifier ceux dans `~/open5gs-2.7.7/build/configs/open5gs/`

## 4. Installation de MongoDB

```bash
sudo apt install -y gnupg curl

curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  sudo gpg --yes -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg arch=amd64 ] \
https://repo.mongodb.org/apt/debian bookworm/mongodb-org/7.0 main" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

sudo apt update && sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod

# Vérification
systemctl is-active mongod
sudo ss -lntp | grep 27017
```

## 5. Application des configurations

```bash
# Copier les fichiers de config depuis ce dépôt
cp config/cp/nrf.yaml ~/open5gs-2.7.7/install/etc/open5gs/
cp config/cp/amf.yaml ~/open5gs-2.7.7/install/etc/open5gs/
cp config/cp/smf.yaml ~/open5gs-2.7.7/install/etc/open5gs/
```

## 6. Ajout de l'abonné via WebUI

```bash
# Installer Node.js
sudo apt install -y nodejs npm

cd ~/open5gs-2.7.7/webui
npm install
npm run dev
```

Ouvrir `http://localhost:9999` — identifiants : `admin / 1423`

Créer un abonné avec :
- IMSI : `999700000000001`
- Clé K : `465B5CE8B199B49FAA5F0A2EE238A6BC`
- OPc : `E8ED289DEBA952E4283B54E88E6183CA`
- AMF : `8000`
- SST : `1` / DNN : `internet`

## 7. Démarrage des fonctions CP

Ouvrir un terminal par fonction, **dans cet ordre** :

```bash
cd ~/open5gs-2.7.7

# 1. NRF — annuaire (obligatoirement en premier)
./install/bin/open5gs-nrfd  -c ./install/etc/open5gs/nrf.yaml

# 2. UDR
./install/bin/open5gs-udrd  -c ./install/etc/open5gs/udr.yaml

# 3. UDM
./install/bin/open5gs-udmd  -c ./install/etc/open5gs/udm.yaml

# 4. AUSF
./install/bin/open5gs-ausfd -c ./install/etc/open5gs/ausf.yaml

# 5. PCF
./install/bin/open5gs-pcfd  -c ./install/etc/open5gs/pcf.yaml

# 6. NSSF
./install/bin/open5gs-nssfd -c ./install/etc/open5gs/nssf.yaml

# 7. SMF
./install/bin/open5gs-smfd  -c ./install/etc/open5gs/smf.yaml

# 8. AMF
./install/bin/open5gs-amfd  -c ./install/etc/open5gs/amf.yaml
```

## 8. Vérification

```bash
# Vérifier que l'AMF écoute sur SCTP 38412
sudo ss -lpnA sctp | grep 38412

# Vérifier que le SMF écoute sur PFCP 8805
sudo ss -lunp | grep 8805
```
