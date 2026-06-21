# Captures réseau

Ce dossier contient les captures Wireshark réalisées pendant les tests de l'architecture 5G.

## Captures PFCP (interface N4 — UDP 8805)

| Fichier | Description |
|---------|-------------|
| `pfcp-heartbeat.pcapng` | Échanges Heartbeat Request/Response entre SMF et UPF |
| `pfcp-session-establishment.pcapng` | Création de session PDU (type 50/51) |
| `pfcp-session-modification.pcapng` | Mise à jour FAR pour tunnel GTP-U N3 (type 52) |
| `pfcp-session-deletion.pcapng` | Suppression de session (type 54/55) |
| `pfcp-flood-test.pcapng` | Flood de Heartbeat Requests (test théorique) |

## Commandes de capture

```bash
# Capturer tout le trafic PFCP
sudo tcpdump -ni ens37 -w pfcp-capture.pcapng udp port 8805

# Capturer tout le trafic GTP-U
sudo tcpdump -ni ens37 -w gtpu-capture.pcapng udp port 2152

# Capturer la séquence complète (PFCP + GTP-U + NGAP)
sudo tcpdump -ni ens37 -w full-session.pcapng \
  'udp port 8805 or udp port 2152 or sctp port 38412'
```

## Filtres Wireshark utiles

```
# Afficher uniquement PFCP
pfcp

# Session Establishment uniquement
pfcp.message_type == 50 or pfcp.message_type == 51

# Heartbeats uniquement
pfcp.message_type == 1 or pfcp.message_type == 2

# Trafic GTP-U
gtp
```

## Description des messages PFCP observés

### PFCP Session Establishment Request (type 50)
- Envoyé par le SMF vers l'UPF
- Contient : Node ID, CP F-SEID, PDR (Packet Detection Rules), FAR (Forwarding Action Rules), QER
- SEID = 0 dans l'en-tête (session non encore créée côté UPF)

### PFCP Session Establishment Response (type 51)
- Envoyé par l'UPF vers le SMF
- Contient : Cause = Request accepted, F-SEID de l'UPF, Created PDR
- Confirme l'installation des règles

### PFCP Session Modification Request (type 52)
- Envoyé après la création pour mettre à jour la FAR
- Ajoute les paramètres GTP-U : Outer Header Creation, N3 3GPP Access
- Permet à l'UPF de transférer les paquets vers le gNB

### PFCP Heartbeat Request/Response (type 1/2)
- Échanges périodiques entre SMF et UPF
- Vérifient que les deux fonctions restent joignables
- Observables avant même la connexion du gNB
