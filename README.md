"# openwrtnotificator" 

PreRequis
 - installer curl

Etape 1 - Configurer un BOT Telegram

Etape 2 - Recupérer le chatid et le token

Etape 3 - Stocker dans un fichier les adresses MAC, IP, hostname des devices connectés

Principe
OpenWrt déclenche des scripts dans /etc/hotplug.d/hostapd/ à chaque événement d’association/déassociation Wi‑Fi, en passant des variables d’environnement comme ACTION, MAC, IFNAME, etc.​
En ajoutant un script dans ce répertoire, nous allons écrire dans une base les information MAC HOST IP.

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

    #!/bin/sh

    DB="/tmp/mac-hostname.db"

    send_to_telegram() {
        logger -p local0.info -t dhcp-remove-notify "$1"
        chat_id=""
        token=""
        if [[ -n "${token}" ]] && [[ -n "${chat_id}" ]]; then
            #curl -okim 10 --data disable_notification="false" --data parse_mode="MarkdownV2" --data chat_id="${chat_id}" --data-urlencode "text=${1}" "https://api.telegram.org/bot${token}/sendMessage" > /dev/null
            curl -skim 10 --data disable_notification="false" --data parse_mode="MarkdownV2" --data chat_id="" --data-urlencode "text=${1}" "https://api.telegram.org/bot/sendMessage" > /dev/null
        else
            logger -p local0.info -t dhcp-remove-notify "Error: Your telegram chat_id or token is empty"
        fi
    }

    logread -f | while read line; do
    # On ne garde que les lignes hostapd qui contiennent l’un des deux événements
    echo "$line" | grep -q "hostapd: phy" || continue
    echo "$line" | grep -Eq "AP-STA-CONNECTED|AP-STA-DISCONNECTED" || continue

    # Déterminer le type d’événement
    if echo "$line" | grep -q "AP-STA-CONNECTED"; then
        EVENT="CONNECTED"
    else
        EVENT="DISCONNECTED"
    fi

    # Récupérer l’interface (phyX-apY)
    IFACE=$(echo "$line" | awk '{for(i=1;i<=NF;i++) if($i ~ /phy[0-9]-ap[0-9]:/) {gsub(":", "", $i); print $i;}}')

    # Récupérer la MAC (champ juste après AP-STA-CONNECTED / AP-STA-DISCONNECTED)
    MAC=$(echo "$line" | awk '{for(i=1;i<=NF;i++) if($i=="AP-STA-CONNECTED" || $i=="AP-STA-DISCONNECTED") {print $(i+1);}}')

    # Lire HOST et IP depuis la DB : format "MAC HOST IP"
    HOST=$(grep "^$MAC " "$DB" 2>/dev/null | awk '{print $2}' | head -n1)
    IP=$(grep "^$MAC " "$DB" 2>/dev/null | awk '{print $3}' | head -n1)

    LINE_DB=$(grep "^$MAC " "$DB" 2>/dev/null | head -n1)
    echo $LINE_DB
        HOST=$(echo "$LINE_DB" | awk '{print $2}')
        IP=$(echo "$LINE_DB" | awk '{print $3}')

        [ -z "$HOST" ] && HOST="unknown"
        [ -z "$IP" ] && IP="unknown"

    # DEBUG sur la console (stdout)
        echo "DEBUG wifi-log: MAC=$MAC HOST=$HOST IP=$IP IFACE=$IFACE EVENT=$EVENT"

    # Message perso
    if [ "$EVENT" = "CONNECTED" ]; then
        msg="Client WiFi connecte: $MAC ($HOST) sur $IFACE $IP"
    else
        msg="Client WiFi deconnecte: $MAC ($HOST) sur $IFACE $IP"
    fi
    logger -t wifi-client2 $msg
    chat_id=""
    token=""
    #curl -skim 10 --data disable_notification="false" --data parse_mode="MarkdownV2" --data chat_id="$chat_id" --data-urlencode "text=msg" "https://api.telegram.org/bot${token}/sendMessage" > /dev/null

    #send_to_telegram "TEST"

      send_to_telegram "\#$(echo "${hostname}" | sed 's/[^a-zA-Z0-9]//g') ${EVENT}
    \`\`\`
    Time: $(date "+%A %d-%b-%Y %T")
    Hostname: ${HOST}
    IP Address: ${IP}
    MAC Address: ${MAC}
    \`\`\`"

    done


Etape 5 Rendre le script exécutable

    chmod +x /usr/sbin/wifi-log-daemon.sh

Voir le résultat en One SHot 

    /usr/sbin/wifi-log-daemon.sh

Etape 6 Creer un script de service

Créer le fichier /etc/init.d/wifi-log-daemon

    #!/bin/sh /etc/rc.common

    START=99
    USE_PROCD=1

    NAME=wifi-log-daemon
    PROG=/usr/sbin/wifi-log-daemon.sh

    start_service() {
        procd_open_instance
        procd_set_param command "$PROG"
        procd_set_param stdout 1      # optionnel : envoie stdout vers logd
        procd_set_param stderr 1
        procd_set_param respawn       # redémarre si le script plante
        procd_close_instance
    }

Puis

    chmod +x /etc/init.d/wifi-log-daemon

2. Activer et tester le service

    /etc/init.d/wifi-log-daemon enable    # auto au boot
    /etc/init.d/wifi-log-daemon start     # démarrage immédiat

Tu peux voir son statut et les logs comme pour les autres services :

text
/etc/init.d/wifi-log-daemon status
logread | grep wifi-client

   

