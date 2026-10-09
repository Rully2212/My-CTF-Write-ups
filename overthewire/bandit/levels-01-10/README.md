# Writeup Belajar Command Linux untuk OverTheWire Bandit

Writeup ini berisi rangkuman command yang sudah dipelajari saat mengerjakan level awal OverTheWire Bandit. Fokusnya bukan menghafal jawaban, tetapi memahami cara kerja command Linux yang dipakai untuk mencari dan membaca file.

> Catatan: Jangan masukkan password atau flag asli ke writeup publik seperti GitHub. Cukup tulis command, penjelasan, dan placeholder seperti `<PASSWORD_BANDIT11>`.

---

## 1. Membaca file bernama `-`

Jika ada file dengan nama:

```bash
-
```

Jangan langsung memakai:

```bash
cat -
```

Karena `cat -` akan dianggap membaca dari input keyboard atau `stdin`, bukan membaca file bernama `-`.

Command yang benar:

```bash
cat ./-
```

Alternatif:

```bash
cat -- -
```

Penjelasan:

```bash
./-
```

artinya file bernama `-` yang berada di direktori saat ini.

```bash
--
```

artinya stop membaca opsi. Jadi karakter `-` setelahnya dianggap sebagai nama file.

Jika terminal sudah terlanjur menunggu input karena menjalankan `cat -`, tekan:

```bash
CTRL + D
```

---

## 2. Membaca file yang namanya mengandung spasi dan diawali `--`

Contoh nama file:

```bash
--spaces in this filename--
```

Masalahnya ada dua:

1. Ada spasi, sehingga harus diberi tanda kutip.
2. Diawali `--`, sehingga bisa dianggap sebagai opsi command.

Command yang benar:

```bash
cat "./--spaces in this filename--"
```

Alternatif:

```bash
cat -- "--spaces in this filename--"
```

Penjelasan:

```bash
"nama file"
```

membuat shell membaca nama file sebagai satu argumen, meskipun ada spasi.

```bash
./
```

menunjukkan bahwa file tersebut berada di direktori saat ini, sehingga tidak dianggap sebagai opsi command.

---

## 3. Membaca hidden file yang diawali titik

File yang diawali titik `.` di Linux adalah hidden file.

Contoh nama file:

```bash
...Hiding-From-You
```

Untuk melihat hidden file, gunakan:

```bash
ls -la
```

Untuk membaca file tersebut:

```bash
cat ./...Hiding-From-You
```

Penjelasan penting:

```bash
.
```

artinya direktori saat ini.

```bash
..
```

artinya direktori parent atau direktori sebelumnya.

```bash
...
```

bukan perintah khusus. Jika ada file bernama `...Hiding-From-You`, maka tiga titik itu hanya bagian dari nama file.

---

## 4. Mencari file dengan `find` berdasarkan ukuran dan permission

Jika ada banyak folder seperti:

```bash
maybehere00
maybehere01
maybehere02
...
maybehere19
```

jangan dicek satu per satu. Gunakan `find`.

Contoh clue: file berukuran `1033 bytes`, human-readable, dan not executable.

Command:

```bash
find . -type f -size 1033c ! -executable -readable
```

Penjelasan:

```bash
.
```

mencari mulai dari direktori saat ini.

```bash
-type f
```

hanya mencari file, bukan folder.

```bash
-size 1033c
```

mencari file dengan ukuran tepat 1033 bytes. Huruf `c` berarti bytes.

```bash
! -executable
```

mencari file yang tidak executable.

```bash
-readable
```

mencari file yang bisa dibaca.

Setelah menemukan path file, baca isinya:

```bash
cat ./path/hasil-find
```

Contoh:

```bash
cat ./maybehere07/.file2
```

Jika ingin langsung membaca hasilnya:

```bash
find . -type f -size 1033c ! -executable -readable -exec cat {} \;
```

---

## 5. Mencari file di seluruh server

Kadang clue mengatakan file ada di "somewhere on the server". Artinya pencarian tidak cukup dari direktori saat ini saja.

Jika memakai:

```bash
find . -type f -user bandit7 -group bandit6 -size 33c
```

maka pencarian hanya dilakukan dari direktori saat ini.

Untuk mencari dari seluruh sistem, gunakan `/`:

```bash
find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null
```

Penjelasan:

```bash
/
```

mencari dari root directory atau seluruh sistem.

```bash
-user bandit7
```

mencari file yang dimiliki user `bandit7`.

```bash
-group bandit6
```

mencari file yang dimiliki group `bandit6`.

```bash
-size 33c
```

mencari file dengan ukuran 33 bytes.

```bash
2>/dev/null
```

menyembunyikan error seperti `Permission denied`.

Setelah path ditemukan, baca file-nya:

```bash
cat /path/file-yang-muncul
```

Atau langsung:

```bash
find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null -exec cat {} \;
```

---

## 6. Kumpulan command `find` yang sering dipakai

Mencari semua file:

```bash
find . -type f
```

Mencari semua folder:

```bash
find . -type d
```

Mencari file berdasarkan nama:

```bash
find . -name "file.txt"
```

Mencari file hidden:

```bash
find . -name ".*"
```

Mencari file berdasarkan ukuran tertentu:

```bash
find . -type f -size 1033c
```

Mencari file lebih besar dari 1 MB:

```bash
find . -type f -size +1M
```

Mencari file lebih kecil dari 1 KB:

```bash
find . -type f -size -1k
```

Mencari file yang bisa dibaca:

```bash
find . -type f -readable
```

Mencari file yang executable:

```bash
find . -type f -executable
```

Mencari file yang tidak executable:

```bash
find . -type f ! -executable
```

Mencari file milik user tertentu:

```bash
find . -type f -user bandit5
```

Mencari file milik group tertentu:

```bash
find . -type f -group bandit5
```

Mencari file berdasarkan permission:

```bash
find . -type f -perm 644
```

Mencari file yang punya permission executable:

```bash
find . -type f -perm /111
```

Mencari file kosong:

```bash
find . -type f -empty
```

Mencari folder kosong:

```bash
find . -type d -empty
```

Menjalankan command ke hasil `find`:

```bash
find . -type f -name "*.txt" -exec cat {} \;
```

---

## 7. Perbedaan `grep`, `cat`, `wget`, `curl`, dan `scp`

`grep` bukan untuk download file. `grep` digunakan untuk mencari teks di dalam file atau output command.

Contoh:

```bash
grep "millionth" data.txt
```

Artinya mencari baris yang mengandung kata `millionth` di file `data.txt`.

Untuk membaca file:

```bash
cat data.txt
```

Untuk download file dari URL:

```bash
wget http://example.com/file.txt
```

atau:

```bash
curl -O http://example.com/file.txt
```

Untuk download file dari server SSH ke komputer lokal:

```bash
scp -P 2220 bandit6@bandit.labs.overthewire.org:/path/file .
```

Penjelasan:

```bash
-P 2220
```

menggunakan port SSH 2220.

```bash
:/path/file
```

lokasi file di server.

```bash
.
```

simpan file di direktori lokal saat ini.

Namun untuk Bandit, biasanya tidak perlu download file. Cukup baca atau proses file langsung di server.

---

## 8. Mencari baris yang muncul satu kali

Jika ingin mencari baris yang hanya muncul sekali di sebuah file, gunakan kombinasi `sort` dan `uniq`.

Command:

```bash
sort data.txt | uniq -u
```

Penjelasan:

```bash
sort data.txt
```

mengurutkan isi file supaya baris yang sama berdekatan.

```bash
uniq -u
```

menampilkan baris yang unik, yaitu baris yang hanya muncul satu kali.

Jika ingin melihat jumlah kemunculan setiap baris:

```bash
sort data.txt | uniq -c
```

Jika ingin menampilkan baris yang jumlah kemunculannya 1:

```bash
sort data.txt | uniq -c | grep " 1 "
```

Command paling ringkas untuk Bandit level ini:

```bash
sort data.txt | uniq -u
```

---

## 9. Mencari string yang bisa dibaca manusia

Untuk mencari teks yang bisa dibaca manusia dari file binary, gunakan `strings`.

Command dasar:

```bash
strings data.txt
```

Jika output terlalu banyak, gabungkan dengan `grep`.

Contoh mencari baris yang mengandung tanda `=`:

```bash
strings data.txt | grep "="
```

Mencari kata tertentu:

```bash
strings data.txt | grep "password"
```

Mengabaikan huruf besar atau kecil:

```bash
strings data.txt | grep -i "password"
```

Alur yang sering dipakai:

```bash
file data.txt
strings data.txt
strings data.txt | grep "="
```

Penjelasan:

```bash
file data.txt
```

melihat jenis file.

```bash
strings data.txt
```

mengambil teks yang bisa dibaca dari file binary.

```bash
grep "="
```

menyaring output yang mengandung tanda `=`.

---

## 10. Decode Base64

Jika file sudah berisi teks base64, jangan di-encode lagi.

Command yang salah:

```bash
base64 data.txt
```

Command di atas akan melakukan encode lagi.

Untuk decode base64, gunakan:

```bash
base64 -d data.txt
```

atau:

```bash
base64 --decode data.txt
```

Setelah mendapatkan password, login ke level berikutnya:

```bash
ssh bandit11@bandit.labs.overthewire.org -p 2220
```

Alur lengkap:

```bash
ls
cat data.txt
base64 -d data.txt
ssh bandit11@bandit.labs.overthewire.org -p 2220
```

---

## 11. Decode ROT13 menggunakan `tr`

ROT13 adalah teknik mengganti huruf dengan huruf lain yang posisinya bergeser 13 langkah.

Command untuk ROT13:

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Alternatif tanpa `cat`:

```bash
tr 'A-Za-z' 'N-ZA-Mn-za-m' < data.txt
```

Penjelasan:

```bash
A-Za-z
```

berarti semua huruf besar `A-Z` dan semua huruf kecil `a-z`.

```bash
N-ZA-Mn-za-m
```

berarti alfabet yang sudah diputar 13 posisi.

Peta ROT13 huruf besar:

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZ
NOPQRSTUVWXYZABCDEFGHIJKLM
```

Artinya:

```text
A -> N
B -> O
C -> P
D -> Q
E -> R
F -> S
G -> T
H -> U
I -> V
J -> W
K -> X
L -> Y
M -> Z
N -> A
O -> B
P -> C
...
Z -> M
```

Bagian:

```bash
N-ZA-M
```

adalah cara singkat menulis:

```text
NOPQRSTUVWXYZABCDEFGHIJKLM
```

Karena:

```bash
N-Z
```

berarti huruf `N` sampai `Z`.

```bash
A-M
```

berarti huruf `A` sampai `M`.

Jadi:

```bash
N-ZA-M
```

sama dengan:

```text
NOPQRSTUVWXYZABCDEFGHIJKLM
```

Untuk huruf kecil:

```bash
n-za-m
```

sama dengan:

```text
nopqrstuvwxyzabcdefghijklm
```

Contoh:

```bash
echo "hello" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Output:

```text
uryyb
```

Jika dijalankan lagi:

```bash
echo "uryyb" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Output:

```text
hello
```

ROT13 unik karena encode dan decode menggunakan command yang sama.

---

## 12. Apa itu `tr`?

`tr` adalah command Linux untuk translate karakter. Fungsinya bisa untuk mengganti, menghapus, atau memadatkan karakter dari input teks.

Format dasar:

```bash
tr 'karakter_asal' 'karakter_pengganti'
```

Contoh mengganti huruf:

```bash
echo "abc" | tr 'abc' '123'
```

Output:

```text
123
```

Artinya:

```text
a -> 1
b -> 2
c -> 3
```

Mengubah huruf kecil menjadi huruf besar:

```bash
echo "hello" | tr 'a-z' 'A-Z'
```

Output:

```text
HELLO
```

Menghapus spasi:

```bash
echo "h e l l o" | tr -d ' '
```

Output:

```text
hello
```

Menghapus angka:

```bash
echo "abc123" | tr -d '0-9'
```

Output:

```text
abc
```

---

## 13. Error umum pada `tr`

Contoh command yang salah:

```bash
cat data.txt | tr 'A-Za-z'
```

Error:

```text
tr: missing operand after 'A-Za-z'
Two strings must be given when translating.
```

Penyebabnya adalah `tr` membutuhkan dua string:

```bash
tr 'karakter_asal' 'karakter_tujuan'
```

Command yang benar untuk ROT13:

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

---

## 14. Ringkasan command penting

Membaca file biasa:

```bash
cat data.txt
```

Membaca file bernama `-`:

```bash
cat ./-
```

Membaca file dengan spasi:

```bash
cat "./--spaces in this filename--"
```

Melihat hidden file:

```bash
ls -la
```

Mencari file berdasarkan ukuran:

```bash
find . -type f -size 1033c
```

Mencari file di seluruh sistem:

```bash
find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null
```

Mencari teks di file:

```bash
grep "kata" data.txt
```

Mencari baris yang hanya muncul sekali:

```bash
sort data.txt | uniq -u
```

Mencari string dari file binary:

```bash
strings data.txt
```

Decode base64:

```bash
base64 -d data.txt
```

Decode ROT13:

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Login SSH Bandit:

```bash
ssh banditX@bandit.labs.overthewire.org -p 2220
```

Ganti `banditX` dengan level yang ingin dimasuki, misalnya `bandit11`.

---

