# 🏆 Football Match Results

ระบบแสดงผลการแข่งขันฟุตบอล พร้อม HT Score, Corner Stats, Score Detail
แบบ Centralized Architecture — ดึงข้อมูลจาก API ต้นทาง → Cache บน yamcha.info → ส่งต่อให้ WordPress Plugin

---

## 🏗️ Architecture

```
hethongapi.com (Source API)
    ↓ Cron every 2 min
yamcha.info/football-match-results/  ← CENTER
    ├── api-proxy.php   (Cron fetcher)
    ├── api.php         (API endpoint)
    ├── index.php       (Admin monitor)
    ├── cache/          (Cached HTML)
    └── .env            (Secret config)
    ↓ HTTPS + X-Yamcha-Key header
WordPress Site (Client)
    ├── REST Proxy      (PHP server-side)
    └── Widget / Shortcode → User sees results
```

---

## 📁 Project Structure

```
football-match-results/           ← Deploy to yamcha.info
├── .env.example                  ← Copy → .env แล้วกรอกค่า
├── .env                          ← ห้าม commit (gitignore)
├── .htaccess                     ← ปิด cache/ และ .env จาก browser
├── api-proxy.php                 ← รันโดย Cron ทุก 2 นาที
├── api.php                       ← API endpoint สำหรับ Plugin
├── index.php                     ← Admin Monitor (password protected)
├── cache/
│   ├── results.html              ← Cached data (auto-created)
│   ├── meta.json                 ← Cache metadata
│   └── proxy.log                 ← Cron log
└── includes/
    ├── Config.php                ← อ่าน .env
    └── Cache.php                 ← จัดการ cache file

football-match-results-client/    ← WordPress Plugin
├── football-match-results-client.php
├── includes/
│   ├── Settings.php              ← Admin settings + AES-256 key storage
│   ├── ApiClient.php             ← Server-side fetch (key ไม่ผ่าน browser)
│   └── Widget.php                ← Elementor widget + Shortcode
└── assets/
    ├── css/results.css           ← Dark theme + Win/Draw/Loss highlight
    └── js/results.js             ← Auto-refresh + Vietnamese→English translation
```

---

## ⚙️ Installation — yamcha.info Center

### 1. Upload Files
```
อัปโหลด football-match-results/ ไปที่:
/public_html/football-match-results/
```

### 2. Create .env
```bash
cp .env.example .env
```

แก้ไข `.env`:
```env
SOURCE_URL=https://hethongapi.com/cron/bongda/ketquabongda/Getketquabongdaaaa.txt
API_KEYS=key_xxxx,key_yyyy,key_zzzz
CACHE_TTL=120
ADMIN_SECRET=your_admin_password
CRON_SECRET=your_cron_secret
```

### 3. Generate API Keys
```bash
# PHP
php -r "echo 'key_' . bin2hex(random_bytes(16));"

# Python
python3 -c "import secrets; print('key_' + secrets.token_hex(16))"
```

### 4. Setup Cron Job (cPanel)
```
Minute: */2  |  Hour: *  |  Day: *  |  Month: *  |  Weekday: *

Command:
/usr/local/bin/php /home/USERNAME/public_html/football-match-results/api-proxy.php
```

### 5. Test
```bash
# SSH test
php api-proxy.php
# Expected: [16:00:00] ✓ OK — 52ms — 103,167 bytes

# Browser monitor
https://yamcha.info/football-match-results/?secret=YOUR_ADMIN_SECRET
```

---

## ⚙️ Installation — WordPress Plugin

### 1. Install Plugin
- WordPress Admin → Plugins → Add New → Upload Plugin
- เลือก `football-match-results-client.zip` → Install → Activate

### 2. Configure
- Settings → **Football Match Results**
- Center URL: `https://yamcha.info/football-match-results`
- API Key: `key_xxxx` (ขอจากผู้ดูแล yamcha.info)
- กด **Test Connection** → ต้องได้ ✅ Connection OK

### 3. Add to Page

**Elementor:**
- ค้นหา widget: `Football Match Results`
- ลากวางในหน้า
- ปรับ Settings ใน panel

**Shortcode:**
```
[football_match_results height="800px" auto_refresh="yes"]
```

---

## 🎨 Score Display

ระบบแสดงผล Win/Draw/Loss ด้วยสีที่ต่างกัน:

| ผลลัพธ์ | สี | CSS Class |
|---------|-----|-----------|
| 🟢 Home Win | พื้นเขียว — คะแนนเขียว | `as-score-vs-xoilacz__home-win` |
| 🔵 Away Win | พื้นน้ำเงิน — คะแนนฟ้า | `as-score-vs-xoilacz__away-win` |
| ⚪ Draw | พื้นเทา — คะแนนขาว | `as-score-vs-xoilacz__draw` |

**ข้อมูลที่แสดงต่อแมตช์:**
```
[Match Row]
03:00 19/02   Wolverhampton  2 : 2  Arsenal    HT: 0:1   Corner: 0-2 | 1-3
Matchday 31   HT 0-1 | 🏁 1-3

[Score Detail Row - ถ้ามี]
FT [0-0], AET [1-0], 1st Leg Ratchaburi 3:0 Persib → Advances
```

---

## 🔑 Key Management

| Key | ใช้สำหรับ | จัดการที่ |
|-----|----------|----------|
| `API_KEYS` | WordPress Plugin ทุก site | `.env` บน yamcha.info |
| `ADMIN_SECRET` | เข้า Monitor `/index.php` | `.env` บน yamcha.info |
| `CRON_SECRET` | เรียก cron ผ่าน HTTP | `.env` บน yamcha.info |

**เพิ่ม Key ใหม่:**
```env
API_KEYS=key_site1,key_site2,key_site3_new
```

**ยกเลิก Key** — ลบออกจาก list ทันที ไม่ต้อง restart

---

## 🔐 Security

- Source URL ซ่อนใน `.env` บน yamcha.info เท่านั้น
- `.htaccess` ปิดการเข้าถึง `.env` และ `cache/` จาก browser
- API Key ส่งผ่าน HTTP Header (`X-Yamcha-Key`) ไม่ผ่าน URL
- WordPress Plugin เก็บ Key แบบ AES-256-CBC encrypted ใน database
- Key ไม่เคยถึง browser — ผ่าน PHP server-side เท่านั้น
- Rate limiting: 60 requests/min per IP
- `hash_equals()` ป้องกัน timing attack

---

## 🌐 API Reference

### `GET /football-match-results/api.php`

**Headers:**
```
X-Yamcha-Key: key_xxxx
```

**Response (Success):**
```json
{
  "success": true,
  "html": "<div class=\"as-content-page\">...</div>",
  "updated_at": 1708329600,
  "updated_iso": "2024-02-19T16:00:00+07:00",
  "cache_age": 45
}
```

**Response (Error):**
```json
{
  "success": false,
  "error": "Invalid or missing API key",
  "code": "INVALID_KEY"
}
```

**Error Codes:**
| Code | HTTP | ความหมาย |
|------|------|----------|
| `INVALID_KEY` | 401 | Key ผิดหรือไม่ได้ส่งมา |
| `RATE_LIMIT` | 429 | เกิน 60 req/min |
| `CACHE_EMPTY` | 503 | Cache ยังไม่มีข้อมูล |

---

## 🖥️ Admin Monitor

เข้าถึง:
```
https://yamcha.info/football-match-results/?secret=ADMIN_SECRET
```

Monitor แสดง:
- Cache Status (Fresh / Stale / Empty)
- Last Updated + ขนาดข้อมูล
- Active Keys (masked)
- Cron Log ล่าสุด 20 บรรทัด
- Live Preview เหมือนกับ Plugin ทุกประการ พร้อม Win/Draw/Loss สี
- Auto-refresh ทุก 30 วินาที

**Actions:**
- 🔄 Fetch Now — ดึงข้อมูลใหม่ทันที
- 🧪 Test API Response — ทดสอบ API endpoint
- ⚡ Run Cron Manually — รัน cron ผ่าน browser

---

## 🌏 Translation (Vietnamese → English)

Plugin แปลอัตโนมัติ client-side รวมถึง Score Detail:

| Vietnamese | English |
|-----------|---------|
| Chủ nhà | Home |
| Khách | Away |
| Vòng 31 | Matchday 31 |
| Bán kết | Semi-final |
| Chung kết | Final |
| Lượt đi | 1st Leg |
| Lượt về | 2nd Leg |
| 90 Phút [0-0] | FT [0-0] |
| 120 Phút [1-0] | AET [1-0] |
| Tiến vào vòng trong | → Advances |
| Tây Ban Nha | Spain |
| Nữ | Women's |

---

## 🕐 Timezone

ระบบใช้เวลาโซนไทย `Asia/Bangkok` (UTC+7) ทั้งหมด:
- PHP: `date_default_timezone_set('Asia/Bangkok')`
- JavaScript: `timeZone: 'Asia/Bangkok'`

---

## 🐛 Troubleshooting

**Cache ว่าง / Cron ไม่ทำงาน**
```bash
# รัน manual บน SSH
php /home/USERNAME/public_html/football-match-results/api-proxy.php

# ดู log
tail -f /public_html/football-match-results/cache/proxy.log
```

**Plugin แสดง "Server unavailable"**
1. เช็ค Center URL ถูกต้องไหม (ไม่ต้องมี `/api.php`)
2. กด Test Connection ใน Settings
3. เช็ค Key ใน `.env` ตรงกับที่ใส่ใน Plugin

**Score Detail ไม่แสดง**
- ข้อมูลนี้มาเฉพาะบางแมตช์ (2nd leg, AET, ผ่านรอบ)
- เช็คว่า CSS `.as-score-detail` ไม่ถูก override

---

## 📋 Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `SOURCE_URL` | ✅ | URL ต้นทางข้อมูลผลบอล |
| `API_KEYS` | ✅ | comma-separated keys สำหรับ Plugin |
| `CACHE_TTL` | ❌ | Cache lifetime วินาที (default: 120) |
| `ADMIN_SECRET` | ✅ | Password เข้า Monitor |
| `CRON_SECRET` | ✅ | Secret สำหรับเรียก cron ผ่าน HTTP |

---

## 🔄 ความแตกต่างจาก Football Match Schedule

| Feature | Schedule | Results |
|---------|----------|---------|
| Source URL | `lichthidaubongda` | `ketquabongda` |
| ข้อมูลหลัก | เวลาแข่ง + ทีม | คะแนนจริง + ผู้ชนะ |
| Score highlight | ไม่มี | 🟢🔵⚪ Win/Draw/Loss |
| Score Detail | ไม่มี | AET, 2nd Leg, Advances |
| Cache file | `schedule.html` | `results.html` |
| WP REST path | `/fms-client/v1/schedule` | `/fmr-client/v1/results` |
| JS global | `FMSClient` | `FMRClient` |
| Widget name | `football_match_schedule` | `football_match_results` |
| Shortcode | `[football_match_schedule]` | `[football_match_results]` |

---

## 📦 Tech Stack

- **Server:** PHP 8.0+, Apache (.htaccess)
- **Cache:** File-based (atomic write)
- **Client:** WordPress 5.0+, Elementor
- **Security:** AES-256-CBC, hash_equals, rate limiting
- **Timezone:** Asia/Bangkok (UTC+7)
