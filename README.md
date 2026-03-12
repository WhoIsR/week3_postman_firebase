# Panduan Pengujian API: Firebase Auth & Backend via Postman

Panduan ini mendokumentasikan langkah demi langkah pengujian alur otentikasi (Register, Verifikasi Email, dan Login) menggunakan **Firebase Identity Toolkit REST API** dan **Postman**. 
Dokumentasi ini dibuat untuk memastikan setiap *endpoint* berfungsi dengan baik sebelum diimplementasikan ke dalam kode aplikasi.

---

## Tahap 1: Persiapan Firebase Console

Sebelum mulai menembak API, Firebase harus disiapkan terlebih dahulu.

1. **Buat Project Baru**: Buka Firebase Console dan buat *project* baru.
2. **Aktifkan Email/Password Auth**: Masuk ke menu `Authentication` > `Sign-in method`. Pilih *provider* **Email/Password** dan aktifkan *toggle* yang pertama.
3. **Dapatkan Web API Key**: Masuk ke pengaturan *project* (ikon gerigi) > `Project settings` > tab `General`. Salin kode pada bagian **Web API Key** (biasanya diawali dengan `AIzaSy`).

> **💡 Catatan Penting:** Jika Web API Key tidak muncul, pancing dengan menambahkan aplikasi Web (`</>`) kosong di bagian "Your apps".

---

## Tahap 2: Setup Environment di Postman

Agar pengujian lebih efisien dan tidak perlu melakukan *copy-paste* berulang kali, kita akan memanfaatkan fitur **Environment** di Postman.

Buat Environment baru dan tambahkan konfigurasi variabel berikut:

* `FIREBASE_API_KEY`: Paste Web API Key dari Firebase di sini.
* `USER_EMAIL`: Isi dengan email untuk keperluan *testing*.
* `USER_PASSWORD`: Isi dengan password yang aman (minimal 6 karakter).
* `FIREBASE_ID_TOKEN`: Biarkan kosong (akan terisi otomatis oleh *script*).

---

## Tahap 3: Eksekusi Pengujian API (End-to-End)

Pastikan untuk mengeksekusi langkah-langkah di bawah ini secara berurutan agar alur otentikasi tidak bermasalah.

### Langkah 1: Registrasi Akun Baru (signUp)
Endpoint ini digunakan untuk mendaftarkan *user* baru ke dalam *database* Firebase.

* **Method:** `POST`
* **URL:** `https://identitytoolkit.googleapis.com/v1/accounts:signUp?key={{FIREBASE_API_KEY}}`
* **Headers:** `Content-Type: application/json`
* **Body (raw > JSON):**
  ```json
  {
    "email": "{{USER_EMAIL}}",
    "password": "{{USER_PASSWORD}}",
    "returnSecureToken": true
  }
  ```

> ** Tips:** Tambahkan *script* berikut di tab **Tests** Postman agar *idToken* dari server otomatis tersimpan ke *Environment*:
> ```javascript
> let json = pm.response.json();
> if (pm.response.code === 200) {
>     pm.environment.set("FIREBASE_ID_TOKEN", json.idToken);
>     pm.environment.set("FIREBASE_LOCAL_ID", json.localId);
>     pm.environment.set("FIREBASE_REFRESH_TOKEN", json.refreshToken);
> }
> ```

<img width="1092" height="605" alt="Screenshot Success Register" src="https://github.com/user-attachments/assets/873710ea-d7d7-46f6-97dd-ac64eb182b7a" />
<br><em>Respon 200 OK saat berhasil melakukan registrasi</em>

<br><br>

<img width="1080" height="441" alt="Screenshot gagal Register" src="https://github.com/user-attachments/assets/c023f2d0-298c-430d-8887-e3f4af638090" />
<br><em>Respon 400 Bad saat gagal karena email sudah terdaftar</em>

<br><br>

<img width="1059" height="445" alt="Screenshot gagal Register" src="https://github.com/user-attachments/assets/5a7210ae-377d-4b0f-abeb-05933828786e" />
<br><em>Respon 400 Bad saat gagal karena mengisi password kurang dari 6 karakter</em>

---

### Langkah 2: Pengiriman Email Verifikasi (sendOobCode)
Akun yang baru dibuat statusnya belum terverifikasi. Kita harus meminta Firebase mengirimkan tautan konfirmasi ke *email yang di daftarkan*.

* **Method:** `POST`
* **URL:** `https://identitytoolkit.googleapis.com/v1/accounts:sendOobCode?key={{FIREBASE_API_KEY}}`
* **Headers:** `Content-Type: application/json`
* **Body (raw > JSON):**
  ```json
  {
    "requestType": "VERIFY_EMAIL",
    "idToken": "{{FIREBASE_ID_TOKEN}}"
  }
  ```

> **Tips:** Setelah mendapat respon 200 OK, Anda bisa membuka kotak masuk email (biasanya ada di bagian spam) dan mengklik tautan verifikasi dari Firebase. Jika diabaikan, token selamanya tidak akan memuat status `email_verified: true`!
<img width="1750" height="668" alt="image" src="https://github.com/user-attachments/assets/8167e4c2-d8c1-41cb-8b40-d01aa12dd660" />
<img width="1157" height="481" alt="image" src="https://github.com/user-attachments/assets/90c51fd4-d819-43cc-b37a-3557a04f9c62" />

---

### Langkah 3: Cek Status Verifikasi (lookup)
Langkah ini digunakan untuk membuktikan apakah sistem Firebase sudah mencatat perubahan status setelah *user* mengklik tautan verifikasi di email. Karena pengujian ini murni menggunakan Firebase tanpa Backend, kita menggunakan *endpoint* `lookup`.

* **Method:** `POST`
* **URL:** `https://identitytoolkit.googleapis.com/v1/accounts:lookup?key={{FIREBASE_API_KEY}}`
* **Headers:** `Content-Type: application/json`
* **Body (raw > JSON):**
  ```json
  {
    "idToken": "{{FIREBASE_ID_TOKEN}}"
  }
  ```

Ketika di-*send* sebelum email diklik, respon JSON pada bagian `users` akan menampilkan `"emailVerified": false`. Namun, jika Anda mengirim ulang request ini **setelah** mengklik tautan di email, statusnya akan langsung berubah menjadi `"emailVerified": true`.

<img width="331" height="67" alt="image" src="https://github.com/user-attachments/assets/2019eebf-dae4-4ecb-bd67-e62dbdd3be79" />
*Respon dari accounts:lookup yang menampilkan status emailVerified: false*

<br><br>

<img width="320" height="69" alt="image" src="https://github.com/user-attachments/assets/e455a197-d722-4f08-9a9a-a6828134b480" />
*Respon dari accounts:lookup yang menampilkan status emailVerified: True*

---

### Langkah 4: Login Ulang (signInWithPassword)
**Catatan Teknis:** Meskipun belum verifikasi email, Firebase tetap akan mengizinkan proses *login* karena tugas utamanya di tahap ini hanyalah mencocokkan email dan *password*. Namun, jika diteruskan ke sistem *Backend*, token dari *login* yang belum diverifikasi pasti akan ditolak.

Oleh karena itu, setelah status di Langkah 3 menunjukkan `true`, lakukan *login* ulang. Hal ini memicu Firebase untuk menerbitkan `idToken` BARU yang sudah ter-*update* dengan informasi bahwa email telah diverifikasi.

* **Method:** `POST`
* **URL:** `https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key={{FIREBASE_API_KEY}}`
* **Headers:** `Content-Type: application/json`
* **Body (raw > JSON):**
  ```json
  {
    "email": "{{USER_EMAIL}}",
    "password": "{{USER_PASSWORD}}",
    "returnSecureToken": true
  }

---

## Troubleshooting

Dalam pengujian API, kita tidak hanya menguji skenario berhasil, tetapi juga wajib mengetahui respons sistem jika terjadi kesalahan. Berikut adalah daftar larangan, *error* bawaan Firebase, dan *bug* teknis yang mungkin kamu temui:

### 1. Copy-Paste 
Ini adalah *error* yang mungkin paling umum terjadi dan masalahnya Postman tidak akan mengeluarkan peringatan apa pun. Jika Anda menyalin (*copy-paste*) *script test* atau format JSON langsung dari dokumen PDF atau Word, sering kali ada "karakter gaib" (seperti *zero-width space* atau format spasi yang salah) yang ikut terbawa.

Akibatnya, saat dijalankan, eksekusi JavaScript di Postman akan mati di tengah jalan (*silent crash*). Meskipun status respons menunjukkan 200 OK, perintah untuk menyimpan token ke dalam Environment akan gagal dieksekusi, sehingga variabel Anda tetap kosong. **Solusi:** Selalu ketik *script test* secara manual di Postman, atau bersihkan teks menggunakan *Notepad* biasa sebelum di- *paste*.

### 2. Kesalahan Saat Registrasi (signUp)
Jangan mencoba mendaftar menggunakan email yang sudah pernah terdaftar sebelumnya di dalam *database*. Jika Anda nekat melakukannya, Firebase akan merespons dengan status 400 Bad Request dan pesan `EMAIL_EXISTS`. Solusi utamanya adalah menghapus akun tersebut di Firebase Console pada menu Authentication, lalu ulangi registrasi. 

Selain itu, Firebase secara bawaan memiliki aturan keamanan. Jika Anda menguji pendaftaran dengan *password* kurang dari 6 karakter, sistem akan menolak dengan *error* `WEAK_PASSWORD`. Jika penulisan email asal-asalan (tidak menggunakan format @domain.com), respons yang keluar adalah `INVALID_EMAIL`. Terakhir, jika di Tahap 1 Anda lupa mengaktifkan *provider* Email/Password, seluruh *request* akan diblokir dari awal dengan pesan `OPERATION_NOT_ALLOWED`.

### 3. Kesalahan Saat Mengelola Token (sendOobCode & lookup)
Satu fakta teknis yang wajib diingat: `idToken` dari Firebase hanya memiliki masa aktif maksimal 1 jam. Jika Anda mendaftar, lalu menunda pengujian pengiriman email verifikasi hingga berjam-jam kemudian, *request* Anda akan gagal dengan *error* `INVALID_ID_TOKEN`. Hal ini juga berlaku jika variabel token di Environment Anda terhapus atau kosong akibat *bug copy-paste* di atas. Untuk mengatasinya, Anda harus melakukan proses Login ulang agar mendapatkan token yang baru.

Jangan pula mencoba melakukan *spam* dengan menekan tombol *Send* berkali-kali saat meminta pengiriman email verifikasi. Firebase memiliki sistem perlindungan yang akan memblokir alamat IP Anda sementara waktu dan mengeluarkan peringatan `TOO_MANY_ATTEMPTS_TRY_LATER`. 

### 4. Kesalahan Autentikasi Saat Login (signInWithPassword)
Ketika melakukan pengujian *login*, pastikan data yang dimasukkan akurat. Kesalahan penulisan kata sandi akan memicu respons `INVALID_PASSWORD`. Jika Anda mencoba menguji proses *login* menggunakan email acak yang belum pernah melewati proses *signUp* di Langkah 1, Firebase akan memberikan respons `EMAIL_NOT_FOUND`.
