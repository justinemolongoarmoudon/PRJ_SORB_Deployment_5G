# PRJ_SORB_Deployment_5G
# Déploiement d'une architecture 5G et analyse du protocole PFCP

> Projet réalisé dans le cadre du BUT Réseaux & Télécommunications — Parcours Réseaux Opérateurs & Multimédias  
> IUT Villetaneuse — Université Sorbonne Paris Nord  
> Encadré par M. OUAMRI

---

## Présentation

Ce projet déploie une architecture 5G simplifiée en environnement virtualisé, et analyse le protocole **PFCP** (Packet Forwarding Control Protocol) sur l'interface **N4** entre le SMF et l'UPF.

L'architecture repose sur trois machines virtuelles :

| VM | Rôle | Adresse IP |
|----|------|-----------|
| **VM1 — UERANSIM** | Simulation UE + gNB | 192.168.200.130 |
| **VM2 — Open5GS CP** | Plan de contrôle (AMF, SMF, NRF, AUSF, UDM, UDR, PCF, NSSF) | 192.168.200.128 |
| **VM3 — Open5GS UPF** | Plan utilisateur (UPF) | 192.168.200.129 |

---

## Architecture

```
UE (UERANSIM)
    │  RRC/NAS simulé
    ▼
gNB (UERANSIM) ──N2 (NGAP/SCTP:38412)──► AMF (Open5GS CP)
    │                                          │
    │  N3 (GTP-U/UDP:2152)              SMF ◄─┘
    ▼                                    │ N4 (PFCP/UDP:8805)
   UPF (Open5GS UP) ◄────────────────────┘
    │
    ▼
 ogstun (10.45.0.1/16) → Internet
```

---

## Composants principaux

| Composant | Rôle |
|-----------|------|
| **UE** | Terminal utilisateur simulé (IMSI, clés d'auth) |
| **gNB** | Station de base 5G — relaie la signalisation vers l'AMF |
| **AMF** | Enregistrement, mobilité, signalisation NAS |
| **SMF** | Gestion des sessions PDU, programmation de l'UPF via PFCP |
| **UPF** | Transport des données utilisateur (GTP-U ↔ ogstun) |
| **NRF** | Annuaire des fonctions réseau |
| **AUSF / UDM / UDR** | Authentification et gestion des abonnés |
| **PCF** | Politiques de session (QoS) |
| **NSSF** | Sélection du slice (SST=1) |
| **MongoDB** | Stockage des profils abonnés |

---

## Interfaces et protocoles

| Interface | Communication | Protocole | Port |
|-----------|--------------|-----------|------|
| N1 | UE ↔ AMF | NAS | — |
| N2 | gNB ↔ AMF | NGAP / SCTP | 38412 |
| N3 | gNB ↔ UPF | GTP-U / UDP | 2152 |
| N4 | SMF ↔ UPF | PFCP / UDP | 8805 |
| SBI | Fonctions CP | HTTP/2 | — |

---

## Paramètres réseau

```
MCC  = 999
MNC  = 70
TAC  = 1
SST  = 1
DNN  = internet
IMSI = 999700000000001

Réseau UE : 10.45.0.0/16
Gateway UPF (ogstun) : 10.45.0.1
Adresse attribuée au UE : 10.45.0.3
```

---

## Structure du dépôt

```
5g-deployment/
├── README.md
├── docs/
│   ├── installation-ueransim.md
│   ├── installation-open5gs-cp.md
│   └── installation-open5gs-upf.md
├── config/
│   ├── gnb/
│   │   └── open5gs-gnb.yaml
│   ├── ue/
│   │   └── open5gs-ue.yaml
│   ├── cp/
│   │   ├── nrf.yaml
│   │   ├── amf.yaml
│   │   └── smf.yaml
│   └── upf/
│       └── upf.yaml
└── captures/
    └── README-captures.md
```

---

## Démarrage rapide

Consulter les guides détaillés dans le dossier (docs/) :

1. [Installation UERANSIM](/docs/installation-ueransim.md)
2. [Installation Open5GS CP](/docs/installation-open5gs-cp.md)
3. [Installation Open5GS UPF](/docs/installation-open5gs-upf.md)

### Ordre de démarrage

```bash
# 1. MongoDB (sur VM CP)
sudo systemctl start mongod

# 2. Fonctions du plan de contrôle (sur VM CP) — dans l'ordre
./install/bin/open5gs-nrfd   -c ./install/etc/open5gs/nrf.yaml
./install/bin/open5gs-udrd   -c ./install/etc/open5gs/udr.yaml
./install/bin/open5gs-udmd   -c ./install/etc/open5gs/udm.yaml
./install/bin/open5gs-ausfd  -c ./install/etc/open5gs/ausf.yaml
./install/bin/open5gs-pcfd   -c ./install/etc/open5gs/pcf.yaml
./install/bin/open5gs-nssfd  -c ./install/etc/open5gs/nssf.yaml
./install/bin/open5gs-smfd   -c ./install/etc/open5gs/smf.yaml
./install/bin/open5gs-amfd   -c ./install/etc/open5gs/amf.yaml

# 3. UPF (sur VM UPF)
sudo ip tuntap add name ogstun mode tun
sudo ip addr replace 10.45.0.1/16 dev ogstun
sudo ip link set ogstun up
sudo ./install/bin/open5gs-upfd -c ./install/etc/open5gs/upf.yaml

# 4. gNB (sur VM UERANSIM)
sudo ./build/nr-gnb -c config/open5gs-gnb.yaml

# 5. UE (sur VM UERANSIM)
sudo ./build/nr-ue -c config/open5gs-ue.yaml
```

---

## Validation

Après démarrage complet, vérifier la connectivité du UE :

```bash
# Vérifier l'interface TUN créée
ip -br address show uesimtun0
# → uesimtun0  UNKNOWN  10.45.0.3/24

# Ping vers l'UPF via le tunnel 5G
ping -I uesimtun0 -c 4 10.45.0.1
# → 0% packet loss ✓

# Observer le trafic PFCP
sudo tcpdump -ni ens37 udp port 8805
```

---

## Analyse PFCP

Les captures Wireshark disponibles dans [`captures/`](captures/) illustrent :

- **PFCP Heartbeat Request/Response** — maintien de la liaison SMF ↔ UPF
- **PFCP Session Establishment Request (type 50)** — création de session avec PDR/FAR/QER
- **PFCP Session Establishment Response (type 51)** — acceptation par l'UPF, retour du F-SEID
- **PFCP Session Modification Request (type 52)** — mise à jour FAR pour tunnel GTP-U N3
- **PFCP Session Deletion Request** — suppression de session

---

## Scénarios d'attaque étudiés (théoriques)

| Attaque | Description | Impact potentiel |
|---------|-------------|-----------------|
| PFCP Flood | Envoi massif de Heartbeat Requests | Surcharge CPU du SMF/UPF |
| Paquets malformés | Messages PFCP avec IEs incorrects | Erreurs dans les journaux, plantage |
| Session Hijacking | Abus de gestion de session | Perturbation des sessions UE |

> ⚠️ Ces scénarios n'ont pas été reproduits en production. Étude théorique uniquement.

---

## Versions utilisées

| Outil | Version |
|-------|---------|
| Open5GS | 2.7.7 |
| UERANSIM | 3.2.8 |
| MongoDB | 7.0 |
| OS | Ubuntu 24 (Debian Bookworm) |

---

## Références

- [UERANSIM — Configuration](https://github.com/aligungr/UERANSIM/wiki/Configuration)
- [UERANSIM — Installation](https://github.com/aligungr/UERANSIM/wiki/Installation)
- [Open5GS — Quickstart](https://github.com/open5gs/open5gs/blob/main/docs/_docs/guide/01-quickstart.md)
- [Open5GS — GitHub](https://github.com/open5gs/open5gs)

---

## Auteurs

- GUEYE Mame Bousso
- MOLONGO–ARMOUDON Justine
- GUEYE Codou
- LE PABIC Ronan
- MOUTIER Amaury
