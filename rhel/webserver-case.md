# Setup Apache HTTP Server di Red Hat Enterprise Linux

Dokumentasi langkah demi langkah untuk mengubah server **RHEL** menjadi web server **Apache HTTP Server (httpd)**
yang menampilkan halaman *Red Hat Academy Day*, lengkap dengan firewall, log, dan troubleshooting dasar.

Semua command bisa langsung di-copy dan di-paste ke terminal (`Ctrl+Shift+V`).
Blok hanya berisi command; output yang diharapkan ditulis di bawahnya.

## Environment

Mengikuti environment kelas Red Hat Academy (RH124/RH134):

| Machine | IP | Peran |
|---|---|---|
| `workstation.lab.example.com` | 172.25.250.9 | Desktop (GUI): terminal dan Firefox |
| `servera.lab.example.com` | 172.25.250.10 | **Server target**: tempat Apache dipasang |
| `serverb.lab.example.com` | 172.25.250.11 | Server kedua (opsional) |
| `bastion.lab.example.com` | 172.25.250.254 | Gateway ke classroom (harus selalu berjalan) |
| `classroom.example.com` | 172.25.254.254 | Server materi/repository kelas |

Label di setiap blok:

- **[workstation]**: terminal di `workstation`
- **[servera]**: terminal yang sudah `ssh` ke `servera` (prompt `[student@servera ~]$`)

Asumsi: login sebagai `student`, SSH ke `servera` tanpa password, `sudo` tanpa password, `dnf` bisa mengambil
package dari repository kelas, `firewalld` aktif, dan SELinux `Enforcing`.

> Command di dokumen ini mengubah konfigurasi server. Jalankan hanya di lab atau VM milikmu.
> Pakai VM RHEL-family sendiri? Jalankan semuanya di satu mesin dan ganti `servera.lab.example.com` dengan `localhost`.

---

## 1. Masuk ke server

**[workstation]**

```bash
ssh student@servera
```

Prompt berubah menjadi `[student@servera ~]$`. Semua langkah berikutnya dijalankan di `servera` kecuali ada label lain.

## 2. Cek kondisi awal

**[servera]**

```bash
whoami
hostname
cat /etc/redhat-release
hostname -I
rpm -q httpd
sudo dnf -q info httpd
systemctl is-active firewalld
sudo firewall-cmd --list-services
getenforce
```

| Command | Yang diharapkan |
|---|---|
| `whoami` | `student` |
| `hostname` | `servera.lab.example.com` |
| `cat /etc/redhat-release` | `Red Hat Enterprise Linux release 9.x ...` |
| `hostname -I` | `172.25.250.10` |
| `rpm -q httpd` | `package httpd is not installed` |
| `dnf -q info httpd` | Informasi package muncul (repository tersedia) |
| `systemctl is-active firewalld` | `active` |
| `firewall-cmd --list-services` | Ada `ssh`, belum ada `http` |
| `getenforce` | `Enforcing` |

## 3. Install Apache

```bash
rpm -q httpd
sudo dnf install -y httpd
rpm -q httpd
```

Sebelum install: `package httpd is not installed`. Sesudahnya muncul nama dan versi package (`httpd-2.4.x-...`).

## 4. Start dan enable service

```bash
systemctl status httpd
sudo systemctl start httpd
sudo systemctl enable httpd
systemctl is-active httpd
systemctl is-enabled httpd
```

Status awal `inactive (dead)`. Setelah itu `active` dan `enabled`.

- `start` = nyalakan **sekarang**.
- `enable` = nyalakan **otomatis setelah reboot**.

## 5. Siapkan dan deploy website

Buat file `index.html` (paste seluruh blok, dari `mkdir` sampai `EOF`):

```bash
mkdir -p ~/rha-demo
cat > ~/rha-demo/index.html <<'EOF'
<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Red Hat Academy Day - Politeknik Negeri Padang</title>
<style>
body{margin:0;min-height:100vh;display:flex;align-items:center;justify-content:center;background:#151515;color:#fff;font-family:Arial,Helvetica,sans-serif}
main{max-width:820px;padding:48px;border-left:8px solid #e00000}
.k{color:#e00000;font-weight:bold;letter-spacing:2px;font-size:14px}
h1{font-size:46px;margin:16px 0;line-height:1.1}
p{color:#c7c7c7;font-size:20px;line-height:1.5}
.f{margin-top:32px;color:#8a8d90;font-size:14px}
</style>
</head>
<body>
<main>
<div class="k">RED HAT ACADEMY DAY &middot; POLITEKNIK NEGERI PADANG</div>
<h1>Hello from a Linux server!</h1>
<p>Halaman ini dikirim oleh Apache HTTP Server (httpd) yang berjalan di Red Hat Enterprise Linux, lalu melewati network dan firewall sampai ke browser kamu.</p>
<div class="f">Build the foundation. Use your resources. Practice consistently.</div>
</main>
</body>
</html>
EOF
```

Salin ke folder website Apache:

```bash
sudo cp ~/rha-demo/index.html /var/www/html/index.html
ls -l /var/www/html
head /var/www/html/index.html
```

File `index.html` ada di `/var/www/html` dan isinya berupa HTML.

> Gunakan `cp`, bukan `mv`. File hasil `cp` mengikuti SELinux context folder tujuan,
> sedangkan `mv` membawa context lama dan bisa menyebabkan `403 Forbidden`.

## 6. Tes dari dalam server

```bash
curl http://localhost
curl -I http://localhost
```

Output pertama berupa isi HTML. Output kedua `HTTP/1.1 200 OK`.

## 7. Tes dari luar sebelum firewall dibuka (opsional)

**[workstation]** (terminal baru di workstation, bukan di servera)

```bash
curl -I http://servera.lab.example.com
```

Gagal dengan `curl: (7) Failed to connect ... No route to host`.
Apache sudah jalan, tetapi firewall di server menolak koneksi dari luar.

## 8. Izinkan akses HTTP di firewall

**[servera]**

```bash
sudo firewall-cmd --list-services
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

Sebelumnya `http` tidak ada di daftar; setelah `--reload`, `http` muncul.

- `--permanent` = simpan agar bertahan setelah reboot.
- `--reload` = terapkan sekarang.

## 9. Akses dari client

**[workstation]**

```bash
curl -I http://servera.lab.example.com
```

Diharapkan `HTTP/1.1 200 OK`. Lalu buka di **Firefox pada workstation**:

```text
http://servera.lab.example.com
```

Halaman *Hello from a Linux server!* tampil.

## 10. Lihat log

**[servera]**

```bash
sudo tail -n 5 /var/log/httpd/access_log
```

Muncul baris dengan IP workstation (`172.25.250.9`), `GET / HTTP/1.1`, dan status `200`.

Untuk memantau real-time, jalankan di terminal lain, refresh Firefox, lalu hentikan dengan `Ctrl+C`:

```bash
sudo tail -f /var/log/httpd/access_log
```

## 11. Verifikasi

**[servera]**

```bash
rpm -q httpd
systemctl is-active httpd
systemctl is-enabled httpd
ls -l /var/www/html/index.html
sudo firewall-cmd --list-services
sudo grep 'GET / ' /var/log/httpd/access_log | tail -n 3
```

**[workstation]**

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://servera.lab.example.com
```

| # | Kriteria | Bukti |
|---|---|---|
| 1 | Apache package tersedia | `rpm -q httpd` menampilkan nama package |
| 2 | Service `httpd` berjalan | `active` |
| 3 | Service `httpd` enabled | `enabled` |
| 4 | File website tersedia | `ls -l` menampilkan `/var/www/html/index.html` |
| 5 | HTTP diizinkan di firewall | `http` ada di daftar services |
| 6 | Website dapat diakses | `curl` dari workstation menampilkan `200` |
| 7 | Request terlihat di log | ada baris `GET / HTTP/1.1` |

---

## 12. Troubleshooting: website tidak bisa diakses

Pola: **Observe the symptom → Check the relevant component → Read the system response → Apply the fix → Verify the result.**

Contoh: service Apache berhenti.

**[servera]** Simulasikan masalah:

```bash
sudo systemctl stop httpd
```

**[workstation]** 1. Observe the symptom (refresh Firefox, atau):

```bash
curl -I http://servera.lab.example.com
```

Hasil: `Connection refused`. Servernya terjangkau, tetapi tidak ada program yang mendengarkan di port 80.

**[servera]** 2. Check dan 3. Read:

```bash
systemctl status httpd
```

Hasil: `Active: inactive (dead)`.

**[servera]** 4. Apply the fix:

```bash
sudo systemctl start httpd
```

**[servera]** 5. Verify:

```bash
systemctl is-active httpd
curl -I http://localhost
```

**[workstation]**

```bash
curl -I http://servera.lab.example.com
```

Hasil: `active` dan `HTTP/1.1 200 OK`; Firefox kembali menampilkan halaman.

### Membaca gejala

| Gejala dari `curl` / browser | Artinya | Cek pertama |
|---|---|---|
| `Connection refused` | Server terjangkau, tidak ada program yang mendengarkan di port 80 | `systemctl status httpd` |
| `No route to host` | Firewall di server menolak | `sudo firewall-cmd --list-services` |
| Timeout | Paket di-drop atau jalur network bermasalah | network, firewall |
| `403 Forbidden` | Apache jalan, tetapi tidak boleh membaca file | `ls -l`, `ls -lZ` |
| `404 Not Found` | Apache jalan, file tidak ada | path dan nama file |

### Masalah umum dan perbaikannya

**Service gagal start** setelah mengubah konfigurasi:

```bash
systemctl status httpd
sudo journalctl -xeu httpd
sudo apachectl configtest
```

Perbaiki file konfigurasi yang ditunjuk pesan error, lalu:

```bash
sudo apachectl configtest
sudo systemctl restart httpd
```

**`403 Forbidden`** karena permission file:

```bash
ls -l /var/www/html
sudo chmod 644 /var/www/html/index.html
```

**`403 Forbidden`** padahal permission normal (SELinux context salah):

```bash
ls -lZ /var/www/html
sudo restorecon -Rv /var/www/html
```

Context yang benar untuk file website adalah `httpd_sys_content_t`.

**`No route to host`** padahal Apache sehat (`curl -I http://localhost` berhasil): lakukan [langkah 8](#8-izinkan-akses-http-di-firewall).

Lihat juga log error Apache:

```bash
sudo tail /var/log/httpd/error_log
```

---

## 13. Reset ke kondisi awal

Menghapus package `httpd`, file website, log, dan aturan firewall `http` (file `~/rha-demo/index.html` tetap ada).
Error pada baris pertama itu normal jika `httpd` belum pernah dipasang.

**[servera]**

```bash
sudo systemctl disable --now httpd
sudo firewall-cmd --permanent --remove-service=http
sudo firewall-cmd --reload
sudo dnf remove -y httpd
sudo rm -f /var/www/html/index.html
sudo rm -rf /var/log/httpd
```

---

## Pemetaan XAMPP ke Linux server

| Di XAMPP | Di Linux server (RHEL) |
|---|---|
| Install XAMPP | `sudo dnf install -y httpd` |
| Klik **Start** pada Apache | `sudo systemctl start httpd` |
| Auto-start | `sudo systemctl enable httpd` |
| Indikator hijau | `systemctl status httpd` |
| `htdocs/index.html` | `/var/www/html/index.html` |
| `localhost` | `localhost` dan `http://servera.lab.example.com` |
| Tombol Logs | `/var/log/httpd/access_log` |
| (tidak terasa di desktop) | Firewall: `firewall-cmd --add-service=http` |

*Same concepts. Different interface. Deeper visibility and control.*

XAMPP sangat baik untuk belajar dan development di desktop. Linux server memberi visibility dan kontrol lebih dalam untuk server administration.
