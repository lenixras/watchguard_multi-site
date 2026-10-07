# Réseau multi-site avec WatchGuard

> **Mise en place d'une architecture réseau sécurisée et interconnectée** — VPN IPsec inter-site, SD-WAN, filtrage sortant et journalisation centralisée sur un parc de sites reliés par Firebox / Fireware.

[![Status](https://img.shields.io/badge/status-d%C3%A9ploy%C3%A9-22c55e)](#aper%C3%A7u)
[![Fireware](https://img.shields.io/badge/Fireware-12.x-E4002B?style=flat-square)](https://www.watchguard.com/help/docs/fireware/12/en-US/)
[![VPN](https://img.shields.io/badge/VPN-IPsec%20%2F%20BOVPN-0ea5e9?style=flat-square)](#1--vpn-ipsec-inter-site)
[![SD-WAN](https://img.shields.io/badge/SD-WAN-QoS%20%2B%20load--balancing-f59e0b?style=flat-square)](#2--sd-wan-qos-load-balancing)
[![Filtrage](https://img.shields.io/badge/Filtrage-Firewall%20%2B%20WebBlocker-a855f7?style=flat-square)](#3--r%C3%A8gles-de-filtrage--webblocker)
[![Logs](https://img.shields.io/badge/Journalisation-Dimension-8b5cf6?style=flat-square)](#4--journalisation-dimension)
[![Stack](https://img.shields.io/badge/stack-WatchGuard%20%7C%20Fireware%20%7C%20BGP%20%7C%20syslog-1e293b?style=flat-square)](#stack--liens-utiles)

![Architecture réseau multi-site WatchGuard](assets/topologie.svg)

---

## Table des matières

1. [Aperçu](#aperçu)
2. [Objectifs & contraintes](#objectifs--contraintes)
3. [Architecture](#architecture)
4. [Fonctionnalités](#fonctionnalités)
   - [1 · VPN IPsec inter-site](#1--vpn-ipsec-inter-site)
   - [2 · SD-WAN (QoS, load-balancing)](#2--sd-wan-qos-load-balancing)
   - [3 · Règles de filtrage & WebBlocker](#3--règles-de-filtrage--webblocker)
   - [4 · Journalisation Dimension](#4--journalisation-dimension)
5. [Segmentation réseau](#segmentation-réseau)
6. [Résilience & tests de bascule](#résilience--tests-de-bascule)
7. [Méthode de déploiement](#méthode-de-déploiement)
8. [Résultats](#résultats)
9. [Compétences mobilisées](#compétences-mobilisées)
10. [Stack & liens utiles](#stack--liens-utiles)

---

## Aperçu

Parc de **5 sites** (siège, 2 agences, 1 site de production, population nomade) interconnectés en **maillage partiel hub-and-spoke** via des tunnels **IPsec BOVPN**, avec :

| Pilier | Réponse apportée |
|--------|------------------|
| **Connectivité** | Tunnels IPsec chiffrés entre tous les sites, routage dynamique (BGP) sur interfaces virtuelles |
| **Performance** | SD-WAN : choix de chemin selon latence / jitter / perte, équilibrage de charge et priorisation VoIP |
| **Contrôle** | Politiques de filtrage par flux, WebBlocker (catégories d'URL), Application Control, IPS/GAV |
| **Visibilité** | Tous les Firebox journalisent vers **WatchGuard Dimension** : rapports, alertes, preuves d'audit |

**En une phrase :** un utilisateur d'agence atteint les applications du siège *comme s'il y était*, avec un trafic contrôlé, priorisé et intégralement tracé.

---

## Objectifs & contraintes

- **Interconnecter** des sites hétérogènes (fibre, 4G de secours) sans MPLS coûteux.
- **Garantir la continuité** des applicatifs critiques (ERP, VoIP, messagerie) en cas de dégradation d'un lien.
- **Contrôler l'accès Internet** de tous les sites depuis un référentiel unique (pas de config locale divergente).
- **Centraliser les logs** pour l'exploitation, la sécurité et les obligations d'audit.
- **Administrer à distance** un maximum de sites (WatchGuard Cloud / WSM), interventions sur site minimales.

---

## Architecture

```mermaid
flowchart TB
    subgraph INT["Transit opérateur"]
        ISP1["ISP-1 · FTTH 1 Gb/s"]
        ISP2["ISP-2 · 4G (secours)"]
    end

    subgraph HQ["Siège — Firebox M390"]
        FW0["Fireware 12.x<br/>BGP · Policy Manager"]
        VLANs["VLAN 10 serveurs<br/>VLAN 20 utilisateurs<br/>VLAN 30 VoIP<br/>VLAN 40 IoT"]
        FW0 --- VLANs
    end

    subgraph BR1["Agence Lyon — T20"]
        FW1["Firebox T20"]
    end
    subgraph BR2["Agence Strasbourg — T20"]
        FW2["Firebox T20"]
    end
    subgraph BR3["Site production — M290"]
        FW3["Firebox M290"]
    end

    DIM["WatchGuard Dimension<br/>logs · rapports · alertes"]

    ISP1 --- FW0
    ISP2 -. "failover" .- FW0

    FW0 <==>|IPsec B1| FW1
    FW0 <==>|IPsec B2| FW2
    FW0 <==>|IPsec B3| FW3

    FW0 -. "syslog :4157" .-> DIM
    FW1 -. syslog .-> DIM
    FW2 -. syslog .-> DIM
    FW3 -. syslog .-> DIM

    classDef wg fill:#e4002b,stroke:#7f1024,color:#fff
    classDef fw fill:#0ea5e9,stroke:#0369a1,color:#fff
    classDef dim fill:#8b5cf6,stroke:#5b21b6,color:#fff
    class FW0 wg
    class FW1,FW2,FW3,ISP1,ISP2 fw
    class DIM dim
```

**Négociation du tunnel (IKEv2 / Phase 1 + Phase 2)** — ce qui se passe réellement à l'établissement d'un BOVPN :

```mermaid
sequenceDiagram
    participant A as Agence · Firebox T20
    participant I as Internet (NAT opérateur)
    participant S as Siège · Firebox M390

    Note over A,S: Phase 1 — IKE_SA_INIT
    A->>I: UDP 500/4500 · proposal AES-256-GCM / DH19 / SHA-256
    I->>S: translation NAT-T
    S-->>A: SA acceptée · PFS group 19

    Note over A,S: Phase 2 — CREATE_CHILD_SA (IPsec)
    A->>S: traffic selectors 10.20.10.0/24 ↔ 10.10.0.0/16
    S-->>A: IPsec SA établie · rekey automatique

    Note over A,S: Tunnel up — BGP (BOVPN virtual interface)
    A->>S: OPENKEEPALIVE + annonce routes site
    S-->>A: routes du siège apprises · ping de contrôle
```

**Répartition du trafic observée** (exemple de reporting Dimension sur 30 jours) :

```mermaid
pie showData
    title Bande passante moyenne par catégorie
    "Office 365 / Teams" : 34
    "Web (HTTPS)" : 22
    "ERP / applicatif inter-site" : 18
    "VoIP (SIP/RTP)" : 12
    "Sauvegardes / réplication" : 8
    "Autre / non classé" : 6
```

---

## Fonctionnalités

### 1 · VPN IPsec inter-site

Interconnexion **BOVPN** (*Branch Office VPN*) entre le siège et chaque site, en **mode routeur** avec *BOVPN virtual interfaces* pour supporter le routage dynamique.

| Paramètre | Choix retenu | Pourquoi |
|-----------|--------------|----------|
| Protocole | **IKEv2**, NAT-T (UDP 4500) | Re-négociation rapide, traversée NAT opérateur |
| Cipher | **AES-256-GCM** | Chiffrement + authentification en une passe (performances matérielles) |
| Intégrité | SHA-256 / DH group 19 (ECP-256) | Compromis sécurité / coût CPU Firebox |
| PFS | Activé (DH 19) | Clés de rekey isolées de la SA Phase 1 |
| Topologie | Hub-and-spoke, **virtual interfaces** | Permet BGP, ajout d'un site sans retoucher 4 configurations |
| Routage | **BGP** (ou routes statiques dépendantes de la PBR) | Basculer un site vers le 4G en annonçant une métrique moins bonne |
| Redondance | 2 tunnels/site + DPD (*Dead Peer Detection*) | Détection de coupure < 10 s, bascule automatique |
| Accès nomades | SSL VPN / IKEv2 client | Même référentiel de politiques que les sites |

**Contrôle d'admission** : chaque tunnel n'accepte que les sous-réseaux déclarés dans les *traffic selectors* — un site compromis ne peut pas explorer le reste du parc.

<details>
<summary><b>Extrait de politique Fireware — flux inter-site (BOVPN)</b></summary>

```text
Policy      : HQ-SRV-to-BR-ALL
From        : Any-External / BOVPN-VIF-*   →   To : Branch-Networks
Action      : Allow
NAT         : Dynamic (IP de l'interface sortante)
Scan        : Gateway AntiVirus ✔  IPS ✔  Application Control ✔
WebBlocker  : hors périmètre (trafic interne)
Timeout     : 3600 s (long-lived sessions ERP)
Logging     : Snmp Trap + Dimension
```

```text
Policy      : BR-ALL-to-HQ-SRV
From        : Branch-Networks   →   To : HQ-Servers (VLAN 10)
Action      : Allow — ports 443, 3389/RDP (jump host), 5432, 445
Refus       : toute autre destination → log + alerte
```

</details>

---

### 2 · SD-WAN (QoS, load-balancing)

Les politiques **SD-WAN** de Fireware décident, *à chaque paquet*, du lien et du tunnel à utiliser, en se basant sur la santé mesurée en continu (latence, jitter, perte de paquets) :

| Priorité | Trafic | Chemin principal | Chemin de secours | Déclencheur de bascule |
|----------|--------|------------------|-------------------|------------------------|
| 1 | VoIP / Teams (UDP audio) | WAN1 fibre | WAN2 4G | perte > 1 % **ou** jitter > 30 ms |
| 2 | ERP, messagerie | WAN1 fibre | WAN2 4G | latence > 150 ms pendant 5 s |
| 3 | Web / HTTPS | Load-balancing WAN1 + WAN2 | — | lien dégradé (réservation 4G) |
| 4 | Sauvegardes, mises à jour | WAN1 (fenêtre horaire) | — | — |

- **Load-balancing** : répartition par *round-robin* pondéré (1 Gb/s vs 4G) sur le trafic non critique.
- **QoS sortant** : files de priorité *high / normal / low* — la VoIP passe toujours en premier, les sauvegardes sont limiter en débit pour ne jamais saturer le lien.
- **Multilink / failover** : le WAN 4G reste en *cold standby* réservé (ou en partage de charge selon la règle), évitant la facturation 4G inutile.
- **Visibilité** : dans WatchGuard Cloud, latence/jitter/perte par site et par WAN, avec historique — ce qui alimente les décisions de renouvellement d'offre opérateur.

```mermaid
flowchart LR
    T["Paquet sortant"] --> D{"SD-WAN policy match ?"}
    D -->|"VoIP"| P1["File haute · WAN1<br/>repli WAN2 si jitter > 30 ms"]
    D -->|"ERP / mail"| P2["File normale · WAN1<br/>repli WAN2 si latence > 150 ms"]
    D -->|"Web"| P3["Load-balancing<br/>WAN1 + WAN2"]
    D -->|"Backup"| P4["File basse · WAN1<br/>limitée à 30 % du débit"]
    P1 --> O["Sortie site"]
    P2 --> O
    P3 --> O
    P4 --> O
```

---

### 3 · Règles de filtrage & WebBlocker

Un **référentiel unique** de politiques, poussé à tous les sites (WatchGuard Cloud / Policy Manager), en lecture « *défaut refus, autorisation explicite* » :

| Couche | Outil WatchGuard | Rôle |
|--------|------------------|------|
| L3/L4 | **Firewall policies** | Flux source/destination/port, zones (External, Internal, BOVPN, Wi-Fi) |
| L7 apps | **Application Control** | Bloque ou limite par application (torrent, streaming non autorisé, VPN non approuvés) |
| URL | **WebBlocker** | Catégories d'URL (malware, phishing, réseaux sociaux, jeux, adultes), liste personnalisée, *allow/deny by category* |
| DNS | **DNSWatch** | Réécriture des réponses DNS vers des nages propres, blocage des domaines malveillants |
| Menaces | **IPS + Gateway AntiVirus + APT Blocker** | Inspection des flux autorisés (SSL décrypté sur flux sortant autorisées) |
| Wi-Fi | **SSID invités isolés** | Captive portal, séparation du VLAN visiteurs |

**Exemple de profil WebBlocker « Corp-Standard »** poussé sur tous les sites :

| Catégorie | Décision | Commentaire |
|-----------|----------|-------------|
| Malware / Phishing / Command & control | **Deny (non modifiable)** | Protection impérative |
| Logiciels piratés, dropped domains | **Deny** | |
| Réseaux sociaux | **Deny en horaire ouvré** / allow le soir | Gain de productivité, sans frustration subie |
| Streaming vidéo | **Limité à 1,5 Mb/s** | La bande passante reste disponible pour l'activité |
| Cloud personnel (storage) | **Deny** | Exfiltration de données maîtrisée |
| Recherche, SaaS métier, presse | **Allow** | |

Chaque refus est **loggé** (utilisateur, FQDN, catégorie, site) → visible dans Dimension et exploitable en alerte.

<details>
<summary><b>Ordre de traitement appliqué sur un flux sortant</b></summary>

```text
1. Zone source / ACL interne          → refus si non conforme à la segmentation
2. IPS (signatures)                   → drop si match critique
3. Application Control                → allow / limit / drop par app
4. WebBlocker (catégorie d'URL)       → allow / deny selon profil utilisateur
5. Politique de pare-feu (dst, port)  → allow seulement si règle explicite
6. Entraînement vers Dimension        → log systématique du verdict
```

</details>

---

### 4 · Journalisation Dimension

**WatchGuard Dimension** (VM centralisée) consolide les logs de **tous** les Firebox — plus aucune donnée ne reste enfermée dans un site.

![Flux de journalisation multi-site vers WatchGuard Dimension](assets/flux-journalisation.svg)

| Élément | Détail |
|---------|--------|
| Collecte | Syslog **UDP 4157** (chiffré en TLS quand le trajet sort du LAN) |
| Sources | Traffic, WebBlocker, IPS/GAV, VPN (connexions et raisons de refus), DHCP, événements matériel |
| Rétention | 30 jours en ligne (au-delà : export/archivage externe) |
| Restitution | Tableaux de bord temps réel, **rapports PDF planifiés** (hebdo/mensuel), recherche plein texte |
| Alertes | E-mail, SNMP trap, webhook → WatchGuard Cloud / outil d'exploitation |
| Interop | Export **CEF / syslog** vers une SIEM externe pour corrélation SOC |
| Conformité | Rapports d'activité requis pour les audits (habilitation, traçabilité des accès) |

**Rapports exploités au quotidien**

1. **Top destinations / utilisateurs** — détecter un poste qui exfiltre, un flux anormal la nuit.
2. **WebBlocker — refus par catégorie** — piloter le réglage des politiques (et la formation des équipes).
3. **Menaces bloquées** — volume, origine, évolution après chaque campagne de phishing.
4. **Disponibilité des tunnels / liens** — prouver le respect du SLA et justifier les bascules 4G.
5. **Connexions VPN nomades** — qui s'est connecté, d'où, quand (obligation d'audit).

---

## Segmentation réseau

| VLAN | Périmètre | Politique dominante |
|------|-----------|---------------------|
| **10** | Serveurs / applications | Accès depuis les sites autorisés uniquement, journaux intensifs |
| **20** | Postes utilisateurs | Sortie Internet contrôlée + accès VLAN 10 (accès journalisé) |
| **30** | VoIP | QoS haute priorité, isolation L2, pas d'accès Internet direct |
| **40** | IoT / imprimantes / MES | Accès sortant minimal, jamais source vers VLAN 10 |
| **50** | Invités (Wi-Fi) | Internet seul, limité en débit, aucune route vers l'interne |

Règle transverse : **un VLAN n'atteint un autre VLAN que par une politique explicite** — la carte des flux est tenue à jour dans la documentation du dépôt.

---

## Résilience & tests de bascule

| Scénario | Comportement observé | Critère de succès |
|----------|----------------------|-------------------|
| Coupure fibre (ISP-1) | Bascule WAN2 4G, BGP ré-annonce avec métrique dégradée | Sessions critiques rétablies < 60 s |
| Mort d'un Firebox d'agence | DPD détecte, tunnel côté siège rejeté, alerte Dimension | Alerte reçue, coupure visible dans les rapports |
| Dégradation latence (SD-WAN) | Bascule VoIP/ERP vers le lien sain | Aucune coupure perceptible en réunion Teams |
| Panne Dimension | Les Firebox continuent de journaliser localement | Pas d'impact sur le trafic, resynchronisation ensuite |

Chaque scénario fait l'objet d'un **test planifié** (créneau de maintenance) et d'une **trace dans Dimension** : on ne prouve pas la résilience, on la mesure.

---

## Méthode de déploiement

```mermaid
timeline
    title Phases du projet
    phase 1 : Audit & relevés : Sites, liens, flux, VLANs existants
    phase 2 : Conception : Plan d'adressage, topologie VPN, profils de politiques
    phase 3 : Baie de test : Firebox de lab, validation IKEv2 / SD-WAN / WebBlocker
    phase 4 : Déploiement siège : M390 en production, Dimension, premiers tunnels
    phase 5 : Vague agences : T20/M290, reprise des politiques depuis Policy Manager
    phase 6 : Durcissement : WebBlocker, IPS, DNSWatch, rapports planifiés
    phase 7 : Recette & runbooks : Tests de bascule, procédures, transfert d'exploitation
```

**Bonnes pratiques appliquées**

- Configuration **versionnée** (export Policy Manager / sauvegardes `.xml` horodatées) avant toute modification.
- Déploiement **site par site**, en fenêtre de maintenance, avec retour arrière possible.
- Comptes d'administration **personnalisés + MFA**, jamais de compte partagé.
- Mises à jour Fireware planifiées (patch sécurité), testées en lab.
- *Zero-touch* quand c'est possible : mêmes profils, mêmes noms de politiques sur tous les sites.

---

## Résultats

<!-- Ajuste les chiffres ci-dessous à ta propre expérience -->
| Indicateur | Avant | Après |
|------------|-------|-------|
| Inter-sites | MPLS partiel, 1 lien unique par site | 5 sites en IPsec + redondance 4G automatique |
| Temps d'établissement d'un nouveau site | Provisionnement opérateur (semaines) | Firebox livrée + tunnel configuré en **< 1 j** |
| Contrôle Internet | Politiques locales divergentes | Référentiel **unique**, poussé à tous les sites |
| Visibilité | Logs sur chaque boîte, consultation rare | Dimension centralisée : rapports hebdo + alertes |
| Continuity VoIP | 1 seul lien, coupures perçues | Bascule SD-WAN transparente (jitter/perte) |
| Audit | Captures d'écran manuelles | Rapports Dimension exportables, rétention 30 j |

---

## Compétences mobilisées

![WatchGuard](https://img.shields.io/badge/WatchGuard-Firebox%20%2F%20Fireware%2012-E4002B?style=flat-square&logo=watchguard&logoColor=white)
![IPsec / IKEv2](https://img.shields.io/badge/IPsec-IKEv2%20%2F%20NAT--T-0ea5e9?style=flat-square)
![BGP](https://img.shields.io/badge/Routing-BGP%20%2F%20virtual%20interfaces-2563eb?style=flat-square)
![SD-WAN](https://img.shields.io/badge/SD-WAN-PBR%20%2F%20QoS%20%2F%20failover-f59e0b?style=flat-square)
![VLAN](https://img.shields.io/badge/L2-LAG%20%2F%20VLAN%20%2F%20trunk-10b981?style=flat-square)
![WebBlocker](https://img.shields.io/badge/WebBlocker-URL%20%2F%20DNS%20filtering-a855f7?style=flat-square)
![IPS/GAV](https://img.shields.io/badge/Threat%20mgmt-IPS%20%2F%20GAV%20%2F%20AppCtrl-ef4444?style=flat-square)
![Dimension](https://img.shields.io/badge/Dimension-Syslog%20%2F%20rapports%20%2F%20alertes-8b5cf6?style=flat-square)
![WatchGuard Cloud](https://img.shields.io/badge/WatchGuard%20Cloud-Gestion%20%C3%A0%20distance-334155?style=flat-square)
![MFA](https://img.shields.io/badge/Access-MFA%20%2F%20SSL%20VPN-64748b?style=flat-square)

---

## Stack & liens utiles

| Rôle | Solution |
|------|----------|
| Pare-feu / routeur | WatchGuard Firebox **M390** (siège), **M290** (production), **T20** (agences) |
| OS | **Fireware 12.x** |
| Interconnexion | IPsec **BOVPN** (virtual interfaces) + BGP |
| SD-WAN | Politiques SD-WAN Fireware + Multilink |
| Filtrage | WebBlocker, DNSWatch, Application Control, IPS, Gateway AntiVirus, APT Blocker |
| Gestion & logs | **WatchGuard Cloud** / WSM Policy Manager + **WatchGuard Dimension** |
| Nomades | SSL VPN / client IKEv2, MFA |

- Documentation officielle : <https://www.watchguard.com/help/docs/fireware/12/en-US/>
- Centre de connaissances : <https://www.watchguard.com/support>
- Dimension : <https://www.watchguard.com/wgrd-products/dimension>

---

*Projet réalisé dans le cadre de la constitution d'un portfolio infrastructure & sécurité réseau — architecture, mise en place, durcissement et exploitation d'un parc multi-site WatchGuard.*
