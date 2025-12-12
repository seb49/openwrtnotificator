"# openwrtnotificator" 

PreRequis
 - installer curl

Etape 1 - Configurer un BOT Telegram

Etape 2 - Recupérer le chatid et le token

Etape 3 - Stocker dans un fichier les adresses MAC, IP, hostname des devices connectés

Crée le fichier /etc/hotplug.d/hostapd/10-wifi-connect-log avec ce contenu :

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

    [ -z "$MAC" ] && exit 0

    DB="/tmp/mac-hostname.db"

    [ -z "$HOST" ] && HOST="unknown"

    # On remplace l’ancienne ligne pour cette MAC
    grep -v "^$MAC " "$DB" 2>/dev/null > "$DB.tmp" || true
    mv "$DB.tmp" "$DB" 2>/dev/null || true
    echo "$MAC $HOST" >> "$DB"



Etaps 4 - Ecrire un script qui va lire les logs OpenWrt
