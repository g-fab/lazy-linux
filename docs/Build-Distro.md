# Build Distro

# Guide complet : Créer une distribution Debian personnalisée avec live-build

## Vue d'ensemble

Ce guide explique comment créer **Lazy Linux**, une distribution Debian 12 (Bookworm) personnalisée avec :

* Interface graphique XFCE + LightDM
* Sécurité renforcée (AppArmor, Fail2ban, UFW)
* Tailscale pré-installé
* SSH sécurisé avec port personnalisable
* Alias et configurations pratiques

***

## Prérequis

### Machine de build

* Debian 12 (Bookworm) ou Ubuntu récent
* Au moins 4 Go de RAM
* 20 Go d'espace disque libre
* Connexion Internet stable

### Installation des outils

```shellscript
sudo apt update
sudo apt install -y live-build git debootstrap
```

# Créer le répertoire du projet

```shellscript
mkdir ~/lazy-linux
cd ~/lazy-linux
```

# Configuration initiale de live-build

```shellscript

sudo lb config \
  --distribution bookworm \
  --archive-areas "main contrib non-free non-free-firmware" \
  --debian-installer live \
  --debian-installer-gui false \
  --bootappend-live "boot=live components locales=fr_FR.UTF-8 keyboard-layouts=fr keyboard-variants=latin9" \
  --bootappend-install "locale=fr_FR.UTF-8 keymap=fr keyboard-configuration/xkb-keymap=fr" \
  --mirror-bootstrap http://deb.debian.org/debian/ \
  --mirror-binary http://deb.debian.org/debian/ \
  --apt-recommends false \
  --system normal \
  --iso-application "Lazy Linux" \
  --iso-volume "LazyLinux-1.0" \
  --image-name "lazy-linux"
```

**Explication des paramètres :**

* `--distribution bookworm`: Version Debian 12
* `--archive-areas`: Inclut les firmwares propriétaires
* `--debian-installer live`: Créer un ISO bootable avec installateur
* `--bootappend-install`: Force le clavier français dès l'installation
* `--image-name "lazy-linux"`: Nom personnalisé de l'ISO

```shellscript
~/lazy-linux/
├── config/
│   ├── hooks/normal/              # Scripts exécutés pendant le build
│   ├── package-lists/             # Listes de packages à installer
│   ├── includes.chroot/           # Fichiers copiés dans le système
│   │   └── etc/skel/.bashrc       # Configuration bash par défaut
│   ├── includes.installer/        # Configuration de l'installateur
│   │   └── preseed.cfg
│   └── includes.binary/           # Fichiers ajoutés à l'ISO
│       └── isolinux/install.cfg

```

## 3. Liste des packages

Créer`config/package-lists/lazy.list.chroot`:

```shellscript
mkdir -p config/package-lists
nano config/package-lists/lazy.list.chroot

```

Contenu:

```shellscript
# Certificats SSL (REQUIS pour dépôts HTTPS)
ca-certificates

# Essentiels système
openssh-server
curl
wget
ufw
fail2ban
htop
nano
vim
net-tools
unattended-upgrades
sudo
#docker-ce
#docker-ce-cli
#containerd.io
#docker-buildx-plugin
#docker-compose-plugin


# Gestion d'énergie et polkit (IMPORTANT pour shutdown/reboot)
policykit-1
libpam-systemd
systemd
dbus


# Sécurité
apparmor
apparmor-utils

# Environnement minimal (si tu veux un bureau léger)
xfce4
lightdm

# Locales
locales

# Serveur X (requis pour l'interface graphique)
xorg
xserver-xorg

# Audit de sécurité
lynis


# Monitoring et diagnostic système
glances
smartmontools
lm-sensors

# Outils réseau
nmap
tcpdump
wireshark
iperf3
mtr-tiny

# Backup et synchronisation
rsync
rclone

# Terminaux alternatifs (optionnel - pour tester)
terminator


# Thèmes et icônes modernes
arc-theme
papirus-icon-theme
fonts-firacode
plank

# Shell moderne
zsh
zsh-autosuggestions
zsh-syntax-highlighting
fonts-powerline
git
```

## 4. Hooks de configuration

## Hook 1 : Configuration Sudo

`config/hooks/normal/0060-configure-sudo.hook.chroot`:

```shellscript
#!/bin/bash

# Créer le groupe sudo s'il n'existe pas
groupadd -f sudo

# Configurer sudoers pour le groupe sudo
echo '%sudo ALL=(ALL:ALL) ALL' > /etc/sudoers.d/sudo-group
chmod 0440 /etc/sudoers.d/sudo-group

```

```shellscript
chmod +x config/hooks/normal/0060-configure-sudo.hook.chroot

```

## Hook 2 : Sécurité (AppArmor + UFW)

`config/hooks/normal/0100-configure-security.hook.chroot`:

```shellscript
#!/bin/bash

# Activer AppArmor
systemctl enable apparmor

# Configuration SSH sécurisée
cat >> /etc/ssh/sshd_config <<EOF

# Configuration de sécurité Lazy Linux
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication yes
PermitEmptyPasswords no
X11Forwarding no
MaxAuthTries 5
LoginGraceTime 2m
StrictModes yes
EOF

# Configurer et activer UFW par défaut
ufw default deny incoming
ufw default allow outgoing
ufw limit 22/tcp comment 'SSH rate limit'
ufw --force enable
systemctl enable ufw

```

```shellscript
chmod +x config/hooks/normal/0100-configure-security.hook.chroot

```

## Hook 3 : Message d'accueil + Script de configuration

`config/hooks/normal/0200-welcome-message.hook.chroot`:

```shellscript
#!/bin/bash

# MOTD personnalisé
cat > /etc/motd <<'EOF'
   _                    _     _                 
  | |    __ _ _____   _| |   (_)_ __  _   ___  __
  | |   / _` |_  / | | | |   | | '_ \| | | \ \/ /
  | |__| (_| |/ /| |_| | |___| | | | | |_| |>  < 
  |_____\__,_/___|\__, |_____|_|_| |_|\__,_/_/\_\
                  |___/                          
                  
EOF

# Script d'onboarding première connexion
cat > /etc/profile.d/lazy-welcome.sh <<'EOF'
#!/bin/bash

WELCOME_FILE="$HOME/.lazy-welcome-shown"

if [ ! -f "$WELCOME_FILE" ]; then
    echo "========================================"
    echo "     Bienvenue sur Lazy Linux 1.0"
    echo "========================================"
    echo ""
    echo "Quelques commandes utiles :"
    echo ""
    echo "  -  Générer une clé SSH :"
    echo "      ssh-keygen -t ed25519 -C \"votre_email\""
    echo ""
    echo "  -  Copier la clé sur un serveur :"
    echo "      ssh-copy-id -p <port> \$USER@<ip>"
    echo ""
    echo "  -  Configurer le firewall :"
    echo "      sudo ufw allow <votre_port_ssh>/tcp"
    echo "      sudo ufw enable"
    echo ""
    echo "  -  Vérifier les connexions actives :"
    echo "      ports                     (alias configuré)"
    echo ""
    echo "  -  Mettre à jour le système :"
    echo "      update                    (alias configuré)"
    echo ""
    echo "  -  Autres alias disponibles :"
    echo "      lla, c, .., ..., snano"
    echo "      (tapez 'alias' pour voir la liste complète)"
    echo ""
    echo "  -  Reconfigurer Lazy Linux :"
    echo "      sudo lazy-setup"
    echo ""
    echo "========================================"
    echo ""
    
    touch "$WELCOME_FILE"
fi
EOF

chmod +x /etc/profile.d/lazy-welcome.sh

# Script de configuration post-installation
cat > /usr/local/bin/lazy-setup <<'EOF'
#!/bin/bash

if [ "$EUID" -ne 0 ]; then
    echo "Veuillez exécuter en tant que root (sudo)"
    exit 1
fi

echo "=== Configuration Lazy Linux ==="
echo ""

# Changer le port SSH
read -p "Nouveau port SSH (22 par défaut) : " SSH_PORT
SSH_PORT=${SSH_PORT:-22}

sed -i "s/^#\?Port .*/Port $SSH_PORT/" /etc/ssh/sshd_config
echo "✓ Port SSH configuré : $SSH_PORT"

# Ajouter règle firewall
ufw allow $SSH_PORT/tcp
echo "✓ Règle firewall ajoutée"

# Redémarrer SSH
systemctl restart sshd
echo "✓ Service SSH redémarré"

echo ""
echo "Configuration terminée !"
echo "N'oubliez pas d'activer ufw : sudo ufw enable"
EOF

chmod +x /usr/local/bin/lazy-setup

```

```shellscript
chmod +x config/hooks/normal/0200-welcome-message.hook.chroot

```

## Hook 4 : Suppression des bloatware

`config/hooks/normal/0300-remove-bloat.hook.chroot`:

```shellscript
#!/bin/bash

# Supprimer les packages inutiles
apt-get purge -y \
    libreoffice* \
    gnome-games* \
    aisleriot \
    gnome-mahjongg \
    gnome-mines \
    gnome-sudoku \
    quadrapassel \
    gnome-calculator 2>/dev/null || true

# Nettoyer
apt-get autoremove -y
apt-get clean

```

```shellscript
chmod +x config/hooks/normal/0300-remove-bloat.hook.chroot

```

## Hook 5 : Hook pour générer les locales

```shellscript
nano ~/lazy-linux/config/hooks/normal/0350-generate-locales.hook.chroot
```

Contenu :

```shellscript
#!/bin/bash

# S'assurer que locales est installé
apt-get install -y locales

# Configurer les locales françaises
echo "fr_FR.UTF-8 UTF-8" > /etc/locale.gen
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen

# Générer les locales (chemin complet)
/usr/sbin/locale-gen

# Définir la locale par défaut
update-locale LANG=fr_FR.UTF-8
echo 'LANG=fr_FR.UTF-8' > /etc/default/locale
echo 'LC_ALL=fr_FR.UTF-8' >> /etc/default/locale

echo "Locales générées et configurées"


```

```shellscript
chmod +x ~/lazy-linux/config/hooks/normal/0350-generate-locales.hook.chroot

```

## Hook 6 : Activer LightDM (interface graphique)

`config/hooks/normal/0400-enable-lightdm.hook.chroot`:

```shellscript
#!/bin/bash

# Vérifier si LightDM est installé
if dpkg -l | grep -q lightdm; then
    # S'assurer que les locales sont générées (chemin complet)
    /usr/sbin/locale-gen fr_FR.UTF-8 2>/dev/null || true
    
    # Activer LightDM au démarrage
    systemctl enable lightdm
    
    # Définir le target graphique par défaut
    systemctl set-default graphical.target
    
    echo "LightDM configuré et activé"
else
    echo "LightDM n'est pas installé, skip"
fi


```

```shellscript
chmod +x config/hooks/normal/0400-enable-lightdm.hook.chroot

```

##

## Hook  7 : Configuration Wireshark

```shellscript
mkdir -p ~/lazy-linux/config/hooks/normal
nano ~/lazy-linux/config/hooks/normal/0360-configure-wireshark.hook.chroot

```

Contenu :

```shellscript
#!/bin/sh
# Configuration automatique de Wireshark pour utilisation non-root

# Ajouter le groupe wireshark si nécessaire
groupadd -f wireshark

# Permettre aux membres du groupe wireshark de capturer les paquets
echo "wireshark-common wireshark-common/install-setuid boolean true" | debconf-set-selections

# Reconfigurer wireshark
DEBIAN_FRONTEND=noninteractive dpkg-reconfigure wireshark-common

# Note: L'utilisateur devra être ajouté au groupe wireshark après installation
# avec: sudo usermod -aG wireshark $USER

```

Rends-le exécutable :

```shellscript
chmod +x ~/lazy-linux/config/hooks/normal/0360-configure-wireshark.hook.chroot

```

## Hook 8 : Configuration lm-sensors

```shellscript
nano ~/lazy-linux/config/hooks/normal/0370-configure-lm-sensors.hook.chroot

```

Contenu :

```shellscript
#!/bin/sh
# Configuration automatique de lm-sensors

# Créer un fichier de configuration par défaut
# sensors-detect sera lancé par l'utilisateur si besoin

# Ajouter un alias pour faciliter la configuration
cat >> /etc/skel/.bashrc << 'EOF'

# Alias pour configurer les capteurs de température
alias sensors-setup='sudo sensors-detect --auto'
EOF

```

Rends-le exécutable :

```shellscript
chmod +x ~/lazy-linux/config/hooks/normal/0370-configure-lm-sensors.hook.chroot

```

## Hook 9 : Message post-installation pour l'utilisateur

```shellscript
nano ~/lazy-linux/config/hooks/normal/0380-post-install-info.hook.chroot
```

Contenu :

```shellscript
#!/bin/sh
# Ajouter des informations post-installation

cat >> /etc/profile.d/lazy-postinstall.sh << 'EOF'
# Vérifier si l'utilisateur doit configurer certains outils

if [ -f ~/.lazy-first-run ]; then
    return
fi

# Créer le marqueur pour ne pas afficher à chaque fois
touch ~/.lazy-first-run

echo ""
echo "📋 Configuration post-installation recommandée:"
echo "   • Wireshark: sudo usermod -aG wireshark \$USER && newgrp wireshark"
echo "   • Sensors:   sensors-setup (alias pour sensors-detect --auto)"
echo ""
EOF

chmod +x /etc/profile.d/lazy-postinstall.sh

```

Rends-le exécutable :

```shellscript
chmod +x ~/lazy-linux/config/hooks/normal/0380-post-install-info.hook.chroot

```

##

##


Pour le build : Configurer AZERTY par défaut
--------------------------------------------

Il faut créer un fichier de configuration X11 :

```shellscript
mkdir -p ~/lazy-linux/config/includes.chroot/etc/X11/xorg.conf.d
nano ~/lazy-linux/config/includes.chroot/etc/X11/xorg.conf.d/00-keyboard.conf

```

Contenu :

```shellscript
Section "InputClass"
    Identifier "keyboard-all"
    MatchIsKeyboard "on"
    Option "XkbLayout" "fr"
    Option "XkbVariant" "latin9"
EndSection

```

## 5. Configuration Bash personnalisée

`config/includes.chroot/etc/skel/.bashrc`:

```shellscript
mkdir -p config/includes.chroot/etc/skel
nano config/includes.chroot/etc/skel/.bashrc

```

Contenu :

```shellscript
# ~/.bashrc: executed by bash(1) for non-login shells.

# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
esac

# History settings
HISTCONTROL=ignoreboth
HISTSIZE=1000
HISTFILESIZE=2000
shopt -s histappend

# Check window size after each command
shopt -s checkwinsize

# Enable color support
if [ -x /usr/bin/dircolors ]; then
    test -r ~/.dircolors && eval "$(dircolors -b ~/.dircolors)" || eval "$(dircolors -b)"
    alias ls='ls --color=auto'
    alias grep='grep --color=auto'
    alias fgrep='fgrep --color=auto'
    alias egrep='egrep --color=auto'
fi

# Lazy Linux custom aliases
alias lla='ls -la'
alias nano="nano -l"
alias snano="sudo nano -l"
alias c="clear"
alias ..="cd .."
alias ...="cd ../.."
alias update="sudo apt update && sudo apt upgrade -y"
alias ports="sudo ss -tunlp"

# Prompt customization
PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '

# Enable programmable completion features
if ! shopt -oq posix; then
  if [ -f /usr/share/bash-completion/bash_completion ]; then
    . /usr/share/bash-completion/bash_completion
  elif [ -f /etc/bash_completion ]; then
    . /etc/bash_completion
  fi
fi
# Charger /etc/profile.d/ même en shell non-login
if [ -d /etc/profile.d ]; then
  for i in /etc/profile.d/*.sh; do
    if [ -r $i ]; then
      . $i
    fi
  done
  unset i
fi

```

## Pour le build : Configurer xfce4-terminal par défaut

Créer la config du terminal :

```shellscript
mkdir -p ~/lazy-linux/config/includes.chroot/etc/skel/.config/xfce4/terminal
nano ~/lazy-linux/config/includes.chroot/etc/skel/.config/xfce4/terminal/terminalrc

```

Contenu :

```
[Configuration]
MiscAlwaysShowTabs=FALSE
MiscBell=FALSE
MiscBellUrgent=FALSE
MiscBordersDefault=TRUE
MiscCursorBlinks=FALSE
MiscCursorShape=TERMINAL_CURSOR_SHAPE_BLOCK
MiscDefaultGeometry=80x24
MiscInheritGeometry=FALSE
MiscMenubarDefault=TRUE
MiscMouseAutohide=FALSE
MiscMouseWheelZoom=TRUE
MiscToolbarDefault=FALSE
MiscConfirmClose=TRUE
MiscCycleTabs=TRUE
MiscTabCloseButtons=TRUE
MiscTabCloseMiddleClick=TRUE
MiscTabPosition=GTK_POS_TOP
MiscHighlightUrls=TRUE
MiscMiddleClickOpensUri=FALSE
MiscCopyOnSelect=FALSE
MiscShowRelaunchDialog=TRUE
MiscRewrapOnResize=TRUE
MiscUseShiftArrowsToScroll=FALSE
MiscSlimTabs=FALSE
MiscNewTabAdjacent=FALSE
MiscSearchDialogOpacity=100
MiscShowUnsafePasteDialog=TRUE
CommandLoginShell=TRUE

```

La ligne clé :**`CommandLoginShell=TRUE`**&#x200B;

Comme ça, le terminal lancera toujours un login shell et le message dans`.bash_profile`s'affichera ! ✅

## Fix  : Welcome message

Le problème :`uxterm`n'est**pas**un login shell non plus. Il faut utiliser**xfce4-terminal**à la place.
**Créer la config xfce4-terminal**(login shell activé) :

```shellscript
mkdir -p ~/lazy-linux/config/includes.chroot/etc/skel/.config/xfce4/terminal
nano ~/lazy-linux/config/includes.chroot/etc/skel/.config/xfce4/terminal/terminalrc

```

Contenu :

```shellscript
[Configuration]
CommandLoginShell=TRUE
FontName=Monospace 10
MiscAlwaysShowTabs=FALSE
MiscBell=FALSE
MiscCursorBlinks=FALSE
MiscDefaultGeometry=80x24
MiscInheritGeometry=FALSE
MiscMenubarDefault=FALSE
MiscMouseAutohide=FALSE
MiscToolbarDefault=FALSE
MiscConfirmClose=TRUE
MiscCycleTabs=TRUE
MiscTabCloseButtons=TRUE
MiscTabPosition=GTK_POS_TOP
MiscHighlightUrls=TRUE
ScrollingBar=TERMINAL_SCROLLBAR_NONE

```

**B. Ajouter xfce4-terminal au lanceur du panel**:

```shellscript
mkdir -p ~/lazy-linux/config/includes.chroot/etc/skel/.config/xfce4/panel
nano ~/lazy-linux/config/includes.chroot/etc/skel/.config/xfce4/panel/launcher-15/16823974972.desktop

```

Contenu :

```shellscript
[Desktop Entry]
Version=1.0
Type=Application
Name=Terminal
Comment=Émulateur de terminal
TryExec=xfce4-terminal
Exec=xfce4-terminal
Icon=utilities-terminal
Categories=System;TerminalEmulator;
StartupNotify=true

```



Autoriser l'utilisateur à éteindre/redémarrer sans sudo
Crée un fichier de règles PolicyKit :

```shellscript
# Créer le bon fichier .pkla
sudo bash -c 'cat > ~/lazy-linux/config/includes.chroot/etc/polkit-1/localauthority/50-local.d/10-shutdown.pkla << "EOF"
[Allow Shutdown and Reboot]
Identity=unix-user:*
Action=org.freedesktop.login1.*
ResultActive=yes
ResultInactive=yes
ResultAny=yes
EOF'

```

Vérifier le contenu:

```shellscript
cat ~/lazy-linux/config/includes.chroot/etc/polkit-1/localauthority/50-local.d/10-shutdown.pkla
```

## 6. Configuration de l'installateur (Preseed)

`config/includes.installer/preseed.cfg`:

```shellscript
mkdir -p config/includes.installer
nano config/includes.installer/preseed.cfg

```

Contenu :

```shellscript
# Éviter les erreurs de montage au démarrage
d-i preseed/early_command string umount /media || true

# Configuration clavier forcée
d-i debian-installer/locale string fr_FR.UTF-8
d-i debian-installer/language string fr
d-i debian-installer/country string FR
d-i keyboard-configuration/xkb-keymap select fr
d-i keyboard-configuration/layoutcode string fr
d-i keyboard-configuration/variantcode string latin9
d-i keyboard-configuration/modelcode string pc105
d-i console-setup/ask_detect boolean false
d-i localechooser/supported-locales multiselect fr_FR.UTF-8

#### Réseau (automatique)
d-i netcfg/choose_interface select auto
d-i netcfg/get_hostname string lazy-linux
d-i netcfg/get_domain string local

#### Miroir Debian (AUTOMATIQUE - pas de question)
d-i mirror/country string manual
d-i mirror/http/hostname string deb.debian.org
d-i mirror/http/directory string /debian
d-i mirror/http/proxy string
d-i apt-setup/use_mirror boolean true
d-i apt-setup/services-select multiselect security, updates
d-i apt-setup/security_host string security.debian.org

#### Compte root DÉSACTIVÉ (pas de mot de passe root demandé)
d-i passwd/root-login boolean false

#### Utilisateur (nom suggéré, modifiable)
d-i passwd/make-user boolean true
#d-i passwd/user-fullname string Administrateur
#d-i passwd/username string fab
d-i passwd/user-default-groups string sudo,cdrom,floppy,audio,dip,video,plugdev,netdev


#### Horloge et fuseau horaire (automatique)
d-i clock-setup/utc boolean true
d-i time/zone string Europe/Paris
d-i clock-setup/ntp boolean true

#### Partitionnement (AVEC CHOIX - question posée)
# NE PAS préconfigurer la méthode ni le disque pour laisser le choix à l'utilisateur
#d-i partman-auto/disk string /dev/sda
#d-i partman-auto/method string regular
#d-i partman-auto/choose_recipe select atomic

# Ces lignes restent pour nettoyer les anciennes configs
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true

# Ces confirmations restent pour éviter les multiples validations
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true
d-i partman/confirm_write_new_label boolean true

#### GRUB (automatique)
d-i grub-installer/only_debian boolean true
d-i grub-installer/with_other_os boolean true
d-i grub-installer/bootdev string default

#### Tasksel et packages (skip les questions)
tasksel tasksel/first multiselect
d-i pkgsel/include string
d-i pkgsel/upgrade select full-upgrade
popularity-contest popularity-contest/participate boolean false

#### Fin de l'installation (reboot automatique)
d-i finish-install/reboot_in_progress note


# Changer le shell par défaut pour zsh
d-i preseed/late_command string \
    in-target bash -c "for USER in \$(ls /home 2>/dev/null); do [ -d /home/\$USER ] && chsh -s /bin/zsh \$USER || true; done"


```

##

##

## 8. Build de l'ISO

## Build complet

```shellscript
cd ~/lazy-linux
sudo lb build 2>&1 | tee ~/build.log

```

**Durée estimée**: 30-40 minutes

## Résultat

L'ISO sera disponible ici :

```shellscript
~/lazy-linux/lazy-linux-amd64.hybrid.iso

```

## 9. Modifications et rebuild

## Modifier un hook, un package, ou un fichier

```shellscript
# Exemple : Modifier un hook
nano config/hooks/normal/0100-configure-security.hook.chroot

# Rebuild partiel (plus rapide, ~5-10 min)
sudo lb clean --binary
sudo lb build 2>&1 | tee ~/build.log

```

## Changement majeur de configuration



**Rappel important**: Quand tu supprimes un fichier comme`install.cfg`ou que tu modifies le preseed, il faut TOUJOURS faire un`--purge`ou supprimer`.build/`manuellement, sinon live-build ne reconstruit pas correctement le chroot.



```shellscript
# Supprimer complètement l'environnement
sudo rm -rf .build/ cache/ chroot/ *.iso* *.contents *.files *.packages *.zsync

# Refaire la config
sudo lb config \
  --distribution bookworm \
  --archive-areas "main contrib non-free non-free-firmware" \
  --debian-installer live \
  --debian-installer-gui false \
  --bootappend-live "boot=live components locales=fr_FR.UTF-8 keyboard-layouts=fr keyboard-variants=latin9" \
  --bootappend-install "locale=fr_FR.UTF-8 keymap=fr keyboard-configuration/xkb-keymap=fr" \
  --mirror-bootstrap http://deb.debian.org/debian/ \
  --mirror-binary http://deb.debian.org/debian/ \
  --apt-recommends false \
  --system normal \
  --iso-application "Lazy Linux" \
  --iso-volume "LazyLinux-1.0" \
  --image-name "lazy-linux"

# Rebuild complet (~30-40 min)
sudo lb build 2>&1 | tee ~/build.log

```

##



##

## Types de clean




| **Commande**        | **Ce qui est supprimé** | **Durée rebuild** | **Usage**                  |
| ------------------- | ----------------------- | ----------------- | -------------------------- |
| `lb clean --binary` | ISO + binaires          | 5-10 min          | Modification hooks/configs |
| `lb clean --purge`  | Tout sauf config/       | 30-40 min         | Changement packages majeur |
| `rm -rf .build/`    | Réinit complète         | 30-40 min         | Problème de build          |

## 10. Test de l'ISO

## Dans VirtualBox/VMware

1. Créer une nouvelle VM
2. Monter l'ISO`lazy-linux-amd64.hybrid.iso`
3. Booter et choisir "Start installer"
4. Suivre l'installation
5. Au premier boot, vérifier :
   * XFCE démarre automatiquement
   * Le message d'accueil s'affiche en console
   * Les alias fonctionnent (`update`,`ports`, etc.)
   * `sudo lazy-setup`permet de configurer le port SSH

## Fonctionnalités de Lazy Linux

## Sécurité

* ✅ AppArmor activé par défaut
* ✅ Fail2ban pré-configuré
* ✅ UFW activé avec règle SSH rate-limited
* ✅ SSH sécurisé (pas de root, port personnalisable)
* ✅ Mises à jour automatiques (unattended-upgrades)

## Réseau

* ✅ Tailscale pré-installé et configuré
* ✅ Net-tools pour diagnostics réseau

## Interface

* ✅ XFCE léger et rapide
* ✅ LightDM comme gestionnaire de connexion
* ✅ Pas de bloatware

## Productivité

* ✅ Alias bash pratiques
* ✅ Script`lazy-setup`pour configuration rapide
* ✅ Message d'accueil informatif
* ✅ Htop pour monitoring

## Localisation

* ✅ Français par défaut (locale, clavier)
* ✅ Timezone Europe/Paris





## Dépannage

## Erreur "config stage required"

```
sudorm -rf .build/ sudo lb config [...]sudo lb build 
```

## Le clavier reste en QWERTY

Vérifier que`config/includes.binary/isolinux/install.cfg`existe avec le paramètre`keymap=fr`.

## XFCE ne démarre pas

Vérifier que :

1. `xfce4`et`lightdm`sont dans`lazy.list.chroot`
2. Le hook`0400-enable-lightdm.hook.chroot`est exécutable

## T

Parfait ! On va intégrer Lynis avec l'option 2. Voici ce qu'il faut faire :

## 1. Créer le hook Lynis

```shellscript
nano ~/lazy-linux/config/hooks/normal/0150-install-lynis.hook.chroot 
```

Contenu :

```shellscript
#!/bin/bash

# Installer Lynis
apt-get install -y lynis

# Créer un script pour l'audit de sécurité
cat > /usr/local/bin/lazy-audit <<'EOF'
#!/bin/bash

echo "========================================="
echo "  Audit de sécurité Lazy Linux"
echo "========================================="
echo ""
echo "Lynis va analyser votre système et proposer"
echo "des recommandations de sécurité."
echo ""
read -p "Lancer l'audit maintenant ? (o/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Oo]$ ]]; then
    sudo lynis audit system
    echo ""
    echo "Rapport complet : /var/log/lynis.log"
    echo "Recommandations : /var/log/lynis-report.dat"
fi
EOF

chmod +x /usr/local/bin/lazy-audit

```

Rendre exécutable :

```shellscript
chmod +x ~/lazy-linux/config/hooks/normal/0150-install-lynis.hook.chroot 
```

## 2. Mettre à jour le .bashrc pour ajouter l'alias

```shellscript
nano ~/lazy-linux/config/includes.chroot/etc/skel/.bashrc 
```

Ajoute cette ligne dans la section des alias (après`alias ports=...`) :

```shellscript
alias lazy-audit="sudo /usr/local/bin/lazy-audit"
```

## 3. Modifier le message d'accueil

```shellscript
nano ~/lazy-linux/config/hooks/normal/0200-welcome-message.hook.chroot 
```

Dans la section du message d'accueil, ajoute après la ligne "Reconfigurer Lazy Linux" :

```shellscript
8 Commandes utiles :
19   • sudo lazy-setup   → Configuration système (port SSH, firewall)
20   • update            → Mise à jour système
21   • ll / lla          → Listage détaillé
22   • ports             → Voir les ports ouverts
23   • snano             → Nano en sudo
24   • c                 → Clear Terminal
25   • lazy-audit        → Auditer la sécurité du système
```



Voilà ! Une fois le build terminé, l'utilisateur verra le message d'accueil avec`lazy-audit`et pourra facilement lancer l'audit de sécurité. 🔒✨

Lynis va vérifier :

* UFW, Fail2ban, AppArmor
* Configuration SSH
* Mises à jour du système
* Permissions fichiers
* Et plein d'autres checks de sécurité !

Le build est toujours en cours ?
