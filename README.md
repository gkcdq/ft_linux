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
### 🔬 Zoom sur l'infrastructure : Gestion des Démons & Modules (udev & lsmod)

Lors de l'exécution combinée de `systemctl status udev` et `lsmod`, le système expose son architecture de gestion de périphériques et la modularité du Kernel personnalisé `5.15.100-milin`.

---

#### 1. Analyse détaillée du statut de systemd-udevd.service

Le service `systemd-udevd` est le gestionnaire d'événements de périphériques en espace utilisateur (userspace). Il écoute les signaux du Kernel (uevents) à chaque fois qu'un composant matériel est détecté ou modifié.

* **`Loaded: loaded (...; static)`** : Le service est correctement lu par SystemD. L'attribut `static` signifie qu'il ne s'active pas via un lien symbolique classique dans `multi-user.target`, mais qu'il est requis de manière structurelle par le système dès le boot.
* **`Active: active (running)`** : Le démon tourne parfaitement en tâche de fond.
* **`TriggeredBy: ... kernel.socket / control.socket`** : Udev utilise des sockets Unix inter-processus (`.socket`). Dès que le Kernel émet un message réseau bas niveau sur le matériel, le socket se réveille et transmet l'information au démon `systemd-udevd`.
* **`Main PID: 381`** : L'identifiant unique du processus principal en mémoire (Process ID) attribué par le Kernel est le `381`.
* **`Status: "Processing with 40 children at max"`** : Udev est capable de paralléliser la configuration de ton matériel. Il s'autorise à créer jusqu'à 40 processus enfants simultanés pour charger des pilotes sans bloquer le démarrage de la VM.
* **`CGroup: /system.slice/...`** : SystemD isole le service dans un groupe de contrôle (Control Group). Cela permet de limiter et de monitorer finement les ressources RAM/CPU consommées spécifiquement par Udev.

---

#### 2. Analyse détaillée du tableau de chargement des modules (`lsmod`)

La commande `lsmod` extrait dynamiquement les informations du système de fichiers virtuel `/proc/modules`. Elle liste les pilotes chargés à chaud à l'instant $T$.

| Nom du Module | Taille (Octets) | Dépendances / Utilisé par | Rôle technique dans la VM VirtualBox |
| :--- | :--- | :--- | :--- |
| **`snd_seq_dummy`** | 16384 | 0 | Module de séquençage ALSA (Audio) "factice", souvent chargé par défaut pour les interfaces de test MIDI. |
| **`snd_hrtimer`** | 16384 | 1 | Timer haute résolution (High-Resolution Timer) du noyau utilisé par le sous-système de son pour la précision du timing audio. |
| **`snd_seq`** | 94208 | 7 (`snd_seq_dummy`) | Le cœur du séquenceur audio Linux. Il gère le routage des événements audio et MIDI. Il est activement utilisé par 7 autres composants ou sous-modules (dont le dummy). |
| **`snd_seq_device`** | 16384 | 1 (`snd_seq`) | Fournit une couche d'abstraction pour lier les périphériques matériels audio au séquenceur général. |
| **`rfkill`** | 32768 | 3 | Sous-système crucial permettant de désactiver les commutateurs radio (Wi-Fi, Bluetooth). Très utilisé par l'interface réseau `wlo1`. |
| **`qrtr`** | 20480 | 4 | *Qualcomm IPC Router*, un protocole réseau interne du noyau utilisé pour la communication inter-processus bas niveau (IPC). |
| **`ns`** | 36864 | 1 (`qrtr`) | Module lié aux espaces de noms (Namespaces) réseau ou IPC spécifiques utilisés pour isoler les sockets de communication. |
| **`binfmt_misc`** | 24576 | 1 | Permet au Kernel de reconnaître et de lancer directement des formats de fichiers binaires non natifs (ex: exécuter des apps Windows via Wine ou des scripts de manière transparente). |
| **`intel_rapl_msr`** | 20480 | 0 | Pilote de gestion d'énergie (Intel Running Average Power Limit via Model-Specific Registers). Il permet au Kernel de lire et limiter la consommation électrique du processeur émulé par VirtualBox. |

#### 🔄 Synergie Udev / Kernel (Ce que cela prouve)
La présence conjointe de ces deux sorties prouve le comportement dynamique de la distribution :
1. Le **Kernel** détecte un composant émulé par VirtualBox (ex: la carte son Intel AC97 ou la carte réseau).
2. Un *uevent* est envoyé sur le socket d'**Udev**.
3. **Udev** fait correspondre l'événement avec ses règles internes (`/lib/udev/rules.d/`) et appelle l'utilitaire `modprobe`.
4. Le module correspondant est chargé en mémoire vive et apparaît instantanément dans la table **`lsmod`**.


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

