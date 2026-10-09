# Instalasi `tea` (Gitea CLI) di Windows

`tea` adalah CLI resmi dari Gitea untuk mengelola issue, pull request, release, dan fitur platform lainnya langsung dari terminal. Panduan ini menjelaskan cara memasangnya di Windows, menghubungkannya ke server Gitea, dan memakainya dengan aman, termasuk bila dipakai oleh AI agent.

> **Catatan peran.** `tea` mengurus operasi platform (issue, PR, review, release). Operasi git biasa (`clone`, `commit`, `push`) tetap memakai `git` dengan SSH key atau credential helper. `tea` tidak menggantikan `git`.

## Daftar isi

1. [Prasyarat](#prasyarat)
2. [Instalasi](#instalasi)
3. [Membuat Access Token di Gitea](#membuat-access-token-di-gitea)
4. [Login ke server Gitea](#login-ke-server-gitea)
5. [Pemakaian dasar](#pemakaian-dasar)
6. [Catatan keamanan untuk AI agent](#catatan-keamanan-untuk-ai-agent)
7. [Pemecahan masalah](#pemecahan-masalah)
8. [Metode instalasi lain](#metode-instalasi-lain)
9. [Referensi](#referensi)

## Prasyarat

- Windows dengan PowerShell
- Akun di server Gitea yang dituju
- Hak untuk membuat Access Token di akun tersebut

## Instalasi

Cara berikut memakai binary siap pakai, jalur paling sederhana karena tidak butuh Go, Make, atau Docker.

### 1. Unduh binary

Buka <https://dl.gitea.com/tea/>, masuk ke folder versi terbaru, lalu unduh file untuk Windows 64-bit (nama file memuat `windows-amd64` dan berakhiran `.exe`).

### 2. Taruh di folder tetap dan ganti nama

Buat folder `C:\tools\tea\`, pindahkan file hasil unduhan ke sana, lalu ubah namanya menjadi `tea.exe`.

```powershell
New-Item -ItemType Directory -Force C:\tools\tea
Rename-Item "C:\tools\tea\<nama-file-yang-diunduh>.exe" "tea.exe"
Test-Path C:\tools\tea\tea.exe   # harus True
```

### 3. Tambahkan ke PATH

Perintah berikut hanya menambahkan ke PATH **user** dan tidak membuat entri ganda.

```powershell
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
if ($userPath -notlike "*C:\tools\tea*") {
  [Environment]::SetEnvironmentVariable("Path", "$userPath;C:\tools\tea", "User")
}
```

Perubahan PATH permanen hanya terbaca oleh terminal yang dibuka **setelahnya**. Untuk memakainya langsung di sesi yang sedang berjalan:

```powershell
$env:Path += ";C:\tools\tea"
```

### 4. Verifikasi

Buka terminal baru, lalu jalankan:

```powershell
tea --version
```

Jika versi tampil, instalasi selesai. Bila `tea` dipakai oleh IDE atau agent (misalnya Antigravity), **restart IDE tersebut** agar membaca PATH yang baru.

## Membuat Access Token di Gitea

`tea` masuk ke server Gitea memakai Access Token. Token berperan seperti kunci akses atas nama akun pemiliknya, dengan batas yang bisa Anda tentukan lewat *scope*.

### Langkah membuat token

1. Login ke Gitea lewat browser dengan akun yang akan dipakai (untuk agent, sebaiknya akun khusus agent).
2. Klik foto profil di pojok kanan atas, pilih **Settings**.
3. Buka tab **Applications**, lalu cari bagian **Manage Access Tokens**.
4. Di **Generate New Token**, isi **Token Name** dengan nama yang menjelaskan fungsinya, misalnya `agent-tea-laptop`.
5. Tentukan **Repository and Organization Access** (lihat bagian berikutnya).
6. Atur level izin untuk tiap kategori (lihat tabel di bawah).
7. Klik **Generate Token**.
8. **Salin nilai token saat itu juga.** Nilainya muncul di bagian atas halaman, berupa string panjang acak, dan **hanya ditampilkan sekali**.

> **Penting:** yang tampil di daftar token setelahnya hanyalah **nama**-nya. Gitea hanya menyimpan hash token, sehingga nilainya tidak bisa dilihat lagi. Menempelkan nama token ke `tea` akan menghasilkan error `user does not exist`. Jika nilainya terlewat, hapus token itu dan buat yang baru.

### Repository and Organization Access

| Pilihan | Artinya |
|---|---|
| **Public only** | Token hanya bisa menjangkau repo publik |
| **All (public, private, and limited)** | Token bisa menjangkau semua repo dan organisasi yang bisa diakses akun pemiliknya |

Pilihan ini hanya dua opsi di sebagian besar versi Gitea, tanpa opsi memilih repo tertentu. Karena itu, pembatasan ke repo tertentu sebaiknya dilakukan lewat **izin akun** (collaborator atau team), bukan lewat token. Lihat [Catatan keamanan](#catatan-keamanan-untuk-ai-agent).

### Kategori izin (scope)

Tiap kategori punya tiga level: **No Access**, **Read**, atau **Read and Write**.

| Kategori | Cakupan | Dipakai `tea` untuk |
|---|---|---|
| `user` | Profil akun | Verifikasi login (`tea login add`, `tea whoami`). **Wajib minimal Read** |
| `issue` | Issue, komentar, label, milestone | Membuat, membaca, dan mengubah issue |
| `repository` | Repo, file, branch, pull request, release | Membaca repo dan PR, mengirim review, merge, release |
| `organization` | Organisasi dan team | Mengakses repo yang berada di bawah organisasi |
| `notification` | Notifikasi | Membaca notifikasi (opsional) |
| `misc` | Endpoint umum seperti info versi server | Sebaiknya Read |
| `package` | Package registry | Umumnya tidak diperlukan |
| `activitypub` | Federasi ActivityPub | Umumnya tidak diperlukan |

> **Catatan:** operasi pull request (baca, review, merge) masuk ke kategori `repository`, bukan `issue`, sepengetahuan saya. Akibatnya, token yang boleh mengirim review juga secara teknis boleh melakukan push dan merge. Batas yang sebenarnya dipaksakan server ada di izin akun, bukan di scope token. Verifikasi di instance Anda.

### Profil token yang disarankan

Pilih sesuai kebutuhan, mulai dari yang paling ketat.

**A. Hanya membaca** (melihat issue dan PR, tanpa mengubah apa pun)

| Kategori | Level |
|---|---|
| user | Read |
| issue | Read |
| repository | Read |
| organization | Read |
| misc | Read |
| lainnya | No Access |

**B. Mengelola issue, review PR dibuat manual** (agent boleh buat dan ubah issue, tidak boleh review)

| Kategori | Level |
|---|---|
| user | Read |
| issue | Read and Write |
| repository | Read |
| organization | Read |
| misc | Read |
| lainnya | No Access |

**C. Mengelola issue dan mereview PR** (buat issue, ubah status, baca PR, approve atau reject)

| Kategori | Level |
|---|---|
| user | Read |
| issue | Read and Write |
| repository | Read and Write |
| organization | Read |
| notification | Read |
| misc | Read |
| package, activitypub | No Access |

Profil C membutuhkan `repository: Read and Write`, sehingga izin akun (team dengan Code = Read) dan branch protection menjadi pembatas utamanya. Untuk akun agent, profil C **hanya aman bila dipasangkan dengan izin akun yang dibatasi**.

### Kelola token yang sudah ada

- **Mencabut token:** di **Settings → Applications → Manage Access Tokens**, klik tombol **Delete** di samping token. Akses langsung hilang.
- **Masa berlaku:** jika form Anda menyediakan kolom kedaluwarsa, isi. Sebagian versi Gitea tidak menampilkannya. Bila tidak ada, ganti token secara berkala.
- **Token lama yang tidak dipakai:** hapus. Token berusia lama dengan akses luas dan tanpa aktivitas hanya menambah risiko.
- **Satu token per fungsi:** jangan memakai satu token untuk banyak alat. Bila satu bocor, cukup cabut yang itu.

### Uji token sebelum dipakai

Cek bahwa token valid dan scope-nya cukup, tanpa `tea`:

```powershell
$env:GITEA_TOKEN = "<nilai-token>"
curl.exe -s -H "Authorization: token $env:GITEA_TOKEN" https://<host-gitea-anda>/api/v1/user
```

| Hasil | Artinya |
|---|---|
| JSON berisi `login`, `email`, dst. | Token valid, scope `user` cukup |
| `token does not have at least one of required scope(s)` | Tambahkan scope `user: Read` |
| `401` atau `user does not exist` | Nilai token salah atau sudah dicabut |

## Login ke server Gitea

Pastikan Anda sudah memiliki **nilai** token dari bagian [Membuat Access Token di Gitea](#membuat-access-token-di-gitea).

### Daftarkan login

```powershell
$env:GITEA_TOKEN = "<nilai-token>"
tea login add --name <nama-login> --url https://<host-gitea-anda> --token $env:GITEA_TOKEN
tea whoami --login <nama-login>
Remove-Item Env:GITEA_TOKEN
```

Jika `tea whoami` menampilkan akun Anda, login berhasil. Baris terakhir menghapus token dari environment sesi.

Setelah itu, bersihkan riwayat PowerShell agar token yang sempat diketik tidak tersimpan:

```powershell
Remove-Item (Get-PSReadLineOption).HistorySavePath
```

## Pemakaian dasar

Jalankan di dalam folder repo yang remote-nya mengarah ke server Gitea. Sertakan `--login` agar tidak muncul prompt interaktif.

```powershell
tea issues list --login <nama-login>
tea issue 42 --login <nama-login>
tea pulls --login <nama-login>
tea --help
```

Sintaks sub-perintah bisa berbeda antarversi. Untuk memastikan, gunakan `tea <perintah> --help`.

## Catatan keamanan untuk AI agent

Bila `tea` dipakai oleh AI agent, ingat bahwa agent bertindak sebagai pemilik token. Beberapa praktik yang disarankan:

- **Gunakan akun terpisah untuk agent**, bukan akun pribadi. Akun pribadi berarti agent memiliki seluruh akses Anda.
- **Batasi akses lewat organisasi.** Buat team khusus di organisasi, pilih hanya repositori yang relevan, dan atur izin per unit (misalnya Issues dan Pull Requests = Write, Code = Read) bila fitur itu tersedia di versi Gitea Anda.
- **Aktifkan branch protection** pada branch utama: larang push langsung, wajibkan approval manusia, dan batasi siapa yang boleh merge.
- **Pakai token dengan scope minimal** dan masa berlaku bila tersedia. Cabut token yang tidak dipakai.
- **Waspadai prompt injection.** Isi issue, komentar PR, dan file repo dibaca agent dan bisa memuat instruksi tersembunyi. Batasi dampaknya dengan izin akun, bukan hanya dengan instruksi tertulis.
- **Minta konfirmasi untuk aksi tulis** (misalnya merge dan penghapusan) di tahap awal pemakaian.
- **Token tersimpan sebagai teks biasa** di file konfigurasi `tea`. Jangan membagikan atau meng-commit file itu.

Uji batas izin sebelum dipakai sungguhan: pastikan `git push` dan merge PR **ditolak** untuk akun agent, dan pastikan approval dari agent tidak memenuhi syarat merge.

## Pemecahan masalah

| Gejala | Penyebab umum | Solusi |
|---|---|---|
| `tea` is not recognized | Terminal masih memakai PATH lama | Buka terminal baru, atau jalankan `$env:Path += ";C:\tools\tea"` |
| `Test-Path` mengembalikan `False` | File belum diganti nama menjadi `tea.exe` atau salah folder | Periksa dengan `Get-ChildItem C:\tools\tea` |
| `user does not exist [uid: 0, name: ]` | Yang ditempel adalah nama token, bukan nilainya, atau scope `user: read` tidak ada | Buat token baru, salin nilainya saat muncul, pastikan `user` Read aktif |
| Prompt `no login matched this repository` | Remote repo tidak cocok dengan URL login (repo GitHub, atau host/port berbeda) | Tambahkan `--login <nama-login>`, atau samakan host remote dengan URL login |
| File `.exe` diblokir Windows | Tanda unduhan dari internet | `Unblock-File C:\tools\tea\tea.exe` |

Untuk memeriksa token tanpa `tea`, lihat [Uji token sebelum dipakai](#uji-token-sebelum-dipakai).

## Metode instalasi lain

| Cara | Catatan |
|---|---|
| `go install gitea.dev/tea@v0.16.0` | Butuh [Go](https://go.dev/dl/). Binary masuk ke `%USERPROFILE%\go\bin`, tambahkan ke PATH. Cocok untuk update berikutnya |
| [MSYS2](https://packages.msys2.org/base/mingw-w64-tea) | Paket pihak ketiga, kurang praktis bila MSYS2 tidak dipakai sehari-hari |
| Kompilasi dari source | Butuh Go 1.26+ dan GNU Make, paling merepotkan di Windows |
| [Docker](https://hub.docker.com/r/gitea/tea) | Kurang cocok untuk agent karena `tea` perlu membaca repo lokal dan konfigurasi |

## Referensi

- Repo dan README resmi: <https://gitea.com/gitea/tea>
- Halaman produk: <https://about.gitea.com/products/tea/>
- Rilis: <https://gitea.com/gitea/tea/releases>
- Unduhan binary: <https://dl.gitea.com/tea/>
