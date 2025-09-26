# Guide de mise à niveau de **CentOS Stream 8** vers **CentOS Stream 9**

### Contexte important
**CentOS Stream 8** est en fin de vie depuis mai 2024. Les dépôts officiels ont été déplacés vers le **CentOS Vault (archive)**. Cette procédure utilise l'outil **Leapp** pour effectuer une mise à niveau en place.


## Prérequis et préparation

### 1. Sauvegarde complète
⚠️ CRITIQUE : Effectuez une sauvegarde complète de votre système avant de commencer.

### 2. Vérification du système actuel
```bash
# Vérifier la version actuelle
cat /etc/centos-release
# Doit afficher : CentOS Stream release 8

# Vérifier l'architecture
uname -m

# Vérifier l'espace disque disponible (minimum 5 GB libres)
df -h /
```


## Configuration des dépôts CentOS Stream 8 (Vault)

### 1. Sauvegarde des fichiers de dépôts actuels

```bash
sudo cp -r /etc/yum.repos.d /etc/yum.repos.d.backup
```

### 2. Configuration des dépôts archivés

```bash
# Supprimer les anciens fichiers de dépôts
sudo rm -f /etc/yum.repos.d/CentOS-Stream-*.repo

# Créer les nouveaux fichiers de dépôts pointant vers le Vault
sudo tee /etc/yum.repos.d/CentOS-Stream-AppStream.repo > /dev/null <<EOF
[appstream]
name=CentOS Stream 8 - AppStream
baseurl=https://vault.centos.org/8-stream/AppStream/\$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
EOF

sudo tee /etc/yum.repos.d/CentOS-Stream-BaseOS.repo > /dev/null <<EOF
[baseos]
name=CentOS Stream 8 - BaseOS
baseurl=https://vault.centos.org/8-stream/BaseOS/\$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
EOF

sudo tee /etc/yum.repos.d/CentOS-Stream-Extras-common.repo > /dev/null <<EOF
[extras-common]
name=CentOS Stream 8 - Extras common packages
baseurl=https://vault.centos.org/8-stream/extras/\$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Extras
EOF

sudo tee /etc/yum.repos.d/CentOS-Stream-Extras.repo > /dev/null <<EOF
[extras]
name=CentOS Stream 8 - Extras
baseurl=https://vault.centos.org/8-stream/extras/\$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
EOF
```

### 3. Test des nouveaux dépôts

```bash
# Nettoyer et reconstruire le cache
sudo dnf clean all
sudo dnf makecache
```
Après ça, tous tes dépôts devraient être fonctionnels depuis le Vault.
Tu pourras installer leapp et lancer la migration vers CentOS Stream 9.


### Mise à jour complète du système

```bash
# Mise à jour complète
sudo dnf update -y

# Redémarrage obligatoire
sudo reboot
```

## Installation et configuration de Leapp

### 1. Installation des prérequis

```bash
# Réinitialisation du module Python 3.6
sudo dnf module disable python36 virt -y
sudo dnf module reset python36 -y

# Activation de Python 3.8
sudo dnf module enable python38 -y

# Installation de Python 3.8
sudo dnf install -y python38 python3-pip

# # Installation d'EPEL et des outils nécessaires
sudo dnf install -y epel-release dnf-plugins-core

# Activation du dépôt Copr pour Leapp
sudo dnf copr enable evgeni/leapp
```

### 2. Installation de Leapp

```bash
sudo dnf install -y leapp leapp-upgrade-el8toel9
```

## Préparation de la migration

### 1. Configuration SSH

```bash
# Configuration SSH - désactivation du login root par mot de passe
sudo sed -i 's/^PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sudo sed -i 's/^PermitRootLogin yes/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

### 2. Configuration Firewalld

```bash
sudo sed -i 's/^AllowZoneDrifting=yes/#AllowZoneDrifting=yes/' /etc/firewalld/firewalld.conf
sudo systemctl restart firewalld
```

### 3. Installation des dépendances VDO (si nécessaire)
```bash
# Vérifier si VDO est utilisé
sudo vdo status 2>/dev/null || echo "VDO non configuré"

# Installer les paquets VDO pour éviter les erreurs
sudo dnf install -y vdo kmod-kvdo
```

### 4. Configuration du fichier de réponses Leapp

```bash
# Création du fichier de dépôts pour la migration
sudo mkdir -p /etc/leapp/files

# Créer le fichier avec des noms qui ne conflictent pas
sudo tee /etc/leapp/files/leapp_upgrade_repositories.repo > /dev/null <<EOF
[cs9-baseos-upgrade]
name=CentOS Stream 9 BaseOS for Upgrade
baseurl=http://mirror.stream.centos.org/9-stream/BaseOS/x86_64/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial

[cs9-appstream-upgrade]
name=CentOS Stream 9 AppStream for Upgrade
baseurl=http://mirror.stream.centos.org/9-stream/AppStream/x86_64/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
EOF
```

### 5. Résolution des inhibiteurs (erreurs bloquantes)

```bash
sudo tee /var/log/leapp/answerfile > /dev/null <<'EOF'
[check_vdo]
confirm = True

[remove_pam_pkcs11_module_check]
confirm = True

[authselect_check]
confirm = True

[confirm_inhibitor_override]
confirm = False
EOF
```

## Exécution de la migration

### 1. Vérification pré-migration

```bash
# Lancer la vérification préalable
sudo leapp preupgrade

# Examiner le rapport généré
sudo cat /var/log/leapp/leapp-report.txt
```

### 2. Résolution des inhibiteurs (si nécessaire)
Si des inhibiteurs sont détectés, consultez le rapport et résolvez-les avant de continuer.


### 3. Lancement de la migration

```bash
# Effectuer la mise à niveau
sudo leapp upgrade

# Redémarrer
sudo reboot

# Le système va redémarrer automatiquement plusieurs fois
# La migration peut prendre 30-60 minutes selon la configuration
```

## Vérifications et nettoyage post-migration

### 1. Vérification de la migration

```bash
# Vérifier la nouvelle version
cat /etc/redhat-release
# Doit afficher : CentOS Stream release 9

# Vérifier le kernel
uname -r
```










### 3. Nettoyage du système

```bash
# Supprimer les paquets leapp (plus nécessaires)
sudo dnf remove -y leapp leapp-deps leapp-upgrade-el8toel9 leapp-upgrade-el8toel9-deps python3-leapp

# Désactiver ou supprimer le dépôt EPEL Modular défaillant
sudo dnf config-manager --disable epel-modular

# Ou le supprimer complètement
sudo rm -f /etc/yum.repos.d/epel-modular.repo

# Nettoyer le cache
sudo dnf clean all

# Remettre SELinux en mode enforcing (s'il était en permissive)
sudo setenforce 1
sudo sed -i 's/SELINUX=permissive/SELINUX=enforcing/' /etc/selinux/config
```

## 11. Vérifications post-migration

```bash
# Vérifier la version (devrait afficher CentOS Stream 9)
cat /etc/redhat-release

# Supprimer les anciens dépôts CentOS 8
sudo rm -f /etc/yum.repos.d/CentOS-Stream-*.repo

# Installer les dépôts officiels pour CentOS Stream 9
sudo dnf install centos-stream-repos

# Réinstaller EPEL pour CentOS 9
sudo dnf remove epel-release epel-next-release
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm

# Essayer à nouveau la mise à jour maintenant
sudo dnf update -y

# Vérifier les dépôts actifs
sudo dnf repolist --enabled

# Nettoyer les dépendances inutiles
sudo dnf autoremove -y
```

### Vérifications finales

```bash
# Vérifier la version kernel
uname -r

# Vérifier les services
sudo systemctl status sshd NetworkManager firewalld

# Vérifier SELinux
getenforce

# Vérifier l'espace disque
df -h
```