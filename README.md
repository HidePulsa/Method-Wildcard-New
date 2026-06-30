# Cloudflare ACM Certificate via SSL for SaaS (Custom Hostnames)
## Metode Terbaru 2026 — Gratis 100 Hostname

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

---

## Setup Banyak Domain

```
⚠️ 1 Domain = 1 Fallback Origin. Tapi 1 VPS bisa handle semuanya.

Akun A → domain1.my.id → fallback.domain1.my.id ─┐
Akun B → domain2.my.id → fallback.domain2.my.id ─┼─→ 1 VPS (nginx catch-all)
Akun C → domain3.my.id → fallback.domain3.my.id ─┘
```

**Per domain, ulangi:**
1. DNS: A record `fallback` → IP VPS → proxy ON
2. SSL/TLS → Custom Hostnames → Add Fallback Origin
3. Bikin custom hostname via API

**VPS: 1x setup aja** (server_name `_` catch-all)

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

## Limit & Biaya

| Item | Free | Berbayar |
|------|------|----------|
| Custom hostname | 100 pertama | Hostname ke-101: $0.10/bulan |
| Certificate | ✅ Gratis per hostname | - |
| Payment method | Verifikasi doang | Tidak dipotong |
| Renew | ✅ Otomatis | - |