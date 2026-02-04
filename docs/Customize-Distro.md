# Customize Distro

## 1. 🎨 Thème XFCE moderne + Icônes

Ajoute dans`lazy.list.chroot`

```shellscript
# Thèmes et icônes modernes
arc-theme
papirus-icon-theme
fonts-firacode
plank

```

Puis crée un hook pour appliquer automatiquement

```shellscript
nano ~/lazy-linux/config/hooks/normal/0500-configure-xfce-theme.hook.chroot
```

Contenu:

```shellscript
cd ~/lazy-linux

cat << 'EOF' | sudo tee config/hooks/normal/0500-configure-xfce-theme.hook.chroot > /dev/null
#!/bin/bash

# Configuration automatique du thème XFCE moderne

# Créer la config XFCE par défaut
mkdir -p /etc/skel/.config/xfce4/xfconf/xfce-perchannel-xml

# Fond d'écran Lazy Linux
cat > /etc/skel/.config/xfce4/xfconf/xfce-perchannel-xml/xfce4-desktop.xml << 'XMLEOF'
<?xml version="1.0" encoding="UTF-8"?>
<channel name="xfce4-desktop" version="1.0">
  <property name="backdrop" type="empty">
    <property name="screen0" type="empty">
      <property name="monitor0" type="empty">
        <property name="workspace0" type="empty">
          <property name="color-style" type="int" value="0"/>
          <property name="image-style" type="int" value="5"/>
          <property name="last-image" type="string" value="/usr/share/backgrounds/lazy-linux-wallpaper.jpg"/>
        </property>
      </property>
    </property>
  </property>
</channel>
XMLEOF

# Thème GTK moderne (Arc-Dark)
cat > /etc/skel/.config/xfce4/xfconf/xfce-perchannel-xml/xsettings.xml << 'XMLEOF'
<?xml version="1.0" encoding="UTF-8"?>
<channel name="xsettings" version="1.0">
  <property name="Net" type="empty">
    <property name="ThemeName" type="string" value="Arc-Dark"/>
    <property name="IconThemeName" type="string" value="Papirus-Dark"/>
  </property>
</channel>
XMLEOF

chmod 644 /etc/skel/.config/xfce4/xfconf/xfce-perchannel-xml/*.xml

# Appliquer aussi pour l'utilisateur live
if [ -d /home/user ]; then
    mkdir -p /home/user/.config/xfce4/xfconf/xfce-perchannel-xml
    cp /etc/skel/.config/xfce4/xfconf/xfce-perchannel-xml/* /home/user/.config/xfce4/xfconf/xfce-perchannel-xml/
    chown -R 1000:1000 /home/user/.config
fi
EOF

sudo chmod +x config/hooks/normal/0500-configure-xfce-theme.hook.chroot


```

Rends-le exécutable :

```shellscript
chmod +x ~/lazy-linux/config/hooks/normal/0500-configure-xfce-theme.hook.chroot

```

## 2. 🐚 Terminal moderne avec Zsh + Powerlevel10k (comme oh-my-posh)



Ajoute dans`lazy.list.chroot`

```shellscript
# Shell moderne
zsh
zsh-autosuggestions
zsh-syntax-highlighting
fonts-powerline

```

Puis crée un hook:

```shellscript
nano ~/lazy-linux/config/hooks/normal/0510-configure-zsh.hook.chroot

```

Contenu:

```shellscript
#!/bin/bash

# Installation automatique de Oh My Zsh et Powerlevel10k

# Installer Oh My Zsh pour tous les utilisateurs
export ZSH=/usr/share/oh-my-zsh

# Vérifier si git est installé
if command -v git >/dev/null 2>&1; then
    # Cloner Oh My Zsh
    if [ ! -d "$ZSH" ]; then
        git clone --depth=1 https://github.com/ohmyzsh/ohmyzsh.git $ZSH || true
    fi
    
    # Installer Powerlevel10k
    if [ ! -d "$ZSH/custom/themes/powerlevel10k" ]; then
        git clone --depth=1 https://github.com/romkatv/powerlevel10k.git $ZSH/custom/themes/powerlevel10k || true
    fi
fi

# Configuration par défaut pour nouveaux utilisateurs
cat > /etc/skel/.zshrc << 'EOF'
# Path to oh-my-zsh
export ZSH=/usr/share/oh-my-zsh

# Theme
ZSH_THEME="powerlevel10k/powerlevel10k"

# Plugins
plugins=(git sudo)

# Source Oh My Zsh si disponible
if [ -f $ZSH/oh-my-zsh.sh ]; then
    source $ZSH/oh-my-zsh.sh
fi

# Aliases Lazy Linux
alias update='sudo apt update && sudo apt upgrade -y'
alias ll='ls -lah --color=auto'
alias lla='ls -la --color=auto'
alias ports='sudo netstat -tulanp'
alias snano='sudo nano'
alias c='clear'
alias lazy-audit='sudo lynis audit system'
EOF

# Configurer zsh comme shell par défaut pour les nouveaux utilisateurs
# Modifie /etc/adduser.conf pour que les nouveaux users aient zsh par défaut
sed -i 's|^DSHELL=.*|DSHELL=/bin/zsh|' /etc/adduser.conf || echo "DSHELL=/bin/zsh" >> /etc/adduser.conf


```

Rends-le exécutable :

```shellscript
chmod +x ~/lazy-linux/config/hooks/normal/0510-configure-zsh.hook.chroot

```



## 3. 🖼️ Fond d'écran personnalisé



Ajoute un beau wallpaper :

```shellscript
cd ~/lazy-linux

cat > fix-xfce.sh << 'EOF'
#!/bin/bash

# Détection automatique des monitors
MONITORS=$(xfconf-query -c xfce4-desktop -l | grep -oP 'monitor[^/]+' | sort -u)

for MONITOR in $MONITORS; do
    # Configuration wallpaper pour tous les workspaces
    for WS in 0 1 2 3; do
        xfconf-query -c xfce4-desktop -p /backdrop/screen0/$MONITOR/workspace$WS/last-image -t string -s /usr/share/backgrounds/lazy-linux-wallpaper.jpg --create 2>/dev/null || \
        xfconf-query -c xfce4-desktop -p /backdrop/screen0/$MONITOR/workspace$WS/last-image -s /usr/share/backgrounds/lazy-linux-wallpaper.jpg
        
        xfconf-query -c xfce4-desktop -p /backdrop/screen0/$MONITOR/workspace$WS/image-style -t int -s 5 --create 2>/dev/null || \
        xfconf-query -c xfce4-desktop -p /backdrop/screen0/$MONITOR/workspace$WS/image-style -s 5
    done
    
    # Propriété sans workspace (pour certains monitors)
    xfconf-query -c xfce4-desktop -p /backdrop/screen0/$MONITOR/last-image -t string -s /usr/share/backgrounds/lazy-linux-wallpaper.jpg --create 2>/dev/null || \
    xfconf-query -c xfce4-desktop -p /backdrop/screen0/$MONITOR/last-image -s /usr/share/backgrounds/lazy-linux-wallpaper.jpg
done

# Thème
xfconf-query -c xsettings -p /Net/ThemeName -s "Arc-Dark" --create 2>/dev/null || xfconf-query -c xsettings -p /Net/ThemeName -s "Arc-Dark"
xfconf-query -c xsettings -p /Net/IconThemeName -s "Papirus-Dark" --create 2>/dev/null || xfconf-query -c xsettings -p /Net/IconThemeName -s "Papirus-Dark"

echo "✅ Config XFCE appliquée ! (wallpaper + thème)"
EOF

chmod +x fix-xfce.sh
sudo cp fix-xfce.sh config/includes.chroot/usr/local/bin/

```

Rebuild:

```shellscript
sudo lb clean && sudo lb build 2>&1 | tee ~/build.log

```

## Pour Executer fix-xfce.sh au 1er login:

```shellscript
cd ~/lazy-linux

# Crée le répertoire autostart
sudo mkdir -p config/includes.chroot/etc/skel/.config/autostart

# Crée le fichier autostart
sudo tee config/includes.chroot/etc/skel/.config/autostart/fix-xfce-once.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Configuration XFCE Lazy Linux
Exec=sh -c 'fix-xfce.sh && rm ~/.config/autostart/fix-xfce-once.desktop'
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
EOF

```

##

## Hook de configuration du terminal XFCE

```
sudo nano ~/lazy-linux/config/hooks/normal/0520-configure-terminal.hook.chroot
```

Contenu:

```shellscript
#!/bin/bash

# Créer le répertoire de config terminal dans /etc/skel
mkdir -p /etc/skel/.config/xfce4/terminal

# Créer la configuration du terminal avec zsh
cat > /etc/skel/.config/xfce4/terminal/terminalrc << 'TERMEOF'
[Configuration]
CommandLoginShell=TRUE
FontName=FiraCode 11
MiscAlwaysShowTabs=FALSE
MiscBell=FALSE
MiscBellUrgent=FALSE
MiscBordersDefault=TRUE
MiscCursorBlinks=FALSE
MiscCursorShape=TERMINAL_CURSOR_SHAPE_BLOCK
MiscDefaultGeometry=80x24
MiscInheritGeometry=FALSE
MiscMenubarDefault=FALSE
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
BackgroundMode=TERMINAL_BACKGROUND_TRANSPARENT
BackgroundDarkness=0.90
ScrollingBar=TERMINAL_SCROLLBAR_NONE
CustomCommand=/bin/zsh
TERMEOF

chmod 644 /etc/skel/.config/xfce4/terminal/terminalrc

```

##

## Modification du preseed pour zsh par défaut

**Fichier**:`~/lazy-linux/config/includes.installer/preseed.cfg`

```shellscript
# Changer le shell par défaut pour zsh
d-i preseed/late_command string \
    in-target bash -c "for USER in \$(ls /home 2>/dev/null); do [ -d /home/\$USER ] && chsh -s /bin/zsh \$USER || true; done"

```



## Fix pour l'écran de verrouillage au démarrage

Ajoute un nouveau hook pour désactiver le lockscreen automatique :

```shellscript
nano ~/lazy-linux/config/hooks/normal/0530-disable-lockscreen.hook.chroot

```

Contenu:

```shellscript
#!/bin/bash

# Désactiver le verrouillage automatique au démarrage

mkdir -p /etc/skel/.config/xfce4/xfconf/xfce-perchannel-xml

cat > /etc/skel/.config/xfce4/xfconf/xfce-perchannel-xml/xfce4-session.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<channel name="xfce4-session" version="1.0">
  <property name="general" type="empty">
    <property name="LockCommand" type="string" value=""/>
    <property name="SaveOnExit" type="bool" value="false"/>
  </property>
  <property name="startup" type="empty">
    <property name="screensaver" type="empty">
      <property name="enabled" type="bool" value="false"/>
    </property>
  </property>
</channel>
EOF

chmod 644 /etc/skel/.config/xfce4/xfconf/xfce-perchannel-xml/xfce4-session.xml



```

Rends-le exécutable :

```shellscript
chmod +x ~/lazy-linux/config/hooks/normal/0530-disable-lockscreen.hook.chroot

```

## Rebuild

```shellscript
cd ~/lazy-linux 
sudo lb clean --chroot 
sudo lb build 2>&1|tee ~/build.log 
```

Ces corrections devraient résoudre les 3 problèmes
