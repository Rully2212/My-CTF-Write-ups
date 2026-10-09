# Laporan Writeup: TryHackme Neighbour CTF (IDOR)
**Penulis:** Rully Miftahur Rozaq

## Pendahuluan
Laporan ini menjelaskan langkah-langkah penyelesaian tantangan Neighbour CTF di platform TryHackMe yang berfokus pada eksploitasi kerentanan IDOR (Insecure Direct Object Reference). IDOR adalah celah keamanan di mana pengguna dapat memanipulasi parameter ID untuk mengakses data milik pengguna lain.

---

## Langkah-Langkah Penyelesaian

### 1. Persiapan dan Akses Target
* Langkah pertama adalah menyalakan mesin di TryHackMe untuk mendapatkan alamat IP target.
* Mengakses IP yang tersedia. Dalam kasus ini, IP target adalah `10.48.173.31`, yang menampilkan halaman formulir login.

### 2. Mendapatkan Kredensial Guest
* Karena belum memiliki akun, pengecekan dilakukan pada kode sumber halaman dengan menekan tombol CTRL + U pada keyboard.
* Ditemukan informasi akun guest pada source code, kemudian login dilakukan menggunakan username `guest` dan password `guest`.

### 3. Eksploitasi IDOR
* Setelah login sebagai guest, website menampilkan pesan peringatan untuk tidak mengintip profil pengguna lain.
* Pengujian IDOR dilakukan dengan mengganti parameter user pada URL dari `guest` menjadi `admin`.
  * **URL Awal:** `http://10.48.173.31/profile.php?user=guest`
  * **URL Modifikasi:** `http://10.48.173.31/profile.php?user=admin`

---

## Hasil Akhir
Setelah berhasil mengubah parameter ID menjadi `admin`, sistem mengarahkan ke halaman profil admin dan menampilkan flag yang dicari.

**Flag:**
> `flag{66be95c478473d91a5358f2440c7af1f}`
