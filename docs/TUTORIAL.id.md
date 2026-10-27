# Tutorial Lengkap muse-bridge — Bahasa Bayi, Pemula Pasti Bisa

> 🇬🇧 English version: [TUTORIAL.md](TUTORIAL.md)

Tutorial ini ngajarin kamu bikin **Muse (agent AI) jadi model di dalam
9Router**, supaya bisa dipakai Hermes kayak model API beneran (ChatGPT,
DeepSeek, Claude) — lengkap sama tool calls-nya. Ditulis pelan-pelan pakai
bahasa bayi. Kalau kamu bisa copy-paste dan baca pelan-pelan, kamu bisa.

---

## Bagian 0 — Kita mau bikin apa sih?

Bayangin restoran:

| Benda di tutorial ini | Di restoran dia jadi |
|---|---|
| **Hermes** | Pelayan. Dia yang ngomong sama kamu (di Telegram), dia yang punya tangan buat kerja |
| **9Router** | Buku pesanan pintar. Semua pesanan model lewat dia |
| **Bridge** | Buku catatan kecil di dapur VPS. Pesanan ditulis jadi file di sini |
| **Kurir (hook)** | Anak yang tiap 5 detik ngecek buku catatan, terus bangunin koki kalau ada pesanan |
| **Koki / pekerja (worker)** | Muse si agent. Dia yang mikir dan nulis jawaban |
| **Tool calls** | Surat balik dari koki isinya perintah: "pelayan, tolong jalanin ini" |
| **VPS** | Rumahnya pelayan. Dapur dan semua alat ada di sini |

Masalah awalnya: koki tinggal di rumah lain dan **nggak punya telepon**
(nggak ada API). Jadi kita akalin pakai buku catatan + kurir. Pesanan ditulis
jadi file, kurir yang bolak-balik nganterin. Hasilnya memang lebih lambat dari
API beneran, tapi jalan — dan koki-nya beneran pinter, bukan model kaleng-kaleng.

Hasil akhir yang mau kita capai:

- Di 9Router kamu ada model bernama **`ms/muse`**
- Hermes bisa milih model itu kayak milih model lain
- Model itu bisa **ngobrol**, bisa **minta tool dijalankan** (dan Hermes yang
  jalanin beneran di VPS), dan bisa **lihat gambar** yang kamu kirim

## Bagian 1 — Bahan-bahan (harus udah punya)

Cek satu-satu, jangan loncat:

1. **Satu VPS** (Linux, kamu pegang root). Ini rumahnya pelayan
2. **9Router** sudah terpasang dan jalan di VPS itu (biasanya di port `20128`)
3. **Hermes** sudah terpasang di VPS yang sama dan sudah nyambung ke 9Router
   itu (pakai model apa pun dulu, yang penting jalan)
4. **Satu agent Muse** di tempat lain (di tutorial ini: Muse di VM
   sandbox) yang punya fitur **hook** — script kecil yang bisa jalan tiap
   beberapa detik dan bisa membangunkan agent
5. **SSH dari tempat agent ke VPS** harus bisa (pakai kunci/key, bukan
   password)

> Belum punya akun Muse untuk nomor 4? Ikuti dulu
> **[Cara Klaim Muse — Pakai VPN & Tanpa VPN](CLAIM-MUSE.id.md)** dan
> tukarkan kode referral **`MZMT4Y`** dalam 48 jam setelah bergabung.

Kalau nomor 2 dan 3 belum beres, beresin itu dulu. Tutorial ini mulai dari
titik "9Router dan Hermes sudah hidup".

Istilah yang bakal sering muncul:

- **Queue / antrean**: folder tempat file pesanan nunggu
- **Pending**: pesanan yang belum dijawab
- **Done**: jawaban yang sudah jadi
- **Provider**: "merek model" di dalam 9Router. Punya kita mereknya `ms`,
  modelnya `muse`, jadi nama lengkapnya `ms/muse`

## Bagian 2 — Pasang bridge di VPS

Bridge itu program kecil (satu file Python, tanpa library aneh-aneh) yang:

1. Nerima request dari 9Router di `http://127.0.0.1:8765/v1/chat/completions`
2. Nyimpen request itu jadi file di folder `queue/pending/`
3. Nunggu sampai ada file jawaban di `queue/done/`
4. Ngembaliin jawaban itu ke 9Router kayak jawaban model biasa

> **Cara cepat:** langkah 2.1–2.4 di bawah ini sudah diotomatiskan oleh
> script installer — `scripts/install-linux.sh` (Linux),
> `scripts/install-macos.sh` (macOS), `scripts/install-windows.ps1`
> (Windows). Script-nya yang bikinin folder, masang bridge, daftarin jadi
> service yang nyala terus, dan ngecek kesehatannya. Langkah manual di
> bawah ini isinya sama persis, ditulis lengkap biar kamu tahu script-nya
> ngapain aja.

### Langkah 2.1 — Bikin foldernya

Di VPS, jalanin:

```bash
mkdir -p /opt/muse-bridge/queue/pending /opt/muse-bridge/queue/done
```

### Langkah 2.2 — Taruh file bridge

Copy file `bridge/bridge.py` dari repo ini ke:

```
/opt/muse-bridge/bridge.py
```

### Langkah 2.3 — Bikin service-nya biar nyala terus

Bikin file `/etc/systemd/system/muse-bridge.service`:

```ini
[Unit]
Description=Muse bridge for 9Router
After=network.target

[Service]
ExecStart=/usr/bin/python3 /opt/muse-bridge/bridge.py
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

Terus nyalain:

```bash
systemctl daemon-reload
systemctl enable --now muse-bridge
```

### Langkah 2.4 — Cek dia hidup

```bash
curl http://127.0.0.1:8765/health
curl http://127.0.0.1:8765/v1/models
```

Kalau dua-duanya jawab normal, bridge sehat. **Bridge ini cuma dengerin di
localhost (127.0.0.1)** — nggak dibuka ke internet, dan itu sengaja. Yang
boleh ngomong sama dia cuma 9Router di mesin yang sama.

## Bagian 3 — Kunci SSH dari rumah koki ke VPS

Kurir dan koki tinggal di mesin agent. Mereka ngambil dan ngantar file
antrean **lewat SSH**. Jadi mesin agent harus bisa masuk ke VPS pakai kunci.

### Langkah 3.1 — Bikin kunci di mesin agent (kalau belum punya)

```bash
ssh-keygen -t ed25519
cat ~/.ssh/id_ed25519.pub
```

Baris panjang yang keluar itu **kunci publik**. Itu yang boleh dibagi.

### Langkah 3.2 — Tempel kunci publiknya di VPS

Di VPS, tambahkan baris kunci publik tadi ke:

```
~/.ssh/authorized_keys
```

### Langkah 3.3 — Bikin wrapper biar nggak ngetik panjang terus

Di mesin agent, simpan dua script dari folder `tools/` repo ini:

- `tools/ssh-vps.sh` — buat jalanin perintah di VPS:
  `ssh-vps.sh "ls /opt/muse-bridge/queue/pending"`
- `tools/scp-vps.sh` — buat copy file dari/ke VPS

Di dalam dua file itu, ganti alamat VPS-nya dengan alamat VPS kamu
(di file contoh tertulis `YOUR_VPS_IP`).

### Langkah 3.4 — Tes

Dari mesin agent:

```bash
./ssh-vps.sh "echo halo dari vps"
```

Kalau tulisan `halo dari vps` muncul, jalurnya beres.

> **Catatan jujur:** di setup aslinya, SSH ini jalannya muter lewat proxy
> khusus karena mesin agent-nya ada di dalam sandbox. Di mesin normal, SSH
> langsung juga cukup. Sesuaikan sama tempat agent kamu tinggal.

## Bagian 4 — Daftarin provider di 9Router

Sekarang kasih tahu 9Router: "ada model baru namanya muse, tanya ke bridge".

> ⚠️ **Pelajaran mahal, baca ini:** daftarin provider-nya **lewat dashboard
> 9Router**, JANGAN main suntik database langsung. Kami sudah pernah coba
> suntik database dan hasilnya setengah rusak: daftar modelnya kelihatan,
> tapi request-nya nggak jalan. Dashboard yang nulis format yang benar.
> Database cuma aman buat bikin *combo* (lihat langkah 4.3).

### Langkah 4.1 — Bikin node provider lewat dashboard

Di dashboard 9Router (dari localhost VPS kamu), tambah provider
**OpenAI-compatible** dengan isi:

| Kolom | Isi |
|---|---|
| Name | `muse` |
| Prefix | `ms` |
| Base URL | `http://127.0.0.1:8765/v1` |

### Langkah 4.2 — Bikin connection

Di provider itu, tambah connection:

| Kolom | Isi |
|---|---|
| Name | `test` (bebas) |
| Auth type | API key |
| API key | Isi apa saja (bridge lokal ini tidak ngecek key-nya — yang penting kolomnya ada dan aktif) |

### Langkah 4.3 — Bikin combo

Bikin combo bernama **`muse`** isinya satu model:

```
["ms/muse"]
```

Combo ini yang bikin kamu bisa manggil modelnya cukup dengan nama `muse`.

### Langkah 4.4 — Bikin API key 9Router buat klien

Di dashboard 9Router, bikin satu API key. Key ini yang nanti dipakai Hermes
(atau klien lain) buat ngomong ke 9Router. Simpan key ini di tempat aman
berizin file `0600`, **jangan** ditulis di repo atau di chat.

## Bagian 5 — Pasang kurir + buku aturan koki (di mesin agent)

Ini bagian paling penting. Dua benda yang dipasang di mesin tempat agent
tinggal:

### Benda 1 — Script kurir (hook)

Tugasnya cuma tiga, dan dia **bodoh** (nggak mikir sama sekali):

1. Tiap **5 detik**, dia lihat folder `queue/pending/` di VPS lewat SSH
2. Kalau kosong → diam. Selesai. **Nggak ada agent yang dibangunin, nggak
   ada token yang kebakar.** Pengecekan doang itu gratis
3. Kalau ada file baru → dia cap file itu sebagai "lagi diurus" (biar nggak
   dibangunin dua kali) terus **bangunin satu agent pekerja**

Isi script-nya kurang lebih: satu perintah SSH buat listing folder, hitung
file yang belum dicap, putuskan "diam" atau "bangunkan pekerja".

### Benda 2 — Buku aturan pekerja (worker prompt)

Pas pekerja bangun, dia dikasih buku aturan. Buku aturan yang dipakai
sekarang ada di `worker/worker-prompt-v4.txt`. Isinya, diterjemahkan ke
bahasa manusia:

**Urutan mikir pekerja (wajib urut):**

1. **Baca file pesanannya.** Isinya request dari Hermes: riwayat chat,
   daftar tool yang Hermes punya, dan lain-lain
2. **Ada gambar/video?** Ekstrak dan **lihat beneran** dulu sebelum jawab.
   Nggak ada kiriman yang boleh dibuang diam-diam
3. **Pesanan basi?** Kalau pesanan umurnya sudah lebih dari 220 detik,
   buang, jangan dijawab. Yang minta sudah nggak nunggu lagi
4. **Sudah ada hasil tool di dalam pesanan?** (artinya ini putaran kedua)
   → tulis jawaban final **dari hasil asli itu**. Nggak boleh ngarang
5. **Belum ada hasil tool, dan tugasnya butuh mesin beneran?** (cek disk,
   baca file, lihat service) → **jangan jawab pakai teks**. Tulis surat
   tool calls: daftar perintah yang harus Hermes jalankan
6. **Selain itu** → jawab teks biasa kayak model normal

**Aturan soal surat tool calls:**

- Cuma boleh pakai nama tool yang Hermes tawarin di pesanannya
- **Jumlahnya tanpa batas** — tapi cuma gabungin perintah yang **saling
  bebas** (nggak tunggu-tungguan hasil). Yang berurutan tetap satu-satu
  per putaran, karena perintah kedua butuh hasil perintah pertama
- Kalau info yang ada sudah cukup buat jawab, langsung jawab. Berhenti
- Perintah yang **mengubah-ubah** (hapus, timpa, restart, matiin, install)
  cuma boleh ditulis kalau tugasnya memang minta itu persis. Nggak boleh
  nambah-nambahin sendiri

**Aturan jujur (ini yang paling mahal pelajarannya):**

- Nggak pernah boleh ngarang angka, nama file, ukuran, atau status service
- Klaim "sudah saya cek" cuma boleh kalau beneran ada hasil aslinya
- Kalau ditanya "kamu akses VPS lewat apa?", jawab yang benar: pesanan
  datang sebagai file antrean lewat SSH; tool calls adalah jalur resmi
  lewat Hermes; SSH itu jalur sampingan punya pekerja sendiri

Kenapa aturan jujur ini ada? Karena pernah kejadian pekerja **berubah-ubah
sikap** dalam satu percakapan: pertama menolak, terus mengaku sudah ngecek
dengan hasil sebagian karangan, terus membantah semuanya. Akar masalahnya
dua: tiap pesanan dilayani pekerja **baru** yang cuma lihat teks (dia nggak
ingat pekerja sebelumnya ngapain), dan aturan lamanya ambigu. Aturan baru
ini yang ngobatin dua-duanya.

### Cara pekerja ngirim jawaban

- Jawaban teks → dia tulis file `done/<id>.json` isinya `{"content": "..."}`
- Surat tool calls → file yang sama, isinya `{"tool_calls": [...]}`
- ID-nya **selalu sama** dengan ID pesanannya. Ini yang bikin jawaban nggak
  bisa ketuker antar sesi: setiap pesanan dan jawaban ditempel label yang
  sama, dan bridge cuma nyocokin label

## Bagian 6 — Sambungkan Hermes

Terakhir, suruh Hermes pakai model barunya.

Di config profil Hermes kamu, arahkan modelnya ke 9Router lokal dengan nama
model `muse` (combo yang tadi dibuat), pakai API key 9Router dari langkah
4.4. Sisanya (base URL 9Router, dll.) sama kayak profil itu memakai model
9Router lainnya.

### Soal pengaman persetujuan (PENTING, jangan diskip)

Hermes punya pengaman: sebelum jalanin perintah berbahaya (hapus file,
restart service, timpa file sistem, dan teman-temannya), dia **nanya kamu
dulu** di chat. Di setup kami, isinya terverifikasi begini:

| Pengaturan | Isi |
|---|---|
| `approvals.mode` | `manual` — wajib nanya dulu |
| Timeout nanya | 60 detik — kamu nggak jawab = nggak jalan |
| Perintah dari cron/terjadwal | `deny` — ditolak, nggak bisa minta izin |
| Subagent menyetujui sendiri | Mati |

Pengaman inilah alasan tool calls di sistem ini berani dibuat **tanpa
batas jumlah**: pekerja boleh ngusulin perintah sebanyak apa pun, tapi yang
**mutusin jalan atau nggak** tetap kamu, di pintu Hermes.

Ada satu mode yang membatalkan semua itu: **`/yolo`**. Kalau kamu nyalain
yolo, Hermes jalanin semuanya tanpa nanya. Pakai yolo cuma kalau kamu sadar
penuh dan lagi ngawasin. **Jangan tinggalin yolo nyala.**

## Bagian 7 — Tes satu-satu (jangan diskip!)

Jalanin urut. Setiap tes ada "tanda lulus"nya.

### Tes 1 — Bridge hidup

```bash
curl http://127.0.0.1:8765/health
```

✅ Lulus: ada jawaban status OK dari bridge.

### Tes 2 — Model kelihatan dari 9Router

Dari VPS, panggil daftar model 9Router pakai API key kamu. 

✅ Lulus: `muse` (dan `ms/muse`) ada di daftar.

### Tes 3 — Chat biasa

Kirim satu chat completion model `muse`, pertanyaan gampang ("2+2 berapa?").

✅ Lulus: dijawab beneran (jawabannya "4"). Tungguin — jawaban pertama bisa
20–30 detik. Itu normal di sistem ini.

### Tes 4 — Tool calls (tes paling penting)

Dari Hermes (Telegram), pakai model `ms/muse`, bilang:

> "Cek disk VPS sekarang pakai tool"

✅ Lulus kalau kejadiannya begini:

1. Hermes **nampilin langkah tool** (misalnya "💻 terminal `df -h`") — itu
   artinya perintahnya Hermes yang jalanin di VPS, bukan karangan
2. Jawaban akhirnya nyebut angka yang **sama persis** dengan kalau kamu
   jalanin `df -h` sendiri

Kalau Hermes cuma ngasih teks perintah buat kamu jalanin manual, berarti
tool calls-nya belum nyambung — cek lagi Bagian 4 dan 5.

### Tes 5 — Banyak cek sekaligus

Bilang ke Hermes:

> "Cek disk, RAM, uptime, dan status service sekaligus pakai tool"

✅ Lulus: dalam **satu giliran**, Hermes jalanin beberapa tool (bukan
dicicil satu-satu per jawaban).

### Tes 6 — Gambar

Kirim gambar apa saja ke Hermes (model `ms/muse`) dan tanya isinya.

✅ Lulus: jawabannya cocok sama gambarnya — berarti gambarnya beneran
dilihat pekerja, bukan ditebak.

### Tes 7 — Dua sesi barengan (anti ketuker)

Buka dua topik/sesi berbeda, suruh dua-duanya kerja di waktu yang hampir
samaan.

✅ Lulus: jawaban masing-masing nyambung sama topiknya sendiri. Sistem
label ID bikin mereka secara desain nggak bisa ketuker — tes ini buat
bukti, bukan buat nebak.

## Bagian 8 — Cara pakai sehari-hari

- **Cocoknya** buat: kerjaan yang penting hasilnya dari data asli (cek-cek
  VPS, baca log, rangkum, mikir berat). Lama dikit nggak masalah
- **Kurang cocok** buat: ngobrol cepat bolak-balik atau tugas yang butuh
  belasan langkah berurutan — tiap langkah nambah ±30 detik
- **Aturan sabar:** di bawah **2 menit** itu normal, bukan macet. Macet itu
  kalau sudah lewat **4 menit** nggak ada jawaban
- **Ingat biayanya:** yang bayar token itu **agent pekerjanya**, dan dia
  bayar tiap ada pesanan beneran (tiap putaran dia baca ulang seluruh
  riwayat sesi, 100–220 KB). Pengecekan antrean yang kosong itu gratis.
  Jadi pakai model ini sebagai "spesialis", bukan buat semua chat receh

## Bagian 9 — Kalau ada masalah

| Gejala | Sebab paling mungkin | Obatnya |
|---|---|---|
| Error `Muse bridge busy` (429) | Antrean penuh: sudah ada 20 pesanan nunggu | Tunggu yang jalan selesai, kirim ulang. Ini batas antrean, bukan rusak |
| Timeout / 504 setelah ±4 menit | Pekerja telat jawab — mesin agent mati, atau antrean kepanjangan | Cek mesin agent hidup nggak; cek antrean di VPS (`ls /opt/muse-bridge/queue/pending`) |
| Hermes cuma jawab teks, nggak pernah pakai tool | Provider/combo salah, atau prompt pekerja versi lama | Cek Bagian 4 (combo `muse` → `["ms/muse"]`) dan prompt pekerja minimal v2 |
| Jawaban ngarang / ngaku ngecek padahal nggak | Aturan jujur di prompt pekerja hilang/keubah | Pasang ulang prompt dari `worker/worker-prompt-v4.txt` |
| Semua mati setelah mesin agent restart | Mesin agent itu gampang kereset? Stack di sisi agent (termasuk hook) harus dipulihkan | Bikin script rebuild + watchdog yang ngecek kesehatan stack dan rebuild otomatis kalau mati |
| Gambar dikirim tapi dijawab ngasal | Prompt pekerja belum ada aturan media | Pakai prompt v4 (ada aturan media di dalamnya) |
| SSH ke VPS putus | Kunci/jaringan | Kurir punya alarm: setelah gagal berkali-kali dia ngabarin pemilik. Cek `ssh-vps.sh "echo ok"` manual |

## Bagian 10 — Keamanan & kata jujur terakhir

**Keamanan:**

- Rahasia (API key, token) **nggak pernah** masuk repo ini. Simpan di file
  terpisah berizin `0600` di mesin masing-masing
- Bridge cuma dengar di localhost VPS. Jangan pernah buka dia ke internet
- Backup sebelum ngoprek: file bridge lama disimpan sebagai
  `bridge.py.bak-<tanggal>`, prompt pekerja lama disimpan sebagai file
  cadangan. Rollback = balikin file cadangan + restart service. Satu menit
- Persetujuan manual Hermes (Bagian 6) adalah kunci pengaman utama.
  Perlakukan `/yolo` kayak pisau tajam: boleh dipakai, jangan ditinggal
  menyala

**Batas-batas yang jujur (biar nggak kaget):**

- Sistem ini **nggak akan pernah secepat API beneran** (2–5 detik). Yang
  jawab itu agent yang harus bangun dan baca konteks dulu. Target
  realistisnya "90% rasa API": protokol lengkap, stabil, cukup cepat
- Pesanan diproses **paralel**: koordinator membagi satu pekerja per
  pesanan (maks 8 barengan, maks 20 pesanan menunggu). Ledakan request
  tetap antre di belakang waktu mikir per putaran
- Video: yang bisa dilakukan pekerja adalah **mengintip beberapa frame**,
  bukan menonton video penuh. Kalau ada yang ngaku bisa nonton full, itu
  ngibul
- Setiap putaran membaca ulang seluruh riwayat sesi. Sesi yang sudah sangat
  panjang akan makin lambat dan makin boros token. Sesekali mulai sesi baru
  itu sehat

---

*Tutorial ini ditulis dari sistem yang beneran jalan dan dites, bukan dari
teori. Angka-angka delay dan ukuran di atas adalah angka asli dari log
sistemnya. Selamat mencoba — kalau mentok, baca lagi Bagian 9 pelan-pelan.*
