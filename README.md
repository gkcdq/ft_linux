# ft_linux - How to Train Your Kernel

Ce projet consiste à configurer, compiler et installer un noyau Linux personnalisé (version >= 4.0) sur une machine virtuelle, en implémentant une hiérarchie de système de fichiers conforme aux standards, une gestion des modules dynamiques, et une connectivité réseau complète.

---

## 📊 Informations de la Distribution

* **Nom de la machine (Hostname) :** `milin`
* **Version du Kernel :** `5.15.100-milin`
* **Architecture :** x86_64 (64-bit)
* **Gestionnaire de démons :** SystemD
* **Chargeur d'amorçage :** GRUB 2

---

## 🛠️ Guide de Construction (Compilation du Kernel)

Voici les étapes exactes qui ont été suivies à l'intérieur de la machine pour concevoir ce système :

### 1. Installation des dépendances du Chapitre IV
```bash
apt update && apt upgrade -y
apt install -y build-essential bzip2 xz-utils bc bison flex libncurses-dev \
libssl-dev libelf-dev kmod udev grub2 systemd-sysv wget curl vim tcl gettext \
texinfo libcap-dev pkg-config autoconf automake check gawk e2fsprogs expect gperf
```
### 2. Récupération et extraction des sources
```Bash
cd /usr/src
wget [https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.100.tar.xz](https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.100.tar.xz)
tar -xf linux-5.15.100.tar.xz
mv linux-5.15.100 kernel-5.15.100
ln -s /usr/src/kernel-5.15.100 /usr/src/linux-5.15.100  # Lien symbolique de conformité barème
cd kernel-5.15.100
```
### 3. Configuration et Signature du Kernel
```Bash
make defconfig
./scripts/config --disable CONFIG_DEBUG_INFO_BTF  # Correction erreur pahole
make menuconfig
# Action : Ajout de "-milin" dans "Local version - append to kernel release"
```

### 4. Compilation et Installation
```Bash
make -j$(nproc)
make modules_install
make install
```

## 1. Vérification du Kernel et de la Signature

Prouve que le noyau tourne sous la version personnalisée avec le login étudiant :
```Bash
uname -a
# Attendu : Linux milin 5.15.100-milin ...
```

Vérifie l'emplacement exact des sources :
```Bash
ls -l /usr/src/
# Attendu : kernel-5.15.100 et le lien symbolique linux-5.15.100
```

## 2. Vérification du Partitionnement (Au moins 3 partitions distinctes)

Prouve la séparation stricte de la racine, du boot et de la swap :
```Bash
lsblk
# Attendu : 
# - Une partition montée sur /boot (ex: sda1)
# - Une partition de type [SWAP] (ex: sda2)
# - Une partition montée sur / (ex: sda3)
```
## 3. Gestionnaire de Modules & Démons

Prouve qu'un gestionnaire de d'arrière-plan et un chargeur de modules (type udev) sont actifs :
```Bash
systemctl status udev --no-pager
lsmod | head -n 10
```
## 4. Nom du Binaire de Boot

Prouve que le fichier binaire de GRUB respecte la nomenclature demandée :
```Bash
ls -l /boot/vmlinuz*
# Attendu : /boot/vmlinuz-5.15.100-milin
```
## 5. Connexion Internet (The Interweb)

Prouve la connectivité brute et la résolution de noms (DNS) :
```Bash
ping -c 3 8.8.8.8
ping -c 3 google.com
```
## 6. Le Crash Test ("The Real Test")

Le correcteur demandera d'installer le paquet screen à la volée pour valider le gestionnaire de paquets :
```Bash
apt install -y screen
screen --version
```
