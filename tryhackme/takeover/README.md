
# TryHackMe - TakeOver CTF Writeup
**Author:** Rully Miftahur R  

---

# 📌 Overview
Challenge ini berasal dari platform **TryHackMe** dengan fokus pada **Subdomain Takeover**.  
Tujuan dari challenge ini adalah melakukan reconnaissance terhadap domain target, menemukan subdomain yang tersedia, dan mengidentifikasi kemungkinan takeover hingga mendapatkan **flag**.

---

# 🧭 Target Information

| Item | Value |
|-----|------| Target IP | 10.49.153.93 |
| Domain | futurevera.thm |
| Tools | ping, nmap, gobuster |
| Attack Type | Subdomain Enumeration / Takeover |

---

# 1️⃣ Connectivity Test

Langkah pertama adalah memastikan bahwa target dapat dijangkau dari mesin attacker.

```bash
ping 10.49.153.93
```

Jika host merespon ping, maka dapat dipastikan bahwa koneksi ke target sudah berhasil.

---

# 2️⃣ Reconnaissance (Port Scanning)

Selanjutnya dilakukan scanning menggunakan **Nmap** untuk melihat service dan port yang terbuka pada target.

```bash
nmap -sV 10.49.153.93
```

Parameter yang digunakan:

| Parameter | Fungsi |
|-----------|--------|
| -sV | mendeteksi versi service yang berjalan pada port |

Dari proses ini kita bisa mengetahui service apa saja yang aktif pada server.

---

# 3️⃣ Domain Discovery

Pada deskripsi challenge terdapat domain berikut:

```
https://futurevera.thm
```

Namun domain ini belum dapat diakses langsung karena belum dikenali oleh sistem lokal.

---

# 4️⃣ Konfigurasi /etc/hosts

Agar domain bisa diakses, kita perlu menambahkan mapping IP dan domain pada file:

```
/etc/hosts
```

Contoh konfigurasi:

```
10.49.153.93 futurevera.thm
```

Setelah disimpan, domain dapat diakses melalui browser.

---

# 5️⃣ Subdomain Enumeration

Setelah domain utama dapat diakses, langkah berikutnya adalah mencari **subdomain** yang mungkin tersedia.

Tool yang digunakan adalah **Gobuster**.

```bash
gobuster vhost -k -u https://futurevera.thm -w /usr/share/wordlists/dirb/common.txt
```

Penjelasan parameter:

| Parameter | Fungsi |
|----------|--------|
| vhost | melakukan virtual host enumeration |
| -k | mengabaikan TLS verification |
| -u | target URL |
| -w | wordlist yang digunakan |

---

# 6️⃣ Hasil Enumeration

Dari proses scanning ditemukan beberapa subdomain:

- `support.futurevera.thm`
- `blog.futurevera.thm`

Subdomain **blog.futurevera.thm** memberikan response:

```
421 Misdirected Request
```

Kode HTTP tersebut menunjukkan bahwa request dikirim ke server yang tidak dikonfigurasi untuk merespon domain tersebut.

---

# 7️⃣ Menambahkan Subdomain ke Hosts

Karena subdomain tidak langsung bisa diakses melalui browser, maka perlu ditambahkan ke file:

```
/etc/hosts
```

Contoh:

```
10.49.153.93 support.futurevera.thm
10.49.153.93 blog.futurevera.thm
```

Setelah konfigurasi ini, subdomain dapat diakses melalui browser.

---

# 8️⃣ Analisis Certificate

Saat mengakses:

```
support.futurevera.thm
```

Saya memeriksa **SSL Certificate** dan menemukan informasi tambahan terkait **DNS yang digunakan oleh domain tersebut**.

Informasi ini menjadi petunjuk penting untuk melanjutkan eksploitasi.

---

# 9️⃣ Subdomain Takeover

Setelah menemukan konfigurasi DNS yang digunakan, saya menyesuaikan kembali konfigurasi pada `/etc/hosts`.

Dengan konfigurasi yang tepat, domain akhirnya dapat diakses secara penuh.

Pada tahap ini saya berhasil menemukan **flag** yang menjadi tujuan dari challenge ini.

---

# 🏁 Flag

```
FLAG_FOUND_HERE
```

---

# 🧰 Tools Used

- Nmap
- Gobuster
- Linux CLI
- Browser (for certificate inspection)

---

# 📚 Key Learning

Beberapa hal yang dipelajari dari challenge ini:

- Cara melakukan **reconnaissance pada target**
- Teknik **subdomain enumeration**
- Memahami **HTTP response code**
- Analisis **SSL certificate**
- Konsep **Subdomain Takeover**

---

# 📖 Conclusion

Challenge ini menunjukkan pentingnya konfigurasi DNS dan subdomain yang benar.  
Kesalahan konfigurasi pada subdomain dapat membuka peluang bagi attacker untuk melakukan **subdomain takeover**, yang dapat digunakan untuk phishing, hosting malware, atau mengambil alih traffic domain.

