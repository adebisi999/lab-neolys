# Procédure d'installation – Lab Néolys

Procédure complète, reproductible, de la création des VM aux tests.

## 0. Prérequis

- PC hôte : Windows, 32 Go de RAM, VirtualBox 7
- ISO : `OPNsense-26.7-dvd-amd64.iso`, Windows 11 (FR, 64 bits), `debian-13.x-amd64-netinst.iso`
- ⚠️ Ne jamais extraire une ISO (avec WinRAR par exemple) : VirtualBox l'utilise telle quelle.

## 1. VM pare-feu `fw-neolys`

| Paramètre | Valeur |
|---|---|
| Type / version | BSD / FreeBSD (64-bit) |
| RAM / CPU / disque | 2048 Mo / 1 / 20 Go |
| EFI | Désactivé |
| I/O APIC | **Activé** (sinon le noyau plante au démarrage) |
| Carte 1 | NAT – Intel PRO/1000 MT Desktop |
| Carte 2 | Réseau interne `LAN-Neolys` – Intel PRO/1000 MT Desktop |
| Carte 3 | Réseau interne `DMZ-Neolys` – Intel PRO/1000 MT Desktop |

### Installation
1. Démarrer sur l'ISO, puis se connecter avec `installer` / `opnsense`.
2. Choisir le clavier, puis **Install (UFS)** sur le disque de 20 Go, et définir le mot de passe root.
3. Choisir **Halt**, puis **retirer l'ISO** (Configuration › Stockage) et mettre le **disque dur en premier** dans l'ordre d'amorçage.
4. Démarrer et vérifier que le système démarre bien sur le disque (`Root file system: /dev/gpt/rootfs`).

### Configuration en console
- **Option 1, attribution des cartes** : WAN `em0`, LAN `em1`, OPT1 `em2` (pas de LAGG, pas de VLAN).
- **Option 2, LAN** : `10.10.10.1/24`, sans passerelle ni IPv6, DHCP de `.100` à `.200`.
- **Option 2, OPT1** : `10.10.20.1/24`, DHCP de `.100` à `.200`.
- **Option 7** : `ping 9.9.9.9` doit répondre.
- **Option 6** (redémarrage) : vérifier que la configuration est conservée.

### Interface web (depuis le poste Windows)
- `https://10.10.10.1`, puis l'assistant : hostname `fw-neolys`, domaine `neolys.lan`, fuseau Europe/Paris, DNS `9.9.9.9`.
- **WAN** : décocher « Block private networks » (le WAN VirtualBox est en 10.0.2.0/24).
- **Interfaces › OPT1** : renommer en `DMZ`.
- **Services › DHCP** : vérifier que les plages du LAN et de la DMZ sont actives.

## 2. Règles du pare-feu (interface DMZ)

| # | Action | Protocole | Source | Destination | Port destination |
|---|---|---|---|---|---|
| 1 | Bloquer | TCP | DMZ net | DMZ address | 443 |
| 2 | Bloquer | TCP | DMZ net | DMZ address | 22 |
| 3 | Bloquer | any | DMZ net | LAN net | * |
| 4 | Autoriser | any | DMZ net | any | * |

Le port **source** reste toujours sur `any`.

## 3. Poste client `win11-neolys`

- Windows 11 **Professionnel** (l'édition Famille ne peut pas rejoindre un domaine AD), 4 Go de RAM, 2 CPU, 64 Go de disque, EFI, TPM 2.0 et Secure Boot.
- Réseau : Réseau interne `LAN-Neolys`.
- Compte local créé à l'installation (Maj + F10, puis `start ms-cxh:localonly`).
- Vérification : `ipconfig` doit afficher une adresse 10.10.10.x et la passerelle 10.10.10.1.

## 4. Modèle Debian, puis clones

### Modèle `debian-template` (réseau NAT pendant l'installation)
- Debian 13 netinst, sans mot de passe root (on utilise `sudo`).
- Sélection des logiciels : **décocher l'environnement de bureau**, cocher **serveur SSH** et **utilitaires usuels**.
- Préparation avant clonage :
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y openssh-server curl vim htop
sudo truncate -s 0 /etc/machine-id
sudo rm -f /etc/ssh/ssh_host_*
sudo apt clean
sudo poweroff
```

### Clones `srv-lan` et `srv-dmz`
- Clic droit › Cloner › **Générer de nouvelles adresses MAC** › Clone intégral.
- Réseau : `LAN-Neolys` pour srv-lan, `DMZ-Neolys` pour srv-dmz.
- Sur chaque clone :
```bash
sudo hostnamectl set-hostname srv-lan      # ou srv-dmz
sudo sed -i 's/debian-template/srv-lan/g' /etc/hosts
sudo systemd-machine-id-setup
sudo dpkg-reconfigure openssh-server       # régénère des clés SSH uniques
sudo reboot
```

## 5. Tests de validation

Voir le tableau des tests dans le [README](../README.md#tests-de-validation).

## 6. Snapshots

VM éteintes, prendre un snapshot `lab1-ok` pour chacune des 5 VM.
