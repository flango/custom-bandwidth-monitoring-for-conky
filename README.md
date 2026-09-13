# custom-bandwidth-monitoring-for-conky
## Conky Netcat Network Monitoring Daemon
### WAN/LAN version
Remember to check proper eth names to use
```bash
ip -br link
```
First make the script
```bash
sudo nano /usr/local/bin/bandwidth-nc-daemon.sh
```
```ini
#!/bin/bash
LAN_IFACE="enp3s0"
WAN_IFACE="enp2s0"
BIND_IP="192.168.10.1"

# Record exact start times and initial metrics
TIME_OLD=$SECONDS
LAN_OLD=$(awk -v iface="$LAN_IFACE" '$0 ~ iface {print $2, $10}' /proc/net/dev)
WAN_OLD=$(awk -v iface="$WAN_IFACE" '$0 ~ iface {print $2, $10}' /proc/net/dev)

LAN_RX_OLD=$(echo "$LAN_OLD" | awk '{print $1}')
LAN_TX_OLD=$(echo "$LAN_OLD" | awk '{print $2}')
WAN_RX_OLD=$(echo "$WAN_OLD" | awk '{print $1}')
WAN_TX_OLD=$(echo "$WAN_OLD" | awk '{print $2}')

while true; do
    sleep 1
    
    # Capture new clock tick and metrics
    TIME_NEW=$SECONDS
    LAN_NEW=$(awk -v iface="$LAN_IFACE" '$0 ~ iface {print $2, $10}' /proc/net/dev)
    WAN_NEW=$(awk -v iface="$WAN_IFACE" '$0 ~ iface {print $2, $10}' /proc/net/dev)
    
    # Calculate actual elapsed seconds (safeguarded to minimum 1s to prevent division by zero)
    ELAPSED=$(( TIME_NEW - TIME_OLD ))
    if [ "$ELAPSED" -lt 1 ]; then ELAPSED=1; fi

    LAN_RX_NEW=$(echo "$LAN_NEW" | awk '{print $1}')
    LAN_TX_NEW=$(echo "$LAN_NEW" | awk '{print $2}')
    WAN_RX_NEW=$(echo "$WAN_NEW" | awk '{print $1}')
    WAN_TX_NEW=$(echo "$WAN_NEW" | awk '{print $2}')
    
    # Dynamic Math: Divide byte differences by actual ELAPSED time
    LAN_DOWN=$(awk -v old="$LAN_RX_OLD" -v new="$LAN_RX_NEW" -v time="$ELAPSED" 'BEGIN {printf "%.2f", (new-old)/time/1024/1024}')
    LAN_UP=$(awk -v old="$LAN_TX_OLD" -v new="$LAN_TX_NEW" -v time="$ELAPSED" 'BEGIN {printf "%.2f", (new-old)/time/1024/1024}')
    
#    WAN_DOWN=$(awk -v old="$WAN_RX_OLD" -v new="$WAN_WAN_RX_NEW" -v time="$ELAPSED" 'BEGIN {printf "%.2f", (new-old)/time/1024/1024}')
    WAN_DOWN=$(awk -v old="$WAN_RX_OLD" -v new="$WAN_RX_NEW" -v time="$ELAPSED" 'BEGIN {printf "%.2f", (new-old)/time/1024/1024}')
    WAN_UP=$(awk -v old="$WAN_TX_OLD" -v new="$WAN_TX_NEW" -v time="$ELAPSED" 'BEGIN {printf "%.2f", (new-old)/time/1024/1024}')
    
    # Roll states forward for the next tick
    TIME_OLD=$TIME_NEW
    LAN_RX_OLD=$LAN_RX_NEW; LAN_TX_OLD=$LAN_TX_NEW
    WAN_RX_OLD=$WAN_RX_NEW; WAN_TX_OLD=$WAN_TX_NEW

    # Broadcast metrics cleanly
    echo "LAN Down: ${LAN_DOWN} MB/s | Up: ${LAN_UP} MB/s" | nc -l -s "$BIND_IP" -p 7070 -q 1 &
    echo "WAN Down: ${WAN_DOWN} MB/s | Up: ${WAN_UP} MB/s" | nc -l -s "$BIND_IP" -p 7071 -q 1 &
    
    wait
done
```

```bash
sudo chmod +x /usr/local/bin/bandwidth-nc-daemon.sh
```
Next make the service
```bash
sudo nano /etc/systemd/system/bandwidth-nc.service
```
```ini
[Unit]
Description=Conky Netcat Network Monitoring Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/bandwidth-nc-daemon.sh
Restart=always
RestartSec=3
User=root

[Install]
WantedBy=multi-user.target

```
Reload systemd, enable the service for boot, and start it right away
```bash
sudo systemctl daemon-reload
sudo systemctl enable bandwidth-nc.service
sudo systemctl start bandwidth-nc.service
```

### LAN
Remember to check proper eth names to use
```bash
ip -br link
```
First make the script
```bash
sudo nano /usr/local/bin/bandwidth-nc-daemon.sh
```
```ini
#!/bin/bash
IFACE="enp31s0"

# Record exact start time and initial metrics
TIME_OLD=$SECONDS
STATS_OLD=$(awk -v iface="$IFACE" '$0 ~ iface {print $2, $10}' /proc/net/dev)
RX_OLD=$(echo "$STATS_OLD" | awk '{print $1}')
TX_OLD=$(echo "$STATS_OLD" | awk '{print $2}')

while true; do
    sleep 1
    
    # Capture new clock tick and metrics
    TIME_NEW=$SECONDS
    STATS_NEW=$(awk -v iface="$IFACE" '$0 ~ iface {print $2, $10}' /proc/net/dev)
    RX_NEW=$(echo "$STATS_NEW" | awk '{print $1}')
    TX_NEW=$(echo "$STATS_NEW" | awk '{print $2}')
    
    # Calculate actual elapsed seconds (safeguard to minimum 1s)
    ELAPSED=$(( TIME_NEW - TIME_OLD ))
    if [ "$ELAPSED" -lt 1 ]; then ELAPSED=1; fi
    
    # Calculate accurate speed per second (MB/s) by dividing by actual elapsed time
    DOWN_SPEED=$(awk -v old="$RX_OLD" -v new="$RX_NEW" -v time="$ELAPSED" 'BEGIN {printf "%.2f", (new-old)/time/1024/1024}')
    UP_SPEED=$(awk -v old="$TX_OLD" -v new="$TX_NEW" -v time="$ELAPSED" 'BEGIN {printf "%.2f", (new-old)/time/1024/1024}')
    
    # Update old variables for next loop
    TIME_OLD=$TIME_NEW
    RX_OLD=$RX_NEW
    TX_OLD=$TX_NEW

    # Grab the current LAN IP
    CLIENT_IP=$(ip -4 addr show dev "$IFACE" | awk '/inet / {print $2}' | cut -d/ -f1)

    # Listen on port 7070 and send the output to any client that connects
    if [ -n "$CLIENT_IP" ]; then
        echo "LAN Down: ${DOWN_SPEED} MB/s | Up: ${UP_SPEED} MB/s" | nc -l -s "$CLIENT_IP" -p 7070 -q 1
    fi
done

```
```bash
sudo chmod +x /usr/local/bin/bandwidth-nc-daemon.sh
```
Next make the service
```bash
sudo nano /etc/systemd/system/bandwidth-nc.service
```
```ini
[Unit]
Description=Conky Netcat Network Monitoring Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/bandwidth-nc-daemon.sh
Restart=always
RestartSec=3
User=root

[Install]
WantedBy=multi-user.target

```
Reload systemd, enable the service for boot, and start it right away
```bash
sudo systemctl daemon-reload
sudo systemctl enable bandwidth-nc.service
sudo systemctl start bandwidth-nc.service
```
## Connect conky to heimdall, router in this example
Add to conky config
```ini
${color grey}Remote Network Monitoring
${color white}$hr
${color lightgrey}Heimdall:${color2} ${alignr} ${texecpi 3 nc -w 1 192.168.10.1 7071 | awk 'NF{print;c=1} END{if(!c) print "${color7}Offline"}'}
${color lightgrey}Heimdall:${color2} ${alignr} ${texecpi 3 nc -w 1 192.168.10.1 7070 | awk 'NF{print;c=1} END{if(!c) print "${color7}Offline"}'}
${color white}$hr
```
Printout
```ini
WAN Down: 0.19 MB/s | Up: 0.18 MB/s
LAN Down: 0.20 MB/s | Up: 0.21 MB/s
```
7071 is to read wan from router
7070 is used for all lan devices and lan from router too when added to a linux host

---

## 📄 License

This project is open-source and licensed under the **MIT License**. Feel free to use, modify, and distribute it as you see fit. See the accompanying `LICENSE` file for full legal details.

