# Atelier Samba : Remplacer Windows Server et Active Directory par Linux - Debian

## 📌 Présentation
L'objectif de cet atelier est de remplacer un serveur Windows Server et sa forêt Active Directory par une solution alternative open-source sous Linux utilisant **Samba 4**. 

**Bénéfice principal :** Économie sur le coût des licences serveur et suppression du besoin d'acheter des licences d'accès client (CAL) pour chaque utilisateur ou poste de travail.

---

## 🛠️ Étape 1 : Machine Virtuelle Debian 12

### 1.1 Spécifications de la VM
Créez une machine virtuelle sous Debian 12 sur Proxmox avec :
- **RAM :** 4 Go
- **Processeur :** 2 vCPUs
- **Stockage :** 50 Go

> ⚠️ **Important :** Lors de l'installation du système, on utilisera `debianSRV` comme nom d'hôte (*hostname*) et `OCLOCK.LAN` comme nom de domaine. Ne cochez aucun environnement de bureau (GNOME, XFCE...), uniquement le serveur SSH.

### 1.2 Configuration Réseau (`/etc/network/interfaces`)
Éditez la configuration réseau afin d'assigner une adresse IP fixe au serveur :

```bash
sudo nano /etc/network/interfaces
```

```text
allow-hotplug ens18
iface ens18 inet static
    address 10.0.0.10
    netmask 255.255.255.0
    network 10.0.0.0
    broadcast 10.0.255.255
    gateway 10.0.0.1
    dns-nameservers 8.8.8.8
    dns-search OCLOCK.LAN
```

Mettez à jour le fichier `/etc/hosts` :

```text
10.0.0.10       debianSRV.OCLOCK.LAN debianSRV
```

Et le fichier `/etc/hostname` :

```text
debianSRV.OCLOCK.LAN
```

Redémarrez le serveur avec `sudo reboot` et validez l'accès avec `ip a` et `ping google.com`.

---

## 📦 Étape 2 : Compilation et Installation de Samba

### 2.1 Installation des Dépendances
Installez les outils de compilation et les bibliothèques requises :

```bash
sudo apt update
sudo apt install -y build-essential libacl1-dev libattr1-dev libblkid-dev \
libgnutls28-dev libreadline-dev gdb pkg-config libpopt-dev python3-dev \
liblmdb-dev flex bison libgpgme-dev libparse-yapp-perl libjansson-dev \
libarchive-dev libdbus-1-dev
```

*(Lors de la configuration Kerberos, validez `OCLOCK.LAN` pour le royaume et `debianSRV` pour le nom d'hôte).*

### 2.2 Compilation de Samba depuis les sources
Téléchargez la dernière version stable de Samba, configurez-la et lancez la compilation :

```bash
wget http://ftp.samba.org/pub/samba/samba-latest.tar.gz
tar zxvf samba-latest.tar.gz
cd samba-*
./configure --enable-debug --enable-selftest
make
sudo make install
```

### 2.3 Provisionnement du Domaine Active Directory
Supprimez tout fichier de configuration préexistant et lancez l'outil de provisionnement :

```bash
sudo rm -f /usr/local/samba/etc/smb.conf
sudo /usr/local/samba/bin/samba-tool domain provision
```

*(Validez les choix par défaut. Le mot de passe administrateur doit obligatoirement respecter la complexité : 7 caractères minimum, avec majuscule, minuscule, chiffre et caractère spécial).*

Démarrage de test et vérification des partages système (`netlogon` et `sysvol`) :

```bash
sudo /usr/local/samba/sbin/samba
sudo /usr/local/samba/bin/smbclient -L localhost -U%
```

---

## ⚙️ Étape 3 : Configurations Avancées

### 3.1 DNS & Kerberos
Modifiez le fichier `/etc/resolv.conf` pour pointer sur le domaine local :

```text
search OCLOCK.LAN
domain OCLOCK.LAN
nameserver 10.0.0.10
```

Vérifiez le contenu de `/usr/local/samba/etc/smb.conf` :

```ini
[global]
    dns forwarder = 10.0.0.1
    netbios name = DEBIANSRV
    realm = OCLOCK.LAN
    server role = active directory domain controller
    workgroup = OCLOCK

[netlogon]
    path = /usr/local/samba/var/locks/sysvol/oclock.lan/scripts
    read only = No

[sysvol]
    path = /usr/local/samba/var/locks/sysvol
    read only = No
```

Liez la configuration Kerberos au système :

```bash
sudo ln -sf /usr/local/samba/share/setup/krb5.conf /etc/krb5.conf
```

Désactivez l'expiration du mot de passe Administrateur :

```bash
sudo /usr/local/samba/bin/samba-tool user setexpiry administrator --noexpiry
```

### 3.2 Synchronisation Temporelle (NTP)
Configurez la synchronisation de l'heure avec `systemd-timesyncd` :

```bash
sudo timedatectl set-ntp true
```

Éditez `/etc/systemd/timesyncd.conf` :

```ini
[Time]
NTP=10.0.0.1
FallbackNTP=0.fr.pool.ntp.org 1.fr.pool.ntp.org
```

Redémarrez le service : `sudo systemctl restart systemd-timesyncd`.

### 3.3 Zone Inversée DNS
Créez la zone de recherche inversée via `samba-tool` :

```bash
sudo /usr/local/samba/bin/samba-tool dns zonecreate debianSRV 0.0.10.in-addr.arpa -U administrator
```

---

## 💻 Étape 4 : Administration et Clients Windows

### 4.1 Jointure du Domaine
1. Connectez un poste client Windows 10/11 Pro sur le même réseau.
2. Définissez le serveur DNS primaire de la carte réseau Windows sur l'IP du serveur Samba (`10.0.0.10`).
3. Rejoignez le domaine `OCLOCK.LAN` en utilisant le compte `administrator`.

### 4.2 Installation des outils d'administration RSAT
Exécutez les commandes suivantes dans une console PowerShell en tant qu'administrateur :

```powershell
Add-WindowsCapability -Online -Name "Rsat.ServerManager.Tools~~~~0.0.1.0"
Add-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
Add-WindowsCapability -Online -Name "Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0"
```

Ouvrez la console `mmc` et ajoutez les composants **Utilisateurs et ordinateurs Active Directory** ainsi que **Gestion des stratégies de groupe (GPO)**.

### 4.3 Partage de Fichiers Samba
Sur le serveur Linux, ajoutez le partage suivant dans `/usr/local/samba/etc/smb.conf` :

```ini
[public]
    path = /home/shares/public
    guest ok = no
    guest only = yes
    writeable = yes
    force create mode = 0666
    force directory mode = 0777
    browseable = yes
```

Créez le dossier physique et ajustez les droits :

```bash
sudo mkdir -p /home/shares/public
sudo chmod 777 /home/shares/public
```

---

## Bonus : Création du Service Systemd (Daemon)

Afin d'exécuter Samba automatiquement au démarrage sous forme de démon (*daemon*), créez le fichier d'unité Systemd suivant :

```bash
sudo nano /etc/systemd/system/samba-ad-dc.service
```

Ajoutez la configuration :

```ini
[Unit]
Description=Samba Active Directory Domain Controller
After=network.target remote-fs.target nss-lookup.target time-sync.target

[Service]
Type=forking
ExecStart=/usr/local/samba/sbin/samba
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Rechargez la configuration de Systemd, activez le service au démarrage et lancez-le :

```bash
sudo systemctl daemon-reload
sudo systemctl enable samba-ad-dc.service
sudo systemctl start samba-ad-dc.service
```

Vérifiez l'état du service avec :

```bash
sudo systemctl status samba-ad-dc.service
```
