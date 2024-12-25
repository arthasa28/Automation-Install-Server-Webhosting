# Automation-Install-Server-Webhosting

## Salin Script Ini
```bash
#!/bin/bash

# Fungsi untuk menampilkan animasi loading
loading_animation() {
    local duration=$1
    local end=$((SECONDS + duration))
    while [ $SECONDS -lt $end ]; do
        printf "."
        sleep 0.5
    done
    printf "\n"
}

# Fungsi untuk membersihkan terminal dan menampilkan judul
clear_and_title() {
    clear
    echo -e "\033[1;34m=============================="
    echo -e "         $1         "
    echo -e "==============================\033[0m"
}

# Fungsi untuk menampilkan pesan sukses
success_message() {
    echo -e "\033[1;32m[SUCCESS] $1\033[0m"
}

# Fungsi untuk menampilkan pesan error
error_message() {
    echo -e "\033[1;31m[ERROR] $1\033[0m"
}

# Fungsi untuk menampilkan pesan peringatan
warning_message() {
    echo -e "\033[1;33m[WARNING] $1\033[0m"
}

# Update system
clear_and_title "System Update"
echo "Updating package lists..."
sudo apt update -y && success_message "System updated!" || error_message "Failed to update system."

clear_and_title "Install Server Hosting"

# Install Webmin
echo "Installing Webmin..."
curl -o webmin-setup-repos.sh https://raw.githubusercontent.com/webmin/webmin/master/webmin-setup-repos.sh
sh webmin-setup-repos.sh <<EOF
y
EOF
loading_animation 10
sudo apt-get install webmin --install-recommends -y && success_message "Webmin installed!" || error_message "Failed to install Webmin."

# Install Usermin
echo "Installing Usermin..."
curl -o usermin-setup-repos.sh https://raw.githubusercontent.com/webmin/webmin/master/setup-repos.sh
sh usermin-setup-repos.sh <<EOF
y
EOF
loading_animation 10
sudo apt-get install usermin --install-recommends -y && success_message "Usermin installed!" || error_message "Failed to install Usermin."

# Install Apache2
echo "Installing Apache2 and dependencies..."
sudo apt install libapr1 libaprutil1 libaprutil1-dbd-sqlite3 libaprutil1-ldap libjansson4 liblua5.2-0 apache2-bin apache2-data apache2-utils apache2 ssl-cert -y && success_message "Apache2 installed!" || error_message "Failed to install Apache2."

# Install PHP
echo "Installing PHP..."
sudo apt install php libapache2-mod-php -y && success_message "PHP installed!" || error_message "Failed to install PHP."

# Create client group
echo "Creating 'client' group..."
sudo groupadd client && success_message "'client' group created!" || error_message "Failed to create 'client' group."

clear_and_title "Install Cloudflare"

# Add Cloudflare GPG key and repository
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list

# Update system
sudo apt update -y && success_message "System updated!" || error_message "Failed to update system."

# Install Cloudflare
sudo apt-get install cloudflared -y && success_message "Cloudflared installed!" || error_message "Failed to install Cloudflared."

clear_and_title "Membuat Tunnel Cloudflare"

# Authenticate Cloudflared
cloudflared tunnel login

# Create Tunnel
warning_message "Ensure the tunnel name is unique and meaningful for identification."
echo -e "\033[1;34mEnter Tunnel Name:\033[0m "
read TUNNEL_NAME
while [ -z "$TUNNEL_NAME" ]; do
    error_message "Tunnel name cannot be empty."
    echo -e "\033[1;34mEnter Tunnel Name:\033[0m "
    read TUNNEL_NAME
done
cloudflared tunnel create "$TUNNEL_NAME"
cloudflared tunnel list

# Configure DNS
warning_message "Ensure the subdomain names are correctly configured in your Cloudflare account."
echo -e "\033[1;34mEnter Subdomain for Webmin:\033[0m "
read SUBDOMAIN_WEBMIN
while [ -z "$SUBDOMAIN_WEBMIN" ]; do
    error_message "Subdomain for Webmin cannot be empty."
    echo -e "\033[1;34mEnter Subdomain for Webmin:\033[0m "
    read SUBDOMAIN_WEBMIN
done

echo -e "\033[1;34mEnter Subdomain for Usermin:\033[0m "
read SUBDOMAIN_USERMIN
while [ -z "$SUBDOMAIN_USERMIN" ]; do
    error_message "Subdomain for Usermin cannot be empty."
    echo -e "\033[1;34mEnter Subdomain for Usermin:\033[0m "
    read SUBDOMAIN_USERMIN
done

echo -e "\033[1;34mEnter Subdomain for Wildcard:\033[0m "
read SUBDOMAIN_WILDCARD
while [ -z "$SUBDOMAIN_WILDCARD" ]; do
    error_message "Subdomain for Wildcard cannot be empty."
    echo -e "\033[1;34mEnter Subdomain for Wildcard:\033[0m "
    read SUBDOMAIN_WILDCARD
done

cloudflared tunnel route dns "$TUNNEL_NAME" "$SUBDOMAIN_WEBMIN"
cloudflared tunnel route dns "$TUNNEL_NAME" "$SUBDOMAIN_USERMIN"
cloudflared tunnel route dns "$TUNNEL_NAME" "$SUBDOMAIN_WILDCARD"

# Configure Cloudflare Tunnel
clear_and_title "Konfigurasi Cloudflare Tunnel"

cd ~/.cloudflared
TUNNEL_ID=$(cloudflared tunnel list | grep "$TUNNEL_NAME" | awk '{print $1}')

warning_message "Ensure the IP address is reachable from Cloudflare."
echo -e "\033[1;34mEnter Server IP Address:\033[0m "
read SERVER_IP
while [ -z "$SERVER_IP" ]; do
    error_message "Server IP cannot be empty."
    echo -e "\033[1;34mEnter Server IP Address:\033[0m "
    read SERVER_IP
done

cat <<EOF >config.yml
tunnel: $TUNNEL_ID
credentials-file: /root/.cloudflared/$TUNNEL_ID.json

ingress:
  - hostname: $SUBDOMAIN_WEBMIN
    service: https://$SERVER_IP:10000
    originRequest:
       noTLSVerify: true
       disableChunkedEncoding: true
       noHappyEyeballs: true

  - hostname: $SUBDOMAIN_USERMIN
    service: https://$SERVER_IP:20000
    originRequest:
       noTLSVerify: true
       disableChunkedEncoding: true
       noHappyEyeballs: true

  - hostname: "$SUBDOMAIN_WILDCARD"
    service: http://$SERVER_IP
  - service: http_status:404
EOF

# Install and start Cloudflare service
cloudflared service install
systemctl start cloudflared

clear_and_title "Installation Complete"
success_message "Webmin can be accessed at: https://$SUBDOMAIN_WEBMIN"
success_message "Usermin can be accessed at: https://$SUBDOMAIN_USERMIN"
success_message "Server setup is complete!"

```
