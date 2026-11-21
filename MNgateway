cat << 'EOF' > v2ray-gateway-manager.sh
#!/bin/bash
set -e

if [ "$EUID" -ne 0 ]; then
  echo "Error: Please run as root (sudo bash v2ray-gateway-manager.sh)"
  exit 1
fi

SSH_PORT=$(echo "$SSH_CONNECTION" | awk '{print $4}')
SSH_PORT=${SSH_PORT:-22}
DEFAULT_WAN=$(ip route show default 2>/dev/null | awk '{print $5}' | head -n1)

select_interface() {
  local prompt_text="$1"
  local -n selected_var="$2"

  mapfile -t ifaces < <(ip -o link show | awk -F': ' '{print $2}' | grep -v '^lo$')

  if [ ${#ifaces[@]} -eq 0 ]; then
    echo "Error: No valid network interfaces found."
    exit 1
  fi

  echo ""
  echo "$prompt_text"
  for i in "${!ifaces[@]}"; do
    if [ "${ifaces[$i]}" == "$DEFAULT_WAN" ]; then
      echo "  $((i+1))) ${ifaces[$i]} [Current WAN / SSH]"
    else
      echo "  $((i+1))) ${ifaces[$i]}"
    fi
  done

  while true; do
    read -rp "Select [1-${#ifaces[@]}]: " choice
    if [[ "$choice" =~ ^[0-9]+$ ]] && [ "$choice" -ge 1 ] && [ "$choice" -le "${#ifaces[@]}" ]; then
      selected_var="${ifaces[$((choice-1))]}"
      break
    else
      echo "Invalid selection. Try again."
    fi
  done
}

convert_and_save_config() {
  local input_file="$1"
  local output_file="$2"
  local socks_port="$3"

  python3 - << PYEOF
import json, sys

in_file = "$input_file"
out_file = "$output_file"
socks_port = int("$socks_port")

with open(in_file, 'r', encoding='utf-8') as f:
    raw = f.read().strip()

inbound_def = [{
    "tag": "socks-in",
    "port": socks_port,
    "listen": "127.0.0.1",
    "protocol": "socks",
    "settings": {"auth": "noauth", "udp": True}
}]
freedom_outbound = {"tag": "direct", "protocol": "freedom"}

def convert_singbox_outbound(sb):
    proto = sb.get("type", "vmess")
    out = {
        "tag": sb.get("tag", "proxy"),
        "protocol": proto,
        "settings": {},
        "streamSettings": {}
    }
    addr = sb.get("server", "")
    port = sb.get("server_port", 443)
    uuid = sb.get("uuid", "")
    
    if proto in ["vmess", "vless"]:
        user_obj = {
            "id": uuid,
            "security": sb.get("security", "auto")
        }
        if "flow" in sb:
            user_obj["flow"] = sb["flow"]
        if proto == "vmess":
            user_obj["alterId"] = sb.get("alter_id", 0)
        out["settings"]["vnext"] = [{
            "address": addr,
            "port": port,
            "users": [user_obj]
        }]
    elif proto in ["trojan", "shadowsocks"]:
        out["settings"]["servers"] = [{
            "address": addr,
            "port": port,
            "password": sb.get("password", "")
        }]
    
    transport = sb.get("transport", {})
    t_type = transport.get("type", "")
    if t_type == "ws":
        out["streamSettings"]["network"] = "ws"
        out["streamSettings"]["wsSettings"] = {
            "path": transport.get("path", "/"),
            "headers": transport.get("headers", {})
        }
    elif t_type == "grpc":
        out["streamSettings"]["network"] = "grpc"
        out["streamSettings"]["grpcSettings"] = {
            "serviceName": transport.get("service_name", "")
        }
    elif t_type == "httpupgrade":
        out["streamSettings"]["network"] = "httpupgrade"
        out["streamSettings"]["httpupgradeSettings"] = {
            "path": transport.get("path", "/")
        }
    
    tls = sb.get("tls", {})
    if tls.get("enabled", False) or sb.get("security") == "tls":
        out["streamSettings"]["security"] = "tls"
        out["streamSettings"]["tlsSettings"] = {
            "serverName": tls.get("server_name", addr),
            "allowInsecure": tls.get("insecure", False)
        }
    reality = tls.get("reality", {})
    if reality.get("enabled", False):
        out["streamSettings"]["security"] = "reality"
        out["streamSettings"]["realitySettings"] = {
            "show": False,
            "fingerprint": "chrome",
            "serverName": tls.get("server_name", ""),
            "publicKey": reality.get("public_key", ""),
            "shortId": reality.get("short_id", "")
        }
        
    return out

final_config = {}

try:
    data = json.loads(raw)
    if isinstance(data, list):
        data = {"outbounds": data}

    outbounds_list = data.get("outbounds", [])
    
    is_singbox = False
    for o in outbounds_list:
        if "type" in o and "protocol" not in o:
            is_singbox = True
            break
            
    if is_singbox:
        converted_outbounds = []
        for o in outbounds_list:
            converted_outbounds.append(convert_singbox_outbound(o))
        converted_outbounds.append(freedom_outbound)
        final_config = {
            "log": {"loglevel": "warning"},
            "inbounds": inbound_def,
            "outbounds": converted_outbounds
        }
    else:
        data["log"] = data.get("log", {"loglevel": "warning"})
        inbounds = data.get("inbounds", [])
        has_socks = False
        for ib in inbounds:
            if ib.get("protocol") == "socks" and int(ib.get("port", 0)) == socks_port:
                has_socks = True
                break
        if not has_socks:
            inbounds.extend(inbound_def)
        data["inbounds"] = inbounds
        final_config = data

except Exception as e:
    print(f"Error parsing JSON: {e}", file=sys.stderr)
    sys.exit(1)

with open(out_file, 'w', encoding='utf-8') as f:
    json.dump(final_config, f, indent=2)

print("Configuration successfully standardized.")
PYEOF
}

test_xray_config() {
  echo "======================================"
  echo "      Test Xray Config Validity       "
  echo "======================================"
  XRAY_CONF="/usr/local/etc/xray/config.json"

  if [ ! -f "$XRAY_CONF" ]; then
    echo -e "\e[31mError: Configuration file not found at $XRAY_CONF\e[0m"
    read -rp "Press Enter to return to menu..."
    return
  fi

  echo "Checking syntax with: xray run -test -c $XRAY_CONF"
  echo ""
  if xray run -test -c "$XRAY_CONF"; then
    echo ""
    echo -e "\e[32m[PASS] Configuration file is valid and syntax is OK.\e[0m"
  else
    echo ""
    echo -e "\e[31m[FAIL] Configuration validation failed. Review the output above.\e[0m"
  fi
  echo ""
  read -rp "Press Enter to return to menu..."
}

install_gateway() {
  echo "======================================"
  echo "        Gateway Installation          "
  echo "======================================"

  select_interface "Select WAN Interface (Internet / SSH):" WAN_IF
  select_interface "Select LAN Interface (Connected to Switch / Clients):" LAN_IF

  if [ "$WAN_IF" == "$LAN_IF" ]; then
    echo "Error: WAN and LAN must be different interfaces."
    return 1
  fi

  read -rp "Enter LAN Gateway IP [192.168.10.1]: " LAN_IP
  LAN_IP=${LAN_IP:-192.168.10.1}

  read -rp "Enter Subnet Netmask [255.255.255.0]: " NETMASK
  NETMASK=${NETMASK:-255.255.255.0}

  read -rp "Enter CIDR prefix [24]: " CIDR
  CIDR=${CIDR:-24}

  read -rp "Enter DHCP Range Start [192.168.10.50]: " DHCP_START
  DHCP_START=${DHCP_START:-192.168.10.50}

  read -rp "Enter DHCP Range End [192.168.10.200]: " DHCP_END
  DHCP_END=${DHCP_END:-192.168.10.200}

  read -rp "Enter Xray Socks5 Inbound Port [10808]: " SOCKS_PORT
  SOCKS_PORT=${SOCKS_PORT:-10808}
  REDSOCKS_PORT=12345

  echo ""
  echo "Applied Configuration:"
  echo "  WAN Interface : $WAN_IF"
  echo "  LAN Interface : $LAN_IF"
  echo "  LAN IP/Subnet : $LAN_IP/$CIDR ($NETMASK)"
  echo "  DHCP Range    : $DHCP_START - $DHCP_END"
  echo "  Socks5 Port   : $SOCKS_PORT"
  echo ""

  echo "Securing SSH connection on port $SSH_PORT..."
  iptables -I INPUT -p tcp --dport "$SSH_PORT" -j ACCEPT
  iptables -I OUTPUT -p tcp --sport "$SSH_PORT" -j ACCEPT

  echo "Installing required packages (including python3)..."
  DEBIAN_FRONTEND=noninteractive apt-get update
  DEBIAN_FRONTEND=noninteractive apt-get install -y curl wget python3 redsocks dnsmasq iptables iptables-persistent netfilter-persistent

  echo "Installing Xray-core..."
  bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install

  echo "Configuring LAN interface ($LAN_IF)..."
  ip link set "$LAN_IF" up
  if ! ip addr show dev "$LAN_IF" | grep -q "${LAN_IP}/${CIDR}"; then
    ip addr add "${LAN_IP}/${CIDR}" dev "$LAN_IF"
  fi

  echo "Configuring dnsmasq on $LAN_IF only..."
  cat << DMEOF > /etc/dnsmasq.conf
interface=$LAN_IF
except-interface=$WAN_IF
bind-interfaces
dhcp-range=$DHCP_START,$DHCP_END,$NETMASK,12h
dhcp-option=option:router,$LAN_IP
dhcp-option=option:dns-server,$LAN_IP,1.1.1.1
server=1.1.1.1
server=8.8.8.8
listen-address=127.0.0.1,$LAN_IP
port=53
DMEOF

  echo ""
  echo "Paste your Config (Xray JSON, Sing-Box JSON, or outbounds array)."
  echo "When finished, type END on a new line and press Enter:"

  TMP_RAW="/tmp/raw_config_input.json"
  > "$TMP_RAW"
  while IFS= read -r line; do
    [[ "$line" == "END" ]] && break
    echo "$line" >> "$TMP_RAW"
  done

  XRAY_CONF="/usr/local/etc/xray/config.json"
  mkdir -p /usr/local/etc/xray

  echo "Detecting format and standardizing config..."
  if ! convert_and_save_config "$TMP_RAW" "$XRAY_CONF" "$SOCKS_PORT"; then
    echo "Error parsing or converting config. Please check your JSON syntax."
    rm -f "$TMP_RAW"
    return 1
  fi
  rm -f "$TMP_RAW"

  echo "Testing Xray config validity..."
  if ! xray run -test -c "$XRAY_CONF"; then
    echo "Error: Generated Xray configuration failed validation."
    return 1
  fi

  echo "Configuring redsocks..."
  cat << RSEOF > /etc/redsocks.conf
base {
    log_debug = off;
    log_info = on;
    log = "stderr";
    daemon = on;
    user = redsocks;
    group = redsocks;
    redirector = iptables;
}

redsocks {
    local_ip = 0.0.0.0;
    local_port = $REDSOCKS_PORT;
    ip = 127.0.0.1;
    port = $SOCKS_PORT;
    type = socks5;
}
RSEOF

  echo "Enabling IPv4 forwarding..."
  sysctl -w net.ipv4.ip_forward=1
  sed -i '/net.ipv4.ip_forward/d' /etc/sysctl.conf
  echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf

  echo "Applying iptables routing rules..."
  iptables -t nat -N REDSOCKS 2>/dev/null || iptables -t nat -F REDSOCKS

  iptables -t nat -A REDSOCKS -d 0.0.0.0/8 -j RETURN
  iptables -t nat -A REDSOCKS -d 10.0.0.0/8 -j RETURN
  iptables -t nat -A REDSOCKS -d 127.0.0.0/8 -j RETURN
  iptables -t nat -A REDSOCKS -d 169.254.0.0/16 -j RETURN
  iptables -t nat -A REDSOCKS -d 172.16.0.0/12 -j RETURN
  iptables -t nat -A REDSOCKS -d 192.168.0.0/16 -j RETURN
  iptables -t nat -A REDSOCKS -p tcp -j REDIRECT --to-ports $REDSOCKS_PORT

  iptables -t nat -D PREROUTING -i "$LAN_IF" -p tcp -j REDSOCKS 2>/dev/null || true
  iptables -t nat -A PREROUTING -i "$LAN_IF" -p tcp -j REDSOCKS

  iptables -t nat -D POSTROUTING -o "$WAN_IF" -j MASQUERADE 2>/dev/null || true
  iptables -t nat -A POSTROUTING -o "$WAN_IF" -j MASQUERADE

  iptables -I INPUT -i "$LAN_IF" -p udp --dport 67:68 --sport 67:68 -j ACCEPT
  iptables -I INPUT -i "$LAN_IF" -p udp --dport 53 -j ACCEPT
  iptables -I INPUT -i "$LAN_IF" -p tcp --dport 53 -j ACCEPT

  netfilter-persistent save 2>/dev/null || true

  echo "Starting services..."
  systemctl restart xray
  systemctl enable xray
  systemctl restart redsocks
  systemctl enable redsocks
  systemctl restart dnsmasq
  systemctl enable dnsmasq

  echo ""
  echo "Installation completed successfully."
}

show_status() {
  echo "======================================"
  echo "           System Status              "
  echo "======================================"

  echo -n "Xray Service:     "
  if systemctl is-active --quiet xray; then
    echo -e "\e[32mActive (Running)\e[0m"
  else
    echo -e "\e[31mInactive / Failed\e[0m"
  fi

  echo -n "Redsocks Service: "
  if systemctl is-active --quiet redsocks; then
    echo -e "\e[32mActive (Running)\e[0m"
  else
    echo -e "\e[31mInactive / Failed\e[0m"
  fi

  echo -n "Dnsmasq (DHCP):   "
  if systemctl is-active --quiet dnsmasq; then
    echo -e "\e[32mActive (Running)\e[0m"
  else
    echo -e "\e[31mInactive / Failed\e[0m"
  fi

  echo ""
  echo "--- Xray Proxy Connectivity Test ---"
  SOCKS_PORT=$(python3 -c "import json; data=json.load(open('/usr/local/etc/xray/config.json')); print(next((i['port'] for i in data.get('inbounds',[]) if i.get('protocol')=='socks'), 10808))" 2>/dev/null || echo "10808")

  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" --socks5 "127.0.0.1:$SOCKS_PORT" --connect-timeout 5 https://www.google.com || echo "000")
  if [ "$HTTP_CODE" == "200" ] || [ "$HTTP_CODE" == "301" ] || [ "$HTTP_CODE" == "302" ]; then
    OUT_IP=$(curl -s --socks5 "127.0.0.1:$SOCKS_PORT" --connect-timeout 5 https://api.ipify.org || echo "Unknown")
    echo -e "Proxy Connection: \e[32mCONNECTED\e[0m (Public IP: $OUT_IP)"
  else
    echo -e "Proxy Connection: \e[31mFAILED\e[0m (Unable to reach Google via Socks5 port $SOCKS_PORT)"
  fi

  echo ""
  echo "--- Connected DHCP Clients ---"
  LEASE_FILE="/var/lib/misc/dnsmasq.leases"
  [ ! -f "$LEASE_FILE" ] && LEASE_FILE="/var/lib/dnsmasq/dnsmasq.leases"

  if [ -s "$LEASE_FILE" ]; then
    printf "%-18s %-16s %-20s %-12s\n" "MAC Address" "IP Address" "Hostname" "Lease Expiry"
    printf "%-18s %-16s %-20s %-12s\n" "------------------" "----------------" "--------------------" "------------"
    while read -r ts mac ip name client_id; do
      exp_date=$(date -d "@$ts" "+%H:%M:%S" 2>/dev/null || echo "$ts")
      printf "%-18s %-16s %-20s %-12s\n" "$mac" "$ip" "$name" "$exp_date"
    done < "$LEASE_FILE"
  else
    echo "No active DHCP leases found."
  fi

  echo ""
  read -rp "Press Enter to return to menu..."
}

manage_services() {
  while true; do
    echo ""
    echo "======================================"
    echo "         Service Management           "
    echo "======================================"
    echo "1) Start All Services"
    echo "2) Stop All Services"
    echo "3) Restart All Services"
    echo "4) Update Config (Paste New Xray/Sing-Box JSON)"
    echo "5) View Xray Logs"
    echo "6) View Redsocks Logs"
    echo "7) Back to Main Menu"
    echo "======================================"
    read -rp "Select an option [1-7]: " s_opt
    case "$s_opt" in
      1)
        systemctl start xray redsocks dnsmasq
        echo "Services started."
        ;;
      2)
        systemctl stop xray redsocks dnsmasq
        echo "Services stopped."
        ;;
      3)
        systemctl restart xray redsocks dnsmasq
        echo "Services restarted."
        ;;
      4)
        SOCKS_PORT=$(python3 -c "import json; data=json.load(open('/usr/local/etc/xray/config.json')); print(next((i['port'] for i in data.get('inbounds',[]) if i.get('protocol')=='socks'), 10808))" 2>/dev/null || echo "10808")
        echo "Paste your new config (Xray or Sing-Box JSON), type END on a new line and press Enter:"
        TMP_RAW="/tmp/new_config_input.json"
        > "$TMP_RAW"
        while IFS= read -r line; do
          [[ "$line" == "END" ]] && break
          echo "$line" >> "$TMP_RAW"
        done
        XRAY_CONF="/usr/local/etc/xray/config.json"
        if convert_and_save_config "$TMP_RAW" "$XRAY_CONF" "$SOCKS_PORT"; then
          if xray run -test -c "$XRAY_CONF"; then
            systemctl restart xray
            echo "New configuration applied and Xray restarted successfully."
          else
            echo "Error: Validation failed on new configuration."
          fi
        fi
        rm -f "$TMP_RAW"
        ;;
      5)
        journalctl -u xray -n 30 --no-pager
        ;;
      6)
        journalctl -u redsocks -n 30 --no-pager
        ;;
      7)
        break
        ;;
      *)
        echo "Invalid choice."
        ;;
    esac
  done
}

uninstall_gateway() {
  echo "======================================"
  echo "       Gateway Uninstallation         "
  echo "======================================"
  read -rp "Are you sure you want to completely remove the gateway setup? (y/N): " confirm
  if [[ ! "$confirm" =~ ^[yY]$ ]]; then
    echo "Operation canceled."
    return
  fi

  systemctl stop redsocks 2>/dev/null || true
  systemctl disable redsocks 2>/dev/null || true
  systemctl stop xray 2>/dev/null || true
  systemctl disable xray 2>/dev/null || true
  systemctl stop dnsmasq 2>/dev/null || true
  systemctl disable dnsmasq 2>/dev/null || true

  iptables -P INPUT ACCEPT
  iptables -P FORWARD ACCEPT
  iptables -P OUTPUT ACCEPT
  iptables -t nat -F
  iptables -t nat -X
  iptables -F
  iptables -X

  netfilter-persistent save 2>/dev/null || true

  sed -i '/net.ipv4.ip_forward=1/d' /etc/sysctl.conf
  sysctl -w net.ipv4.ip_forward=0
  sysctl -p /etc/sysctl.conf 2>/dev/null || true

  DEBIAN_FRONTEND=noninteractive apt-get purge -y redsocks dnsmasq iptables-persistent netfilter-persistent 2>/dev/null || true
  DEBIAN_FRONTEND=noninteractive apt-get autoremove -y 2>/dev/null || true

  rm -f /etc/redsocks.conf
  rm -f /etc/dnsmasq.conf

  bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ remove --purge 2>/dev/null || true
  rm -rf /usr/local/etc/xray /var/log/xray

  echo ""
  echo "Uninstall completed."
}

while true; do
  echo ""
  echo "======================================"
  echo "      V2Ray / Redsocks Gateway        "
  echo "======================================"
  echo "1) INSTALL"
  echo "2) STATUS (DHCP clients, Proxy Test, Services)"
  echo "3) SERVICE MANAGEMENT (Start, Stop, Restart, Update Config)"
  echo "4) TEST CONFIG VALIDITY (xray run -test -c)"
  echo "5) UNINSTALL"
  echo "6) EXIT"
  echo "======================================"
  read -rp "Select an option [1-6]: " opt
  case "$opt" in
    1) install_gateway ;;
    2) show_status ;;
    3) manage_services ;;
    4) test_xray_config ;;
    5) uninstall_gateway ;;
    6) exit 0 ;;
    *) echo "Invalid option." ;;
  esac
done
EOF
chmod +x v2ray-gateway-manager.sh
sudo ./v2ray-gateway-manager.sh
