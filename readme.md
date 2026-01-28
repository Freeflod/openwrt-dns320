# OpenWrt pour D-Link DNS-320 Rev A

[English version below](#english-version)

---

## 🇫🇷 Version Française

### À propos de ce fork

Ce dépôt est un fork du projet [APCCV/openwrt-dns320](https://github.com/APCCV/openwrt-dns320), qui apporte le support d'OpenWrt pour le D-Link DNS-320 Rev A.

**Ma démarche :**

J'avais un vieux DNS-320 Rev A qui dormait dans un sac depuis des années. Il y a longtemps, j'avais réussi à booter Debian dessus via USB, mais je n'avais jamais terminé de flasher le NAND et n'avais pas eu le temps d'aller plus loin.

En découvrant le projet d'APCCV, j'ai été très enthousiaste à l'idée de flasher tout l'OS dans le NAND. Après quelques péripéties (notamment un brickage temporaire dû à des problèmes USB et ma méconnaissance d'u-boot), j'ai réussi à installer OpenWrt 24.10.5-r1 grâce à kwboot et au travail remarquable d'APCCV.

Ce fork documente mon expérience et fournit des guides détaillés de débrickage et d'installation pour aider d'autres personnes dans la même situation.

### Liens utiles

- **📁 Projet original :** [APCCV/openwrt-dns320](https://github.com/APCCV/openwrt-dns320)
- **💬 Discussion OpenWrt Forum :** [OpenWrt for D-Link DNS320 A1](https://forum.openwrt.org/t/openwrt-for-d-link-dns320-a1/224382)
- **📖 README OpenWrt officiel :** [README_OpenWRT.md](./README_OpenWRT.md)

### 📚 Documentation

#### Guides de débrickage et installation

- **🇫🇷 Guide français :** [dns320-recovery-guide-fr.md](./dns320-recovery-guide-fr.md)
- **🇬🇧 English guide:** [dns320-recovery-guide-en.md](./dns320-recovery-guide-en.md)

Ces guides couvrent :
- ✅ Comment j'ai briqué le NAS (et comment l'éviter)
- ✅ Recovery via kwboot (console série)
- ✅ Flash complet d'OpenWrt dans le NAND
- ✅ Résultats attendus à chaque étape
- ✅ Troubleshooting des problèmes USB

### 🚀 Prochaines étapes

**Mise en route du NAS :**
- Configuration réseau
- Installation des packages NAS (Samba, NFS, etc.)
- Récupération du RAID1 existant (si applicable)
- Configuration de l'interface web LuCI
- Optimisations et mise en production

Documentation à venir !

### 🙏 Remerciements

Un immense merci à **[APCCV](https://github.com/APCCV)** pour son travail exceptionnel sur le port OpenWrt DNS-320 et pour le système de recovery USB !

---

## 🇬🇧 English Version

### About this fork

This repository is a fork of the [APCCV/openwrt-dns320](https://github.com/APCCV/openwrt-dns320) project, which brings OpenWrt support to the D-Link DNS-320 Rev A.

**My approach:**

I had an old DNS-320 Rev A that had been sleeping in a bag for years. Long ago, I successfully booted Debian on it via USB, but I never flashed the NAND and didn't have time to go further.

When I discovered APCCV's project, I was very excited about flashing the entire OS into the NAND. After some adventures (including a temporary brick due to USB issues and my limited u-boot knowledge), I successfully installed OpenWrt 24.10.5-r1 thanks to kwboot and APCCV's remarkable work.

This fork documents my experience and provides detailed unbricking and installation guides to help others in the same situation.

### Useful links

- **📁 Original project:** [APCCV/openwrt-dns320](https://github.com/APCCV/openwrt-dns320)
- **💬 OpenWrt Forum discussion:** [OpenWrt for D-Link DNS320 A1](https://forum.openwrt.org/t/openwrt-for-d-link-dns320-a1/224382)
- **📖 Official OpenWrt README:** [README_OpenWRT.md](./README_OpenWRT.md)

### 📚 Documentation

#### Unbricking and installation guides

- **🇫🇷 French guide:** [dns320-recovery-guide-fr.md](./dns320-recovery-guide-fr.md)
- **🇬🇧 English guide:** [dns320-recovery-guide-en.md](./dns320-recovery-guide-en.md)

These guides cover:
- ✅ How I bricked the NAS (and how to avoid it)
- ✅ Recovery via kwboot (serial console)
- ✅ Complete OpenWrt flash to NAND
- ✅ Expected results at each step
- ✅ USB issues troubleshooting

### 🚀 Next steps

**NAS setup:**
- Network configuration
- NAS packages installation (Samba, NFS, etc.)
- Existing RAID1 recovery (if applicable)
- LuCI web interface configuration
- Optimizations and production deployment

Documentation coming soon!

### 🙏 Acknowledgments

Huge thanks to **[APCCV](https://github.com/APCCV)** for the exceptional work on the DNS-320 OpenWrt port and for the USB recovery system!

---

## 📋 License

This fork maintains the same license as the original APCCV project.

See [README_OpenWRT.md](./README_OpenWRT.md) for more details on the OpenWrt build.