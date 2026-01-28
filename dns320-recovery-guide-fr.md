# DNS-320 Rev A - Débrickage et Flash OpenWrt

## Matériel Requis

- D-Link DNS-320 Rev A
- Console série (UART) sur JP4 : TX, RX, GND (3.3V, 115200 baud)
- Adaptateur USB-TTL (FTDI FT232 ou équivalent)
- Clé USB formatée en **FAT32** avec table MBR
- PC Linux avec `u-boot-tools` et `tio` installés

### Accès Console Série

```bash
# Installation de tio
sudo apt install tio

# Connexion au NAS
tio /dev/ttyUSB0

# (le baudrate 115200 est auto-détecté)
# Pour quitter : CTRL+T puis Q
```

**Test du câblage :** Au boot du NAS, tu devrais voir défiler du texte. Si rien n'apparaît, vérifie :
- TX/RX inversés (TX du NAS → RX de l'adaptateur)
- GND connecté
- Adaptateur en 3.3V (pas 5V !)

## Fichiers Nécessaires

Télécharger depuis [APCCV/openwrt-dns320](https://github.com/APCCV/openwrt-dns320/releases) :
- `u-boot.kwb`
- `env.bin`
- `ubi.bin`

---

## ⚠️ Comment j'ai Briqué le NAS

**Erreur "No partition table" ignorée**

```bash
=> fatload usb 0:1 0x1000000 u-boot.kwb
** No partition table - usb 0 **
```

❌ **J'ai ignoré l'erreur et continué** → la clé USB n'était pas correctement reconnue, fichier pas chargé.

**Cause possible :** Partition mal créée (pas de table MBR) ou clé USB incompatible.

**Effacement du NAND sans vérification**

```bash
=> nand erase 0x0 0x0e0000
=> nand write 0x1000000 0x0 0x0e0000
```

Aucune erreur affichée → j'ai cru que ça marchait.  
**Résultat :** NAND effacé avec rien dedans → **NAS briqué** 🧱

**Leçon :** Toujours vérifier `printenv filesize` après `fatload` !

---

## 🛠️ Recovery via kwboot

**Symptôme du brick :** Console série complètement muette au boot.

### Procédure

```bash
# Dans un terminal : connexion série
tio /dev/ttyUSB0

# Dans un autre terminal : kwboot
sudo apt install u-boot-tools

# Débranche l'alimentation du NAS

# Lance kwboot
sudo kwboot -t -B 115200 /dev/ttyUSB0 -b u-boot.kwb

# Rebranche l'alimentation pendant que kwboot tourne
```

**Note :** Pendant kwboot, `tio` va afficher des caractères bizarres (normal, c'est le transfert). Attends la fin.

### ✅ Résultat attendu

```
Sending boot message. Please reboot the target...
Sending boot image...
[=====================] 100%
Done finishing transfer
```

Puis le prompt u-boot apparaît : `=>`

**Si tu vois ça** → u-boot chargé en RAM (temporaire).  
**Si rien** → problème de câblage série ou mauvais fichier u-boot.kwb.

---

## ✅ Installation Permanente

### Préparation Clé USB

**Formatage FAT32 :**

```bash
# Crée une partition MBR
sudo fdisk /dev/sdX
# o (table MBR), n (nouvelle partition), p (primaire), 1, ENTER, ENTER
# t, c (FAT32 LBA), w (écrire)

# Formate en FAT32
sudo mkfs.vfat -F 32 /dev/sdX1

# Monte et copie
sudo mount /dev/sdX1 /mnt
cp u-boot.kwb env.bin ubi.bin /mnt/
sudo umount /mnt
```

---

## Étape 1 : Flash u-boot Permanent

**Dans u-boot :**

```bash
usb reset
usb storage
```

### ✅ Résultat attendu

```
Bus ehci@50000: USB EHCI 1.00
scanning bus ehci@50000 for devices... 2 USB Device(s) found
       scanning usb for storage devices... 1 Storage Device(s) found

  Device 0: Vendor: ... Prod: ... Rev: ...
            Type: Removable Hard Disk
            Capacity: XXXX MB
```

**Si tu vois ça** → clé USB détectée.  
**Si "0 Storage Device(s) found"** → retire/rebranche la clé, recommence `usb reset`.

---

```bash
fatload usb 0:1 0x1000000 u-boot.kwb
```

### ✅ Résultat attendu

```
590156 bytes read in 45 ms (12.5 MiB/s)
```

**Si tu vois ça** → fichier chargé OK.  
**Si "No partition table" ou timeout** → retire/rebranche la clé USB, `usb reset`, retente.

---

```bash
printenv filesize
```

### ✅ Résultat attendu

```
filesize=90156
```

**Taille en décimal :** 590156 bytes (correspond à la taille du fichier u-boot.kwb)

**Si vide ou 0** → le fichier n'est PAS chargé, ne continue pas !  
**Si valeur différente** → fichier partiellement chargé ou corrompu, retente.

---

```bash
nand erase 0x0 0x0e0000
nand write 0x1000000 0x0 0x0e0000
saveenv
reset
```

### ✅ Résultat attendu après reset

Le NAS reboot et affiche :

```
U-Boot 2020.04 (Dec 17 2025 - 21:08:22 +0000)
D-Link DNS-320
...
Hit any key to stop autoboot:
=>
```

**Si tu vois ça SANS kwboot** → u-boot permanent OK ! 🎉  
**Si rien** → u-boot pas flashé, recommence avec kwboot.

---

## Étape 2 : Charger l'Environnement APCCV

```bash
usb reset
fatload usb 0:1 0x1000000 env.bin
env import -b 0x1000000 ${filesize}
```

### ✅ Résultat attendu

```
[quelques lignes de variables importées]
```

**Si erreur** → fichier env.bin corrompu ou pas chargé.

---

```bash
setenv ethaddr 'XX:XX:XX:XX:XX:XX'  # Ta MAC
printenv mtdparts
```

### ✅ Résultat attendu

```
mtdparts=orion_nand:0x0e0000@0x0(uboot),0x20000@0x0e0000(ubootenv),0x7f00000@0x100000(ubi)
```

**Si tu vois ça** → environnement correct.  
**Si différent** → charge mal env.bin, recommence.

---

```bash
saveenv
```

---

## Étape 3 : Flash OpenWrt (ubi.bin)

```bash
nand erase 0x100000 0x7f00000
```

### ✅ Résultat attendu

```
NAND erase: device 0 offset 0x100000, size 0x7f00000
Skipping bad block at  0xXXXXXXXX  (peut apparaître)
Erasing at 0x7fe0000 -- 100% complete.
OK
```

**Si tu vois ça** → partition UBI effacée.  
**Si erreur** → problème NAND hardware (rare).

---

```bash
usb reset
fatload usb 0:1 0x1000000 ubi.bin
```

### ✅ Résultat attendu

```
11010048 bytes read in 456 ms (23 MiB/s)
```

**Si tu vois ça** → fichier chargé.  
**Si timeout `EHCI timed out on TD`** → retire/rebranche clé USB, `usb reset`, retente (peut prendre 2-3 essais).

---

```bash
printenv filesize
```

### ✅ Résultat attendu

```
filesize=a80000
```

**Taille en décimal :** 11010048 bytes (correspond à la taille du fichier ubi.bin)

**Si vide** → pas chargé, recommence !  
**Si valeur différente** → fichier partiellement chargé, retente le `fatload`.

---

```bash
nand write 0x1000000 0x100000 ${filesize}
```

### ✅ Résultat attendu

```
NAND write: device 0 offset 0x100000, size 0xa80000
 11010048 bytes written: OK
```

**Si tu vois ça** → OpenWrt flashé ! 🎉  
**Si erreur** → problème NAND.

---

```bash
reset
```

---

## Boot OpenWrt Réussi

### ✅ Résultat attendu

```
U-Boot 2020.04 (Dec 17 2025 - 21:08:22 +0000)
...
ubi0: attaching mtd2
ubi0: scanning is finished
ubi0: attached mtd2 (name "ubi", size 127 MiB)
...
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 6.6.x ...
...

Please press Enter to activate this console
login[758]: root login on 'ttyS0'


BusyBox v1.36.1 (2025-12-17 21:08:22 UTC) built-in shell (ash)

  _______                     ________        __
 |       |.-----.-----.-----.|  |  |  |.----.|  |_
 |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
 |_______||   __|_____|__|__||________||__|  |____|
          |__| W I R E L E S S   F R E E D O M
 -----------------------------------------------------
 OpenWrt 24.10.5, r29087-d9c5716d1d
 -----------------------------------------------------

root@(none):~# 

```

**Si tu vois ça** → ✅ **Installation réussie !** 🎉

**Login :** 
Tu es ensuite invité à changer le mot de passe root. Fais le !

---

## ❌ Problèmes Possibles

### "No partition table" avec FAT32

```bash
=> fatload usb 0:1 0x1000000 u-boot.kwb
** No partition table - usb 0 **
```

**Cause :** Clé USB sans table de partition MBR, ou partition directement formatée.  
**Solution :** Recréer la clé avec fdisk (voir section Préparation Clé USB) :
- Créer table MBR (`o` dans fdisk)
- Créer partition primaire (`n`, `p`, `1`)
- Type FAT32 LBA (`t`, `c`)
- Formater (`mkfs.vfat -F 32`)

Après ça, `fatload` devrait marcher.

```
ubi0 error: the layout volume was not found
Kernel panic - not syncing: VFS: Unable to mount root fs
```

**Cause :** Mauvais `bootargs` (ubi.mtd incorrect).  
**Solution :** Recharge `env.bin` avec `env import`.

### Timeouts USB répétés

```
EHCI timed out on TD - token=0x...
```

**Cause :** Combinaison clé USB + vieux contrôleur Kirkwood.  
**Solution :** Retire/rebranche la clé, `usb reset`, retente (patience !).

### Rien au boot après flash

**Cause :** u-boot ou ubi.bin mal flashé.  
**Solution :** Recommence avec kwboot depuis le début.

---

## 📋 Résumé des Points Clés

### ✅ Ce qui marche

- **FAT32 fonctionne** avec `fatload` (pas besoin d'ext4)
- **Vérifier `filesize`** après chaque `fatload` (crucial !)
  - La valeur de `filesize` doit **correspondre exactement** à la taille du fichier chargé
  - Si différent → fichier partiellement chargé ou corrompu
- **Serial console indispensable** pour recovery
- **env.bin d'APCCV** évite les erreurs de config
- **Retenter les chargements USB** si timeout (normal sur vieux hardware)

### ❌ À éviter

- Ignorer `No partition table` → créer table MBR avec fdisk
- Effacer le NAND sans vérifier `filesize`
- Ne pas vérifier que le fichier est bien chargé avant de flasher

---

## Spécifications Finales

**Software :**
- U-Boot: 2020.04 (APCCV)
- OpenWrt: 24.10.5-r1
- Kernel: Linux 6.6.x LTS

**Partitionnement NAND :**
```
0x000000 - 0x0e0000 : u-boot     (896KB)
0x0e0000 - 0x100000 : u-boot-env (128KB)
0x100000 - 0x8000000 : ubi       (127MB)
```

---

## 🙏 Remerciements

Merci à **APCCV** pour le port OpenWrt DNS-320 !

**Projet :** https://github.com/APCCV/openwrt-dns320  
**Date :** Janvier 2026

