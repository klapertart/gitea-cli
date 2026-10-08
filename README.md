# Instalasi `tea` (Gitea CLI) di Windows

`tea` adalah CLI resmi dari Gitea untuk mengelola issue, pull request, release, dan fitur platform lainnya langsung dari terminal. Panduan ini menjelaskan cara memasangnya di Windows, menghubungkannya ke server Gitea, dan memakainya dengan aman, termasuk bila dipakai oleh AI agent.

> **Catatan peran.** `tea` mengurus operasi platform (issue, PR, review, release). Operasi git biasa (`clone`, `commit`, `push`) tetap memakai `git` dengan SSH key atau credential helper. `tea` tidak menggantikan `git`.

## Daftar isi

1. [Prasyarat](#prasyarat)
2. [Instalasi](#instalasi)
3. [Login ke server Gitea](#login-ke-server-gitea)
4. [Pemakaian dasar](#pemakaian-dasar)
5. [Catatan keamanan untuk AI agent](#catatan-keamanan-untuk-ai-agent)
6. [Pemecahan masalah](#pemecahan-masalah)
7. [Metode instalasi lain](#metode-instalasi-lain)
8. [Referensi](#referensi)

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

## Login ke server Gitea

### 1. Buat Access Token

Di browser, login ke Gitea lalu buka **Settings → Applications → Manage Access Tokens** dan buat token baru.

Scope yang disarankan untuk kebutuhan issue dan review PR:

| Scope | Level | Keterangan |
|---|---|---|
| user | Read | Wajib, dipakai `tea` untuk memverifikasi login |
| issue | Read and Write | Membuat dan mengubah issue, komentar, label |
| repository | Read and Write | Dibutuhkan untuk membaca PR dan mengirim review |
| organization | Read | Bila repo berada di bawah organisasi |
| lainnya | No Access | Aktifkan hanya jika benar-benar perlu |

> **Penting:** nilai token hanya ditampilkan **satu kali**, tepat setelah klik *Generate Token*. Yang tampil di daftar token hanyalah **nama**-nya. Menempelkan nama token ke `tea` akan menghasilkan error `user does not exist`. Jika nilainya terlewat, buat token baru.

### 2. Daftarkan login

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

Untuk memeriksa token tanpa `tea`:

```powershell
curl.exe -s -H "Authorization: token $env:GITEA_TOKEN" https://<host-gitea-anda>/api/v1/user
```

Respons JSON berisi `login` berarti token valid.

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
