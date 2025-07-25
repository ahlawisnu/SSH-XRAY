# SSH-XRAY

> 🧩 **Multiplexing Protokol (TLS + WebSocket) dalam Satu Port**  
Dengan satu port (misalnya 443), kamu bisa menjalankan beberapa layanan seperti:
- HTTPS (Nginx/Apache)
- VMess over WebSocket (Xray)
- SSH over WebSocket (Websockify)

Ini memungkinkan kamu untuk **menghemat port**, serta **menyembunyikan protokol proxy** di balik HTTPS/SSL yang aman.

---

# 🔧 Topologi Multiplexing dengan Nginx

```
Port 443
│
├── Host Header: domain.com → HTTPS (Nginx)
├── Path: /vmess → Proxy ke Xray (VMess WS)
└── Path: /ws   → Proxy ke Websockify (SSH WS)
```

---

## ✅ Fitur Script Ini:

- Instalasi dan konfigurasi otomatis:
  - Nginx sebagai reverse proxy multipath
  - Xray (VMess WebSocket)
  - Websockify (SSH over WebSocket)
- SSL otomatis via Let's Encrypt
- Satu port (`443`) untuk semua layanan
- Mudah dikembangkan lebih lanjut

---

## 📜 Script Multiplexing di Port 443

Simpan script berikut sebagai `multiplexing-all-in-one.sh`, lalu jalankan:

```bash
chmod +x multiplexing-all-in-one.sh && ./multiplexing-all-in-one.sh
```

### 🔽 Script: `multiplexing-all-in-one.sh`

```bash
#!/bin/bash

# Konfigurasi
DOMAIN="yourdomain.com"       # Ganti dengan domain Anda
IP_SERVER=$(curl -s ifconfig.me)

echo "🔧 Memulai instalasi Multiplexing TLS (VMess WS + SSH WS + Web di port 443)..."

# Update sistem
apt update && apt upgrade -y

# Install dependensi
apt install curl wget unzip nginx certbot python3-certbot-nginx dropbear stunnel4 python3-pip xray -y

# Hapus config default Nginx
rm -rf /etc/nginx/sites-enabled/default
rm -rf /etc/nginx/sites-available/default

# Generate SSL certificate
echo "🔐 Mendapatkan sertifikat SSL..."
certbot --nginx -d $DOMAIN --non-interactive --agree-tos -m admin@$DOMAIN
systemctl stop nginx
sleep 5
systemctl start nginx

# Buat UUID acak untuk VMess
UUID=$(cat /proc/sys/kernel/random/uuid)

# Konfigurasi Xray
cat << EOF > /usr/local/etc/xray/config.json
{
  "inbounds": [
    {
      "port": 10000,
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "$UUID",
            "level": 0,
            "alterId": 0
          }
        ],
        "disableInsecureEncryption": true
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
          "path": "/vmess"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
EOF

# Restart Xray
systemctl restart xray
systemctl enable xray

# Instal websockify untuk SSH over WS
pip3 install websockify
mkdir -p /var/www/ssh-ws

# Buat service SSH over WebSocket
cat << EOF > /etc/systemd/system/ssh-ws.service
[Unit]
Description=WebSocket Tunnel for SSH
After=network.target

[Service]
ExecStart=/usr/local/bin/websockify --ssl-only --cert=/etc/letsencrypt/live/$DOMAIN/fullchain.pem --key=/etc/letsencrypt/live/$DOMAIN/privkey.pem 2096 $IP_SERVER:22 --log-file=/var/log/ssh-ws.log
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

# Aktifkan dan mulai SSH WS
systemctl daemon-reload
systemctl enable ssh-ws
systemctl start ssh-ws

# Konfigurasi Nginx Multipath
cat << EOF > /etc/nginx/sites-available/proxy.conf
map \$http_upgrade \$connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 443 ssl;
    server_name $DOMAIN;

    ssl_certificate /etc/letsencrypt/live/$DOMAIN/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/$DOMAIN/privkey.pem;

    location / {
        root /var/www/html;
        index index.html;
    }

    location /vmess {
        proxy_pass http://127.0.0.1:10000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host \$host;
    }

    location /ws {
        proxy_pass http://127.0.0.1:2096;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host \$host;
    }
}
EOF

ln -s /etc/nginx/sites-available/proxy.conf /etc/nginx/sites-enabled/
nginx -t && systemctl restart nginx

# Tampilkan info koneksi
echo ""
echo "🌐 Informasi Multiplexing:"
echo "Domain     : $DOMAIN"
echo "UUID       : $UUID"
echo "Path VMess : /vmess"
echo "Path SSH   : /ws"

echo ""
echo "🔗 Link VMess untuk client V2Ray:"
echo "vmess://$(echo -n "{\"add\":\"$DOMAIN\",\"aid\":\"0\",\"host\":\"$DOMAIN\",\"id\":\"$UUID\",\"net\":\"ws\",\"path\":\"/vmess\",\"port\":\"443\",\"ps\":\"VMESS-WS-MUX\",\"tls\":\"tls\",\"type\":\"none\",\"v\":\"2\"}" | base64 -w0)"

echo ""
echo "🔗 URL WebSocket untuk SSH:"
echo "wss://$DOMAIN:443/ws"

echo ""
echo "🎉 Instalasi selesai! Semua layanan berjalan di port 443."
```

---

## 🧪 Cara Uji Coba

### 1. Uji Koneksi VMess:
- Salin link VMess dari output script.
- Impor ke aplikasi seperti **V2RayN**, **Shadowrocket**, **NekoBox**, dll.

### 2. Uji Koneksi SSH via WebSocket:
Gunakan tools seperti `wscat`:

```bash
npm install -g wscat
wscat -c wss://yourdomain.com:443/ws
```

Lalu masukkan username/password SSH secara manual.

---

## 🛡️ Tips Keamanan Tambahan

- Batasi akses SSH hanya dari localhost:
  ```bash
  sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
  echo "AllowUsers tunneluser" >> /etc/ssh/sshd_config
  systemctl restart ssh
  ```

- Gunakan Fail2Ban untuk proteksi login SSH.

- Gunakan Cloudflare (proxy mode ON) untuk tambahan CDN/WAF.

---

## 🔄 Auto Renew Certificate

Certbot akan otomatis perbarui sertifikat setiap 90 hari. Untuk uji coba:

```bash
certbot renew --dry-run
```
