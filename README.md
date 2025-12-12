"# openwrtnotificator" 

Etape 1 - Configurer un BOT Telegram

Etape 2 - Recupérer le chatid et le token

Etape 3 - Stocker dans un fichier les adresses MAC, IP, hostname des devices connectés

#!/bin/sh

DB="/tmp/mac-hostname.db"

logread -f | while read line; do
    echo "$line" | grep -q "hostapd: phy" || continue
    echo "$line" | grep -Eq "AP-STA-CONNECTED|AP-STA-DISCONNECTED" || continue

    if echo "$line" | grep -q "AP-STA-CONNECTED"; then
        EVENT="CONNECTED"
    else
        EVENT="DISCONNECTED"
    fi

    IFACE=$(echo "$line" | awk '{for(i=1;i<=NF;i++) if($i ~ /phy[0-9]-ap[0-9]:/) {gsub(":", "", $i); print $i;}}')
    MAC=$(echo "$line" | awk '{for(i=1;i<=NF;i++) if($i=="AP-STA-CONNECTED" || $i=="AP-STA-DISCONNECTED") {print $(i+1);}}')

    # Lire HOST et IP depuis la DB : format "MAC HOST IP"
    HOST=$(grep "^$MAC " "$DB" 2>/dev/null | awk '{print $2}' | head -n1)
    IP=$(grep "^$MAC " "$DB" 2>/dev/null | awk '{print $3}' | head -n1)

    [ -z "$HOST" ] && HOST="unknown"
    [ -z "$IP" ] && IP="unknown"

    if [ "$EVENT" = "CONNECTED" ]; then
        logger -t wifi-client "Client WiFi connecté: $MAC ($HOST) IP=$IP sur $IFACE"
    else
        logger -t wifi-client "Client WiFi déconnecté: $MAC ($HOST) IP=$IP sur $IFACE"
    fi
done


Etaps 4 - Ecrire un script qui va lire les logs OpenWrt
