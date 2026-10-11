# Application : Configuration des VLANs, Trunks et Routage inter-VLAN

Ce document détaille la mise en place de VLANs sur des commutateurs Cisco, la configuration de liens trunk (802.1q), la création d'interfaces SVI pour l'administration, ainsi que deux méthodes de routage inter-VLAN (Router on a stick et commutateur couche 3).

## Sommaire

1. [Rappels sur le fonctionnement d'un switch](#1-rappels-sur-le-fonctionnement-dun-switch)
2. [Introduction aux VLANs](#2-introduction-aux-vlans)
3. [Configuration des VLANs sur commutateur](#3-configuration-des-vlans-sur-commutateur)
4. [Notion de Trunk et de VLAN natif](#4-notion-de-trunk-et-de-vlan-natif)
5. [Routage inter-VLAN](#5-routage-inter-vlan)
6. [Mise en place de l'accès SSH](#6-mise-en-place-de-laccès-ssh)
7. [Sauvegarde et restauration (FTP)](#7-sauvegarde-et-restauration-ftp)
8. [Vérifications et dépannage](#8-vérifications-et-dépannage)
9. [Questions d'application](#9-questions-dapplication)

---

## 1. Rappels sur le fonctionnement d'un switch

Un switch (couche 2) commute les trames en fonction des adresses MAC de destination grâce à la table CAM.

### Table CAM d'un switch

```text
SW-02#show mac address-table
          Mac Address Table

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0005.5ed4.5bd1    DYNAMIC     Fa0/24
   1    00d0.ff85.4444    DYNAMIC     Fa0/1
   1    00d0.ffd0.d77b    DYNAMIC     Fa0/23
   1    00e0.8fbe.7bc2    DYNAMIC     Fa0/22
   1    00e0.a340.a418    DYNAMIC     Fa0/24
   1    00e0.f946.7d33    DYNAMIC     Fa0/24
```

### Problématiques des réseaux plats

- Comment réduire les domaines de diffusion sur un réseau ?
- Comment séparer les flux entre différentes entités/groupes de travail ?

Limites de la segmentation physique :

- Architecture fortement dépendante du câblage
- Problèmes d'évolutivité
- Coûts d'infrastructure et maintenance élevés

---

## 2. Introduction aux VLANs

Un VLAN (Virtual LAN) est une segmentation logique d'un réseau physique en plusieurs réseaux virtuels.

**Objectifs des VLANs :**

- Réduire les domaines de diffusion (broadcast)
- Améliorer les performances
- Administration simplifiée
- Sécurité et confidentialité par séparation des flux

**Typologie des VLANs :**

| Type | Principe | Avantages | Inconvénients |
|---|---|---|---|
| Par port (niveau 1) | Association ports physiques → VLAN | Simple à mettre en œuvre | Peu souple (déplacement de machine) |
| Par adresse MAC (niveau 2) | Association MAC → VLAN | Mobilité possible | Lourde à administrer |
| Par protocole (niveau 3) | Association sous-réseau IP → VLAN | Flexible, dynamique | Problèmes de sécurité |

---

## 3. Configuration des VLANs sur commutateur

### Schéma du réseau et plan d'adressage

| VLAN | VLAN name | Switch ports | Réseau |
|---|---|---|---|
| 10 | etudiant | 1-8 | 192.168.10.0/24 |
| 20 | enseignant | 9-16 | 192.168.20.0/24 |
| 30 | administration | 17-23 | 192.168.30.0/24 |
| 100 | native | - | - |
| 101 | management | 24 | 192.168.101.0/24 |

### Interfaces SVI pour l'administration

- Switch 0 (SW1) : interface Vlan 101 - `192.168.101.254`
- Switch 1 (SW2) : interface Vlan 101 - `192.168.101.253`

### Adressage des machines

| Machine | Adresse IP |
|---|---|
| PC0 | 192.168.10.1/24 |
| PC3 | 192.168.10.2/24 |
| PC1 | 192.168.20.1/24 |
| PC4 | 192.168.20.2/24 |
| PC2 | 192.168.30.1/24 |
| PC5 | 192.168.30.2/24 |
| PC6 (management) | 192.168.101.1/24 |

### Configuration complète du Switch 0 (SW1)

```text
enable
configure terminal

# Nom du switch
hostname SW1

# Création des VLANs
vlan 10
 name etudiant
vlan 20
 name enseignant
vlan 30
 name administration
vlan 101
 name management
vlan 100
 name native

# Affectation des ports au VLAN 10 (ports 1 à 8)
interface range f0/1-8
 switchport mode access
 switchport access vlan 10

# Affectation des ports au VLAN 20 (ports 9 à 16)
interface range f0/9-16
 switchport mode access
 switchport access vlan 20

# Affectation des ports au VLAN 30 (ports 17 à 23)
interface range f0/17-23
 switchport mode access
 switchport access vlan 30

# Port de management (port 24) au VLAN 101
interface f0/24
 switchport mode access
 switchport access vlan 101

# Interface SVI pour administration sur VLAN 101
interface vlan 101
 ip address 192.168.101.254 255.255.255.0

# Configuration du trunk sur G0/1
interface g0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,100,101
 switchport trunk native vlan 100

# Fin de configuration
end
show vlan brief
show interfaces trunk
```

> [!NOTE]
> Les lignes commençant par `#` servent de commentaires dans ce document. Dans l'IOS Cisco, les commentaires s'écrivent avec `!`.

### Configuration du Switch 1 (SW2)

Adapter la configuration de SW1 pour SW2 en modifiant :

- Le nom d'hôte : `SW2`
- L'adresse de l'interface SVI vlan 101 : `192.168.101.253`

---

## 4. Notion de Trunk et de VLAN natif

Un trunk (802.1q) permet de faire circuler plusieurs VLANs sur un même lien physique.

**Format de la trame 802.1q :**

| Préambule | MAC Destination | MAC Source | TPID (0x8100) | TCI | Type/Length | Données | FCS |
|---|---|---|---|---|---|---|---|

**TCI (Tag Control Information) :**

- Priority (3 bits) : priorité 802.1p (0 à 7)
- DEI (1 bit) : Drop Eligible Indicator
- VID (12 bits) : VLAN ID (1 à 4094)

**Bonnes pratiques Cisco :**

- Éteindre tous les ports non utilisés d'un switch
- Changer tous les ports du VLAN 1 vers un autre VLAN
- Utiliser un VLAN particulier pour la gestion (management)
- Ne pas utiliser le VLAN 1 par défaut comme VLAN de gestion
- Utiliser SSH plutôt que Telnet
- Désactiver DTP, VTP et CDP

### Désactivation des protocoles indésirables

```text
# Désactivation de DTP (Dynamic Trunking Protocol)
(config-if)# switchport nonegotiate

# Désactivation de VTP (VLAN Trunk Protocol)
(config)# vtp mode transparent

# Désactivation de CDP (Cisco Discovery Protocol) - globale
(config)# no cdp run

# Désactivation de CDP - par interface
(config)# interface fa0/1
(config-if)# no cdp enable
```

---

## 5. Routage inter-VLAN

### Solution 1 : Router on a stick

**Principe :** un routeur relié par un seul lien trunk vers le switch, avec des sous-interfaces virtuelles.

**Schéma des réseaux et passerelles :**

| VLAN | VLAN name | Réseau | Passerelle |
|---|---|---|---|
| 10 | etudiant | 192.168.10.0/24 | 192.168.10.254 |
| 20 | enseignant | 192.168.20.0/24 | 192.168.20.254 |
| 30 | administration | 192.168.30.0/24 | 192.168.30.254 |
| 101 | supervision | 192.168.101.0/24 | - |

#### Configuration du routeur (sous-interfaces)

```text
enable
configure terminal

# Configuration de l'interface physique
interface g0/0/0
 no shutdown

# Sous-interface pour VLAN 10
interface g0/0/0.10
 encapsulation dot1q 10
 ip address 192.168.10.254 255.255.255.0

# Sous-interface pour VLAN 20
interface g0/0/0.20
 encapsulation dot1q 20
 ip address 192.168.20.254 255.255.255.0

# Sous-interface pour VLAN 30
interface g0/0/0.30
 encapsulation dot1q 30
 ip address 192.168.30.254 255.255.255.0

# Vérification
show interfaces
```

### Solution 2 : Commutateur couche 3

**Principe :** un commutateur couche 3 assure directement le routage entre les VLANs.

#### Configuration switch couche 3

```text
enable
configure terminal

# Activation du routage IP
ip routing

# Création des VLANs
vlan 10
vlan 20
vlan 100

# Interfaces SVI avec adresses IP (passerelles)
interface vlan 10
 ip address 192.168.10.254 255.255.255.0
interface vlan 20
 ip address 192.168.20.254 255.255.255.0

# Port routé (L3) pour connecter un serveur
interface fa0/24
 no switchport
 ip address 192.168.100.254 255.255.255.0

# Trunks vers les switchs L2
interface fa0/22
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 switchport trunk native vlan 100
```

---

## 6. Mise en place de l'accès SSH

Administration à distance des commutateurs depuis un PC du VLAN management.

### Configuration SSH sur switch

```text
enable
configure terminal

# Interface SVI pour administration
interface vlan 101
 ip address 192.168.101.254 255.255.255.0

# Nom de domaine pour clé RSA
ip domain-name gsb.org

# Génération clé RSA (2048 bits)
crypto key generate rsa

# Configuration des lignes virtuelles
line vty 0 15
 login local
 transport input ssh

# Compte utilisateur
user admin password drowssap

# Version SSH et paramètres
ip ssh version 2
ip ssh authentication-retries 5
ip ssh time-out 60

# Mot de passe enable
enable secret drowssap

# Sauvegarde
copy running-config startup-config

# Vérification
show ip ssh
```

> [!WARNING]
> Le mot de passe `drowssap` est un mot de passe de TP. Ne jamais le réutiliser sur un équipement réel.

### Connexion SSH depuis un PC

```bash
# Depuis un terminal (PC6 : 192.168.101.1)
ssh -l admin 192.168.101.254
```

---

## 7. Sauvegarde et restauration (FTP)

### Sauvegarde vers serveur FTP

```text
# Sauvegarde de la configuration running sur serveur FTP
SW1# copy running-config ftp:
Address or name of remote host []? 192.168.101.200
Destination filename [SW1-config]?
```

### Restauration depuis serveur FTP

```text
# Attention : la restauration écrase la configuration courante
SW2# copy ftp: running-config
Address or name of remote host []? 192.168.101.200
Source filename []? SW1-config
```

> [!WARNING]
> Le nom d'hôte et l'adresse SVI seront également copiés ! Pensez à les modifier après restauration.

---

## 8. Vérifications et dépannage

**Commandes utiles :**

- `show vlan brief` : affiche les VLANs et leurs ports
- `show interfaces trunk` : affiche les trunks et VLANs autorisés
- `show mac address-table` : affiche la table CAM
- `show ip interface brief` : affiche les interfaces et adresses IP
- `ping <adresse>` : vérifier la connectivité

**Problèmes fréquents :**

- VLAN natif différent sur les deux extrémités du trunk → erreurs console
- Mode trunk asymétrique (on/off) → lien inopérant
- Mauvaise configuration IP (masque, passerelle) → pas de communication
- VLAN non autorisé sur trunk → paquets non encapsulés

---

## 9. Questions d'application

### Questions sur le TP

1. À quel moment le paquet ICMP est-il étiqueté (taggué) avec le numéro de VLAN ?
2. Quel est le type de protocole Ethernet II avant étiquetage ?
3. Quel est le type de trame Ethernet II après étiquetage ?
4. Comment est numéroté le numéro de VLAN ?
5. Le paquet est-il toujours étiqueté quand il arrive à PC3 ? Pourquoi ?

### Ressources

- Cisco - Configuration VLANs
- The Geek Stuff - Enable SSH on Cisco
- Wikipedia - Ethernet frame
