"# openwrtnotificator" 

PreRequis
 - installer curl

Etape 1 - Configurer un BOT Telegram

Etape 2 - Recupérer le chatid et le token

Etape 3 - Stocker dans un fichier les adresses MAC, IP, hostname des devices connectés

Principe
OpenWrt déclenche des scripts dans /etc/hotplug.d/hostapd/ à chaque événement d’association/déassociation Wi‑Fi, en passant des variables d’environnement comme ACTION, MAC, IFNAME, etc.​
En ajoutant un script dans ce répertoire, tu peux utiliser logger pour écrire un message personnalisé dans le log système.

Crée le fichier /etc/hotplug.d/dhcp/10-mac-hostname-db avec ce contenu :

    #!/bin/sh

    # On traite les nouveaux baux et les mises à jour
    case "$ACTION" in
        add|update)
            ;;
        *)
            exit 0
            ;;
    esac

    MAC="$MACADDR"
    HOST="$HOSTNAME"    # ou DHCP_HOSTNAME selon ce que tu as vu dans /tmp/dhcp-debug.log
    DHCP_HOSTNAME="$DHCP_HOSTNAME"
    IP="$IPADDR"

    [ -z "$MAC" ] && exit 0

    DB="/tmp/mac-hostname.db"

    [ -z "$HOST" ] && HOST="unknown"

    # On remplace l `^yancienne ligne pour cette MAC
    grep -v "^$MAC " "$DB" 2>/dev/null > "$DB.tmp" || true
    mv "$DB.tmp" "$DB" 2>/dev/null || true
    echo "$MAC $HOST $IP" >> "$DB"


Ensuite

    chmod +x /etc/hotplug.d/hostapd/10-wifi-connect-log
    /etc/init.d/network restart

Warning

Selon version d’OpenWrt, le nom exact de l’ACTION peut être AP-STA-CONNECTED ou similaire, au lieu de connected. A vérifier avec logread | grep hostapd pour ajuster la condition. 


Etaps 4 - Ecrire un script qui va lire les logs OpenWrt
