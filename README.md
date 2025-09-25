# Mise à niveau de "CentOS Stream 8" vers "CentOS Stream 9"

**CentOS Stream 8** est en fin de vie depuis mai 2024.

Les dépôts officiels de **CentOS Stream 8** ont été déplacés dans le **CentOS Vault (archive)**.
Il faut éditer tes fichiers **.repo** pour pointer vers ces dépôts archivés.


## 1.	Sauvegarde tes anciens fichiers

### Sauvegarde des fichiers de dépôts actuels

```bash
sudo cp -r /etc/yum.repos.d /etc/yum.repos.d.backup
```

# 2. Préparation des dépôts CentOS Stream 8 (archive)

### Configuration des nouveaux dépôts archivés

```bash
cat /etc/yum.repos.d/CentOS-Stream-AppStream.repo

[appstream]
name=CentOS Stream 8 - AppStream
#mirrorlist=http://mirrorlist.centos.org/?release=$stream&arch=$basearch&repo=AppStream&infra=$infra
baseurl=http://vault.centos.org/8-stream/AppStream/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial


cat /etc/yum.repos.d/CentOS-Stream-BaseOS.repo

[baseos]
name=CentOS Stream 8 - BaseOS
#mirrorlist=http://mirrorlist.centos.org/?release=$stream&arch=$basearch&repo=BaseOS&infra=$infra
baseurl=http://vault.centos.org/8-stream/BaseOS/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial


cat /etc/yum.repos.d/CentOS-Stream-Extras-common.repo

[extras-common]
name=CentOS Stream 8 - Extras common packages
#mirrorlist=http://mirrorlist.centos.org/?release=$stream&arch=$basearch&repo=extras-extras-common
baseurl=http://vault.centos.org/8-stream/extras/$basearch/extras-common/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Extras


cat /etc/yum.repos.d/CentOS-Stream-Extras.repo

[extras]
name=CentOS Stream 8 - Extras
# mirrorlist=http://mirrorlist.centos.org/?release=$stream&arch=$basearch&repo=extras&infra=$infra
baseurl=http://vault.centos.org/8-stream/extras/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
```


### Nettoyage et mise à jour du cache

```bash
sudo dnf clean all
sudo dnf makecache
```

Après ça, tous tes dépôts devraient être fonctionnels depuis le Vault.
Tu pourras installer leapp et lancer la migration vers CentOS Stream 9.


# Prérequis

## 1.	Vérifiez que votre système est bien CentOS Stream 8

### Vérification de la version actuelle

```bash
cat /etc/centos-release

# Doit afficher :
CentOS Stream release 8
```

### Mise à jour complète du système

```bash
sudo dnf update -y
sudo reboot
```

## 3. Préparation des modules et dépendances

### Désactivation des modules problématiques

```bash
sudo dnf module disable python36 virt -y

# Réinitialisation du module Python 3.6
sudo dnf module reset python36 -y

# Activation de Python 3.8
sudo dnf module enable python38 -y

# Installation de Python 3.8
sudo dnf install -y python38 python3-pip

# Installation d'EPEL et des dépendances nécessaires
sudo dnf install -y epel-release
sudo dnf install -y dnf-plugins-core
sudo dnf copr enable evgeni/leapp
```

## 4. Installation de Leapp et préparation de la migration

### Installation de Leapp

```bash
sudo dnf install -y leapp leapp-upgrade-el8toel9
```


## 5. Résolution des problèmes connus avant la migration

```bash
# Configuration SSH - désactivation du login root par mot de passe

sudo sed -i 's/^PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sudo sed -i 's/^PermitRootLogin yes/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sudo systemctl restart sshd


# Configuration de firewalld
sudo sed -i 's/^AllowZoneDrifting=yes/#AllowZoneDrifting=yes/' /etc/firewalld/firewalld.conf
sudo systemctl restart firewalld

# Vérification et installation de VDO si nécessaire
# Vérifier si VDO est utilisé
sudo vdo status

# Si aucun périphérique VDO n'est configuré, installer le paquet manquant :
sudo dnf install -y vdo kmod-kvdo

# Vérification des périphériques VDO
sudo vdo status
```

## 6. Configuration des dépôts CentOS Stream 9 pour la migration

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
```

## 7. Résolution des inhibiteurs (erreurs bloquantes)

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

## 8. Exécution de la vérification pré-migration

### Lancement de la vérification préalable

```bash
sudo leapp preupgrade
```

## 9. Exécution de la migration

### Effectuer une mise à niveau

```bash
sudo leapp upgrade

# Redémarrer
sudo reboot
```


## 10. Vérification de la nouvelle version

```bash
# Confirmer la version actuelle
cat /etc/redhat-release

# Vérifier la version du kernel
uname -r

# Vérifier les services
systemctl status
```


## 11. Nettoyage post-migration

```bash
# Supprimer les paquets leapp (plus nécessaires)
sudo dnf remove -y leapp leapp-deps leapp-upgrade-el8toel9 leapp-upgrade-el8toel9-deps python3-leapp

# Nettoyer les dépendances inutiles
sudo dnf autoremove -y

# Remettre SELinux en mode enforcing (s'il était en permissive)
sudo setenforce 1
sudo sed -i 's/SELINUX=permissive/SELINUX=enforcing/' /etc/selinux/config

# Mise à jour complète du nouveau système
sudo dnf update -y
```


## 12. Vérifications importantes

### Vérifier les services critiques

```bash
sudo systemctl status sshd
sudo systemctl status NetworkManager
sudo systemctl status firewalld

# Vérifier les logs d'erreur
sudo journalctl -p err -b

# Vérifier l'espace disque
df -h
```