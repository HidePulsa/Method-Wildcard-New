# Cloudflare ACM Certificate via SSL for SaaS (Custom Hostnames)
## Metode Terbaru 2026 — Gratis 100 Hostname

> ⚠️ **PERINGATAN KERAS!**
> 
> METHOD INI GRATIS. JANGAN DIJUAL BELIKAN, JANGAN DIPERJUALBELIKAN.
> 
> Siapa pun yang ketauan jual method ini, semoga rezekinya seret, akunnya kena banned massal, dan hidupnya ga berkah. Ini ilmu gratis buat sharing, bukan buat cari untung di atas kebodohan orang lain.
> 
> **KALO LU JUAL METHOD INI, LU ANJING.**

---

## Daftar BIN Credit Card

Bisa dipake buat verifikasi payment method Cloudflare (auth $1, direfund):

```
4463340000856
42588164012
55988805
```

Generate CC lengkap di [chckr.cc](https://chckr.cc) atau [namso-gen.com](https://namso-gen.com).

---

## Prasyarat
- Akun Cloudflare (free)
- Payment method terverifikasi (CC/PayPal/Google Pay — tidak dipotong)
- Domain sendiri di Cloudflare
- VPS dengan nginx

---

## STEP 1: Tambah Payment Method

1. Buka `https://dash.cloudflare.com/<ACCOUNT_ID>/billing/payment`
2. Klik **Add Payment Method**
3. Pilih Credit Card / PayPal 
4. Isi detail kartu ( Gunakan Credit Card ini 5598880520345093|07|2029|870 atau bisa generate menggunakan bin ini 55988805 di chckr.cc)
5. Cloudflare auth ~$1 (direfund otomatis)
6. Status: ✅ Verified

---

## STEP 2: Setup Fallback Origin + DNS (PER DOMAIN!)

> ⚠️ **Fallback Origin wajib di-set di SETIAP domain (zone), bukan per akun.**
> Kalo lu punya 3 domain, harus setup 3x.

### 2a. Buat DNS Record (2 record)
**DNS → Records → Add Record:**

**Record 1 — Fallback Origin:**
```
Type: A
Name: fallback
IPv4: <IP VPS ANDA>
Proxy: ✅ ON (oranye)
```

**Record 2 — Wildcard Tunnel (sekali aja):**
```
Type: A
Name: *
IPv4: <IP VPS ANDA>
Proxy: OFF (abu-abu)
```
> ⚡ Record `*` bikin semua subdomain auto-resolve ke VPS. Ga perlu tambah DNS lagi per hostname.

### 2b. Set Fallback Origin di Custom Hostnames
**SSL/TLS → Custom Hostnames**

Isi form:
```
Fallback Origin: fallback.domain-anda.my.id
```
Klik **Add Fallback Origin**

✅ Error "zone does not have a fallback origin" hilang

---

## STEP 3: Setup Nginx di VPS

```bash
ssh root@<IP_VPS>

# Buat direktori ACME challenge
mkdir -p /var/www/acme/.well-known/acme-challenge

# Bikin nginx config
cat > /etc/nginx/sites-enabled/acme-challenge << 'EOF'
server {
    listen 80;
    server_name _;

    location /.well-known/acme-challenge/ {
        root /var/www/acme;
        try_files $uri =404;
    }

    location / {
        return 200 "OK";
        add_header Content-Type text/plain;
    }
}
EOF

# Test & reload
nginx -t && systemctl reload nginx
```

---

## STEP 4: Bikin Custom Hostname via API

### 4a. Dapatkan API Token
1. **Profile (ikon user)** → **API Tokens**
2. **Create Token** → pilih template **"Edit zone DNS"**
3. Permission: `Zone:SSL and Certificates:Edit` + `Zone:DNS:Edit`
4. Zone Resources: Include → Specific zone → `domain-anda.my.id`
5. Copy token

### 4b. Dapatkan Zone ID
Buka dashboard domain → scroll bawah kanan → **Zone ID**

### 4c. Bikin Custom Hostname via API

```bash
ZONE_ID="zone_id_anda"
API_TOKEN="token_anda"
TARGET="support.zoom.us"

curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/custom_hostnames" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"hostname\": \"$TARGET.domain-anda.my.id\",
    \"ssl\": {
      \"method\": \"http\",
      \"type\": \"dv\"
    }
  }"
```

Response sukses:
```json
{
  "success": true,
  "result": {
    "id": "xxx",
    "hostname": "support.zoom.us.domain-anda.my.id",
    "ssl": {
      "status": "pending_validation",
      "validation_records": [
        {
          "txt_name": "_acme-challenge.support.zoom.us",
          "txt_value": "xxx"
        }
      ]
    }
  }
}
```

---

## STEP 5: Validasi Certificate

### HTTP Method (otomatis — nginx)
Cloudflare otomatis kirim request ke:
```
http://support.zoom.us.domain-anda.my.id/.well-known/acme-challenge/<token>
```
Nginx serve file → validated → certificate **ACTIVE** dalam 2-5 menit.

### Kalo masih pending, cek log nginx:
```bash
tail -f /var/log/nginx/access.log | grep acme-challenge
```

---

## STEP 6: Pakai Certificate

Setelah status **Active**:

### 6a. Di Cloudflare (reverse proxy)
```
DNS → Add Record:
Type: CNAME
Name: target-domain-anda
Target: support.zoom.us.domain-anda.my.id
Proxy: ✅ ON
```
SSL/TLS mode: **Full (strict)**

### 6b. Atau export cert manual
```bash
curl -X GET "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/custom_hostnames/$HOSTNAME_ID" \
  -H "Authorization: Bearer $API_TOKEN"
```

---

## API Endpoints yang Dipake

### Flow:

```
1. GET  /zones/:zone/dns_records          → cek record "fallback"
      POST /zones/:zone/dns_records       → bikin kalo belum ada
2. PUT  /zones/:zone/custom_hostnames/fallback_origin → set fallback
3. POST /zones/:zone/custom_hostnames     → bikin custom hostname (HTTP)
4. GET  /zones/:zone/custom_hostnames/:id → cek status certificate
5. GET  /zones/:zone/custom_hostnames     → list semua hostname
```

### Request & Response:

**1. Cek/Tambah DNS Record:**
```bash
# GET — list DNS
curl "https://api.cloudflare.com/client/v4/zones/$ZONE/dns_records" \
  -H "Authorization: Bearer ***  -H "Content-Type: application/json"

# POST — bikin record fallback
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE/dns_records" \
  -H "Authorization: Bearer ***  -H "Content-Type: application/json" \
  -d '{"type":"A","name":"fallback","content":"<VPS_IP>","proxied":true,"ttl":1}'
```

```json
// Response
{ "success": true, "result": { "id": "xxx", "name": "fallback.domain.my.id", "type": "A", "content": "18.143.144.152" } }
```

**2. Set Fallback Origin:**
```bash
curl -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE/custom_hostnames/fallback_origin" \
  -H "Authorization: Bearer ***  -H "Content-Type: application/json" \
  -d '{"origin":"fallback.domain.my.id"}'
```

```json
// Response
{ "success": true, "result": { "origin": "fallback.domain.my.id", "status": "active" } }
```

**3. Bikin Custom Hostname (HTTP validation):**
```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE/custom_hostnames" \
  -H "Authorization: Bearer ***  -H "Content-Type: application/json" \
  -d '{"hostname":"support.zoom.us.domain.my.id","ssl":{"method":"http","type":"dv"}}'
```

```json
// Response
{
  "success": true,
  "result": {
    "id": "3d8e6b03-xxx",
    "hostname": "support.zoom.us.domain.my.id",
    "status": "pending",
    "ssl": { "status": "pending_validation", "method": "http", "type": "dv" }
  }
}
```

**4. Cek Status Certificate:**
```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE/custom_hostnames/$HOSTNAME_ID" \
  -H "Authorization: Bearer ***
```

```json
// Response (active)
{
  "success": true,
  "result": {
    "id": "3d8e6b03-xxx",
    "hostname": "support.zoom.us.domain.my.id",
    "status": "active",
    "ssl": { "status": "active", "method": "http", "type": "dv" }
  }
}
```

**5. List Semua Custom Hostname:**
```bash
curl "https://api.cloudflare.com/client/v4/zones/$ZONE/custom_hostnames" \
  -H "Authorization: Bearer ***
```

```json
// Response
{
  "success": true,
  "result": [
    {
      "id": "xxx",
      "hostname": "support.zoom.us.domain.my.id",
      "status": "active",
      "ssl": { "status": "active" }
    }
  ]
}
```

**6. Delete Custom Hostname:**
```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/zones/$ZONE/custom_hostnames/$HOSTNAME_ID" \
  -H "Authorization: Bearer ***
```

**7. Delete DNS Record:**
```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/zones/$ZONE/dns_records/$RECORD_ID" \
  -H "Authorization: Bearer ***  -H "Content-Type: application/json"
```

---

## Otomatisasi (Bot / Web / API / Script)

Endpoint di atas bisa dipake buat bikin automation lewat:
- **Telegram Bot** (grammy/telegraf)
- **Web Dashboard** (React/Next.js)
- **REST API** (Express/FastAPI)
- **CLI Script** (bash/node)

Tinggal implementasiin flow: **cek DNS → set fallback → bikin hostname → cek status**.

---

## Troubleshooting

| Error | Solusi |
|-------|--------|
| "zone does not have a fallback origin set" | STEP 2 — set Fallback Origin + DNS |
| "Pending (Error)" | Cek TXT records di DNS, atau ganti ke HTTP method |
| "Certificate status: Pending Validation" | Tunggu 2-5 menit, refresh halaman |
| Nginx ga serve challenge | `chmod 755 /var/www/acme`, cek owner |
| Firewall block port 80 | `ufw allow 80/tcp` |
| Custom hostname limit | Free plan: 100 hostname. Hapus yg ga dipake |
| "Authentication error" | API token ga punya `SSL and Certificates:Edit` — tambah permission |

### Rekomendasi Setting Custom Hostname:
- **Minimum TLS:** 1.2 (1.0 vulnerable, 1.3 kurang kompatibel)
- **SSL Certificate Authority:** Google Trust Services (default)

---

---

## ☕ Support

Method ini gratis. Kalo terbantu, boleh traktir kopi ☕

![QRIS](./images/qris-donate.jpg)

---

## Limit & Biaya

| Item | Free | Berbayar |
|------|------|----------|
| Custom hostname | 100 pertama | Hostname ke-101: $0.10/bulan |
| Certificate | ✅ Gratis per hostname | - |
| Payment method | Verifikasi doang | Tidak dipotong |
| Renew | ✅ Otomatis | - |