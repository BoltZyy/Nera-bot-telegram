<div align="center">

# 🧠🤖 Dual-Backend Telegram AI Bot
**Satu Bot, Dua Otak: Google Gemini Cloud & KoboldCPP Lokal dalam Satu Genggaman.**

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![Telegram Bot](https://img.shields.io/badge/python--telegram--bot-v21%2B-26A5E4?style=for-the-badge&logo=telegram&logoColor=white)](https://python-telegram-bot.org/)
[![Google GenAI](https://img.shields.io/badge/Google_GenAI-Official_SDK-4285F4?style=for-the-badge&logo=google&logoColor=white)](https://ai.google.dev/)
[![SQLite](https://img.shields.io/badge/SQLite-3_Tables-003B57?style=for-the-badge&logo=sqlite&logoColor=white)](https://www.sqlite.org/)

*Ngobrol santai pakai Gemini, lalu mendadak butuh LLM lokal dari PC rumah via Cloudflare Tunnel? Tinggal ketik `/setproxy` langsung dari chat—tanpa restart server, tanpa edit kode.*

</div>

---

## 💡 Kenapa Bot Ini Dibuat?

Kebanyakan bot AI hanya terkunci pada satu provider. Saat API *limit*, bot mati. Saat ingin pakai model lokal (*self-hosted* di PC rumah), harus bongkar kode lagi.

Bot ini hadir dengan konsep **Dynamic Dual-Backend Router**:
1. **Mode Cloud (Google AI Studio):** Cepat, hemat kuota dengan default `gemini-3.5-flash-lite`, mendukung **analisis gambar (Multimodal)**, dan punya **Multi-API Key Failover** (jika Key #1 kena `429 Rate Limit`, otomatis pindah ke Key #2 tanpa drama).
2. **Mode Lokal (KoboldCPP Proxy):** Cukup jalankan KoboldCPP di PC + Cloudflare Tunnel, tempel URL-nya lewat perintah `/setproxy` di Telegram, dan seluruh percakapan langsung dialihkan secara *real-time* ke mesin Anda sendiri.

---

## 🗺️ Arsitektur Sistem

```mermaid
flowchart TD
    U[👤 User Telegram\nTeks / Foto / Tombol] --> R{🔀 AI Router\nCek SQLite bot_config}
    
    R -- "kobold_proxy_url TERISI" --> K[🔌 KoboldCPP Proxy\nCloudflare Tunnel /v1/chat/completions]
    K --> S[⚡ Streaming ke Telegram\nAnti-FloodWait & Auto-Split]
    
    R -- "kobold_proxy_url KOSONG" --> G[✨ Google AI Studio\nSDK google-genai v1beta]
    G --> F{🔑 Key Pool Failover\nKey 1 -> Key 2 -> Key N}
    F -- "Teks + Multimodal Foto" --> S
    
    S --> DB[(🗄️ SQLite Persistent\nchat_history | bot_config | user_settings)]
```

---

## ✨ Fitur Utama

- **⚡ Real-Time Streaming Response:** Jawaban diketik bertahap langsung di Telegram (mendukung SSE streaming untuk Gemini maupun KoboldCPP) lengkap dengan proteksi *Anti-FloodWait* dan pemecah pesan otomatis jika melebihi 4.096 karakter.
- **🛡️ Multi-API Key Failover:** Pasang banyak API Key Gemini di `.env`. Jika satu key kehabisan kuota (`429` / `RESOURCE_EXHAUSTED`), bot otomatis beralih ke key cadangan berikutnya.
- **📸 Multimodal Vision:** Kirim foto (dengan atau tanpa *caption*) langsung di chat untuk dianalisis secara visual oleh Gemini.
- **🎛️ Kontrol Penuh via Inline Keyboard:** Ganti model AI cukup lewat klik tombol kotak-kotak di `/models`—preferensi model disimpan **per user**.
- **💾 Persisten di 3 Tabel SQLite:**
  - `chat_history` — Memori percakapan terisolasi per `user_id`.
  - `bot_config` — Menyimpan URL `kobold_proxy_url` agar tidak hilang saat container server *restart*.
  - `user_settings` — Menyimpan pilihan model Gemini masing-masing pengguna.
- **🚀 Pterodactyl & Termux Friendly:** Menggunakan *path* relatif terhadap direktori script (`BASE_DIR`) sehingga aman saat di-*deploy* dari Termux $\rightarrow$ GitHub $\rightarrow$ VPS Panel (seperti Wispbyte).

---

## 🤖 Pilihan Model Gemini Dinamis

Setiap pengguna bisa mengganti model favoritnya kapan saja lewat `/models` atau `/setmodel <slug>`:

| Model Slug | Keterangan |
| :--- | :--- |
| `gemini-3.5-flash-lite` | ⭐ **Default** — Paling seimbang, super cepat & hemat token |
| `gemini-flash-lite-latest` | Alias resmi versi *flash-lite* terbaru (`v1beta`) |
| `gemini-flash-latest` | Alias resmi versi *flash* terbaru (`v1beta`) |
| `gemini-3.1-flash-lite` | Ringan dan responsif untuk obrolan harian |
| `gemini-3.8-flash` | Penalaran lebih matang dengan latensi rendah |
| `gemini-4-hydrogen` | Performa tinggi untuk tugas kompleks |

---

## 🎮 Daftar Perintah Telegram

| Perintah | Fungsi |
| :--- | :--- |
| `/start` | Menyapa user, cek status backend aktif, & tampilkan tombol cepat |
| `/models` | Membuka menu tombol interaktif untuk memilih model Gemini |
| `/setmodel <slug>` | Mengganti model aktif milik Anda secara manual |
| `/setproxy <url>` | Mengaktifkan KoboldCPP Proxy (otomatis dirapikan ke `/v1/chat/completions`) |
| `/clearproxy` | Menghapus proxy dari database & kembali ke Google AI Studio |
| `/reset` | Menghapus riwayat memori obrolan Anda di database SQLite |

---

## 🚀 Panduan Instalasi & Deployment

<details>
<summary><b>1️⃣ Struktur Direktori & File Penting (Klik untuk buka)</b></summary>

```text
.
├── bot.py              # Script utama bot Dual-Backend
├── requirements.txt    # Daftar dependensi Python
├── .env                # Token & API Key (JANGAN di-commit ke GitHub!)
├── .gitignore          # Mengabaikan .env dan file database *.db
└── bot_memory.db       # Dibuat otomatis oleh SQLite saat bot pertama kali jalan
```

Contoh isi `.gitignore` agar rahasia & database di VPS tidak tertimpa saat `git pull`:
```gitignore
.env
*.db
*.db-wal
*.db-shm
__pycache__/
venv/
```

</details>

<details>
<summary><b>2️⃣ Konfigurasi <code>.env</code> & <code>requirements.txt</code></b></summary>

**`requirements.txt`**
```txt
python-telegram-bot>=21.0
google-genai>=1.0.0
requests>=2.31.0
python-dotenv>=1.0.0
```

**`.env`**
```env
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
GEMINI_API_KEYS=AIzaSyKeyPertama,AIzaSyKeyCadanganKedua,AIzaSyKeyCadanganKetiga

# Opsional: Isi dengan Telegram User ID Anda (pisahkan koma) agar hanya Admin
# yang bisa menjalankan /setproxy & /clearproxy. Kosongkan jika bebas untuk testing.
ADMIN_USER_IDS=

DB_PATH=bot_memory.db
MAX_HISTORY_MESSAGES=20
PROXY_TIMEOUT_SECONDS=120
STREAM_EDIT_INTERVAL=1.3
SYSTEM_PROMPT=Kamu adalah asisten AI yang cerdas, ramah, akurat, dan menjawab dengan jelas.
```

</details>

<details>
<summary><b>3️⃣ Menjalankan di Lokal / Termux / Pterodactyl Panel (Wispbyte)</b></summary>

1. **Clone repositori & masuk ke folder:**
   ```bash
   git clone https://github.com/username-anda/nama-repo-anda.git
   cd nama-repo-anda
   ```

2. **Install dependensi:**
   ```bash
   pip install -r requirements.txt
   ```

3. **Buat file `.env` dan isi token/key Anda**, lalu jalankan:
   ```bash
   python bot.py
   ```

> **Catatan Wispbyte / Pterodactyl:** Pastikan *Startup Command* atau file utama mengarah ke `bot.py`, dan upload file `.env` langsung melalui **File Manager** panel. Saat update kode, cukup jalankan `git pull` tanpa khawatir `.env` dan `bot_memory.db` hilang!

</details>

<details>
<summary><b>4️⃣ Sinkronisasi Menu Perintah di @BotFather (Opsional)</b></summary>

Bot sudah otomatis mendaftarkan menu saat menyala (`post_init`), namun Anda juga bisa menyetelnya permanen di **@BotFather** lewat perintah `/setcommands`:

```text
start - Menu utama, status backend & tombol cepat
models - Pilih model Gemini lewat tombol interaktif
setmodel - Ganti model Gemini secara manual (/setmodel <slug>)
setproxy - Set URL Cloudflare KoboldCPP (/setproxy <url>)
clearproxy - Hapus proxy & kembali ke Google AI Studio
reset - Hapus riwayat memori chat Anda
```

</details>

---

<div align="center">

Dibuat dengan ☕ dari **Termux** $\rightarrow$ **GitHub** $\rightarrow$ **Pterodactyl VPS**.  
Jika proyek ini membantu, jangan lupa kasih ⭐ di pojok kanan atas!

</div>