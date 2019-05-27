


# ArchLinux sous VMWare

## Création de la VM

1. Créer la VM sur l'hyperviseur
![Interface de VMWare en Server](https://i.imgur.com/0bMDkuh.png)
2. Créer la machine en Custom en targetant la version actuelle de VMWare Workstation
3. En Guest OS, sélectionnez `Other 3.x or later Linux (64-bit)`
4. Nommez votre machine
5. Sélectionnez `UEFI` en Firmware
6. Attribuez la RAM et le CPU selon votre convenance  (Nous allons prendre 2 core / 2GB pour l'exemple)
7. En configuration Network, sélectionnez `NAT` (Ou `Bridged` dans le cas d'un Failover) : 
![Config Network](https://i.imgur.com/eGdaDtC.png)
8. En contrôleur SCSI et Virtual Disk, laissez `LSI Logic` et `SCSI` 
9. Créer votre disque virtuel sans allouer directement tout le disque (Nous allons prendre 250GB pour l'exemple)
10. Avant de finir la préparation cliquez sur <kbd>Customize Hardware</kbd>  
11. Monter l'ISO de Arch:
![ISO](https://i.imgur.com/sgaqk95.png)

## Configuration Failover - Partie 1

*Concerne uniquement le cas d'une configuration IP failover*

Si vous avez une IP Failover que vous souhaitez dédié à la VM:
1. Attribuez une adresse MAC sur l'interface client Online.
2. Attribuez l'adresse MAC à la VM dans les paramètres et sélectionnez le mode Bridged si ce n'est pas déjà fait
![Config VMWare Failover](https://i.imgur.com/YAnwB3S.png)
3. Démarrez la VM
4. Passez votre clavier en français avec `loadkeys fr`
5. Entrez les commandes suivantes (en remplacant `FAILOVERIP` par votre IP Failover et ens32 par votre interface):
```BASH
ip link set ens32 up
ip addr add FAILOVERIP/32 broadcast FAILOVERIP dev ens32
ip route add FAILOVERIP dev ens32 proto kernel scope link src FAILOVERIP metric 100
ip route add 62.210.0.1 dev ens32 proto static scope link metric 100
ip route add default via 62.210.0.1 dev ens32 proto static metric 100
echo "nameserver 62.210.16.6" >> /etc/resolv.conf
echo "nameserver 62.210.16.7" >> /etc/resolv.conf
```
6. Testez de `ping google.fr` ( <kbd>Ctrl</kbd> + <kbd>C</kbd> pour arrêter ) 

## Setup de base

1. Démarrez la VM et n'oubliez pas d'activer le <kbd>Verr. Num</kbd> dans la VM
2. Passez votre clavier en français avec `loadkeys fr`
3. Préparez votre disque dur avec `cgdisk /dev/sda`
	* Appuyez sur <kbd>New</kbd>
	* Laissez vide le First Sector
	* Tapez `512M` pour le Size
	* Tapez `ef00` pour l'hex code de la partition (Afin de la définir comme EFI System)
	* Donnez lui `EFI` comme nom et validez
 --
	* Descendez vers l'espace libre de 249.5 GB et sélectionnez sur <kbd>New</kbd>
	* Laissez vide le First Sector
	* Tapez `240G` pour le Size
	* Tapez `8300` pour l'hex code de la partition (Afin de la définir comme Linux System)
	* Donnez lui `LINUX` comme nom
 --
	* Descendez vers l'espace libre de 249.5 GB et sélectionnez sur <kbd>New</kbd>
	* Laissez vide le First Sector
	* Laissez vide le Size
	* Tapez `8200` pour l'hex code de la partition (Afin de la définir comme Linux Swap)
	* Donnez lui `SWAP` comme nom
--
	* Le résultat final devrait ressembler à ça:   ![cgdisk](https://i.imgur.com/8WwOM46.png)
	* Si oui cliquez sur <kbd>Write</kbd>, tapez `yes` et cliquez sur <kbd>Quit</kbd>
4) Maintenant formatons les disques:
	* `mkfs.fat -F32 /dev/sda1`
	* `mkfs.ext4 /dev/sda2`
	* `mkswap /dev/sda3`
	* `swapon /dev/sda3`
5) Montons les disques:
	* `mount /dev/sda2 /mnt`
	* `mkdir /mnt/boot`
	* `mkdir /mnt/home`
	* `mount /dev/sda1 /mnt/boot`
6) Éditez la liste de miroir avec la commande suivante : `nano /etc/pacman.d/mirrorlist`, allez jusqu'à trouver un serveur français, couper la ligne (<kbd>Ctrl</kbd> + <kbd>K</kbd>) et coller là en haut (<kbd>Ctrl</kbd> + <kbd>U</kbd>). Quitter le fichier en enregistrant
7) Lancer la première phase d'installation avec la commande suivante:
`pacstrap /mnt base base-devel zip unzip p7zip vim mc alsa-utils syslog-ng mtools dosfstools lsb-release zsh grub os-prober efibootmgr netctl open-vm-tools openssh wget`
8) Générez la table de partitionnement système avec : `genfstab -U -p /mnt >> /mnt/etc/fstab`

## Chroot du système

1) Utilisez `arch-chroot /mnt` pour chroot le système
2) Éditez /etc/vconsole.conf
```
KEYMAP=fr-latin9
FONT=lat9w-16
```
3) `echo LANG=fr_FR.UTF-8 > /etc/locale.conf`
4) `sed -i 's/#fr_FR.UTF/fr_FR.UTF/' /etc/locale.gen` (Juste pour décommentez la ligne FR UTF-8)
5) `locale-gen` 
![locale-gen](https://i.imgur.com/i5tD32B.png)
6) `export LANG=fr_FR.UTF-8`
7) `ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime`
8) `hwclock --systohc --utc`
9) On définit l'hostname avec `echo "NomDeLaMachineXD" > /etc/hostname`
10) On crée les images d'init: `mkinitcpio -p linux`
11) On crée l'image EFI de GRUB: `grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch_grub --recheck`
12) On crée le fichier de config de GRUB: `grub-mkconfig -o /boot/grub/grub.cfg`
13) On met l'image par défaut: `mkdir /boot/EFI/boot`
`cp /boot/EFI/arch_grub/grubx64.efi /boot/EFI/boot/bootx64.efi`
14) On édite le fichier /etc/pacman.conf en ajoutant à la fin
```
[archlinuxcn]
Server = https://cdn.repo.archlinuxcn.org/$arch
```
16) `pacman -Syu archlinuxcn-keyring yaourt ntp cronie git` (Ajoutez `networkmanager` si vous n'utilisez pas de failover)
17) `sed -i 's/#ForwardToSyslog=no/ForwardToSyslog=yes/' /etc/systemd/journald.conf`
18) On active les services:
```
systemctl enable syslog-ng@default 
systemctl enable vmtoolsd.service
systemctl enable cronie 
systemctl enable ntpd 
systemctl enable sshd
systemctl enable NetworkManager
```
19) On définie un mot de passe root: `passwd`
20) On crée un utilisateur: `useradd -m -g wheel -c 'Nom complet' -s /bin/zsh pseudo`
21) On définit un mot de passe à l'utilisateur: `passwd pseudo`
22) On utilise `visudo`, on descend jusqu'à la ligne `# %wheel ALL=(ALL) ALL` et on appuie sur <kbd>Suppr</kbd>, puis on tape `:wq` et on appuie sur  <kbd>Entrée</kbd> pour quitter vim
23) `ln -s /dev/null /etc/udev/rules.d/80-net-setup-link.rules` pour retourner aux anciens noms d'interface
24) On tape `exit` pour quitter le chroot puis `umount -R /mnt` pour démonter les disques.
25) Enfin, on `reboot`

## Configuration Failover - Partie 2

*Concerne uniquement le cas d'une configuration IP failover*
1) Connectez vous en root et écrivez ce fichier dans `/etc/systemd/system/network.service`
```
[Unit]
Description=IP Failover Config
Wants=network.target
Before=network.target
BindsTo=sys-subsystem-net-devices-eth0.device
After=sys-subsystem-net-devices-eth0.device

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/ip link set eth0 up
ExecStart=/sbin/ip addr add IPFAILOVER/32 broadcast IPFAILOVER dev eth0
ExecStart=/sbin/ip route add IPFAILOVER dev eth0 proto kernel scope link src IPFAILOVER metric 100
ExecStart=/sbin/ip route add 62.210.0.1 dev eth0 proto static scope link metric 100
ExecStart=/sbin/ip route add default via 62.210.0.1 dev eth0 proto static metric 100
ExecStop=/sbin/ip addr flush dev eth0
ExecStop=/sbin/ip route flush dev eth0
ExecStop=/sbin/ip link set eth0 down

[Install]
WantedBy=multi-user.target
```
2)  `echo "nameserver 62.210.16.6" >> /etc/resolv.conf`
`echo "nameserver 62.210.16.7" >> /etc/resolv.conf`
3) `systemctl enable network`
4) `reboot` 

## Configuration utilisateur

1) Appuyez sur <kbd>0</kbd> pour quitter la configuration de .zshrc
2) `sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"`
3) `wget https://files.inori.moe/ressource/.zshrc -o .zshrc`
4) `mv .zshrc.1 .zshrc`
5) `sudo pacman -S byobu`
6) `byobu-enable`
7) `sudo reboot`

## Configuration sshd

1) Générer vos clés privés en local.
2) Ouvrir les ports sur le VMNet Manager
3) FeelAutistMan
