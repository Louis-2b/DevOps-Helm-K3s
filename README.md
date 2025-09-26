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


### Nettoyage et mise à jour du cache

```bash
sudo dnf clean all
sudo dnf makecache
```

Après ça, tous tes dépôts devraient être fonctionnels depuis le Vault.
Tu pourras installer leapp et lancer la migration vers CentOS Stream 9.


# Prérequis

## 1. Vérifier votre système

### Vérification de la version actuelle (Vérifiez que votre système est bien CentOS Stream 8)

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
# Réinitialisation du module Python 3.6
sudo dnf module disable python36 virt -y
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
EOF
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

# Sortie :

============================================================
                      REPORT OVERVIEW
============================================================

HIGH and MEDIUM severity reports:
    1. GRUB2 core will be automatically updated during the upgrade
    2. Leapp detected loaded kernel drivers which are no longer maintained in RHEL 9.
    3. Packages not signed by Red Hat found on the system

Reports summary:
    Errors:                      0
    Inhibitors:                  0
    HIGH severity reports:       3
    MEDIUM severity reports:     0
    LOW severity reports:        3
    INFO severity reports:       3

Before continuing, review the full report below for details about discovered problems and possible remediation instructions:
    A report has been generated at /var/log/leapp/leapp-report.txt
    A report has been generated at /var/log/leapp/leapp-report.json

============================================================
                   END OF REPORT OVERVIEW
============================================================
```

## 9. Exécution de la migration

### Effectuer une mise à niveau

```bash
sudo leapp upgrade

# Sortie :

============================================================
                      REPORT OVERVIEW
============================================================

HIGH and MEDIUM severity reports:
    1. GRUB2 core will be automatically updated during the upgrade
    2. Leapp detected loaded kernel drivers which are no longer maintained in RHEL 9.
    3. Packages not signed by Red Hat found on the system

Reports summary:
    Errors:                      0
    Inhibitors:                  0
    HIGH severity reports:       3
    MEDIUM severity reports:     0
    LOW severity reports:        3
    INFO severity reports:       3

Before continuing, review the full report below for details about discovered problems and possible remediation instructions:
    A report has been generated at /var/log/leapp/leapp-report.txt
    A report has been generated at /var/log/leapp/leapp-report.json

============================================================
                   END OF REPORT OVERVIEW
============================================================

# Redémarrer
sudo reboot
```

## 10. Nettoyage post-migration

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