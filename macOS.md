# Panduan Claude Code + Claude Desktop dengan DeepSeek (macOS)

Panduan memasang dan mengonfigurasi **Claude Code CLI** serta **Claude Desktop mode 3P (gateway)** dengan **DeepSeek** di macOS.

Panduan Windows tersedia di [Readme.MD](Readme.MD).

> Tutorial ini mengikuti perilaku Claude Desktop 3P dan API DeepSeek yang kompatibel dengan Anthropic Messages API. Nama menu dapat berubah pada versi aplikasi yang lebih baru.

---

## Prasyarat

- macOS 14 Sonoma atau lebih baru (khusus Cowork)
- Apple silicon atau Intel (x64)
- Terminal (zsh)
- Homebrew (opsional, tetapi memudahkan instalasi)
- Node.js LTS dan Git
- API Key DeepSeek dari <https://platform.deepseek.com>
- Untuk Cowork:
  - Claude Desktop versi `.dmg`
  - Hardware virtualization aktif
  - Ruang disk cukup (bundle workspace > 1 GB; sisakan sekitar 20–30 GB)
  - Akses ke `downloads.claude.ai` dan `api.deepseek.com`

---

# Bagian 1 — Claude Code CLI

## 1. Install Xcode Command Line Tools

```bash
xcode-select --install
```

## 2. Install Homebrew (jika belum ada)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Ikuti instruksi di akhir instalasi untuk menambahkan Homebrew ke `PATH` (biasanya `/opt/homebrew/bin` di Apple silicon atau `/usr/local/bin` di Intel).

## 3. Install Node.js dan Git

```bash
brew install node git
```

Verifikasi:

```bash
node -v
npm -v
npx -v
git --version
```

## 4. Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

Jika muncul error `EACCES`, jangan langsung memakai `sudo`. Gunakan installer resmi:

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

atau atur prefix npm lokal:

```bash
npm config set prefix ~/.npm-global
export PATH="$HOME/.npm-global/bin:$PATH"
npm install -g @anthropic-ai/claude-code
```

Tambahkan baris `export PATH="$HOME/.npm-global/bin:$PATH"` ke `~/.zshrc` bila memakai prefix lokal.

## 5. Environment variable DeepSeek

Tambahkan ke `~/.zshrc`. Ganti `<API_KEY_DEEPSEEK>` dengan API key Anda.

```bash
export ANTHROPIC_API_KEY="<API_KEY_DEEPSEEK>"
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"

export ANTHROPIC_MODEL="deepseek-v4-flash"
export ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-pro[1m]"
export ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-flash-vision-exp"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-flash"
export CLAUDE_CODE_SUBAGENT_MODEL="deepseek-v4-flash"

export CLAUDE_EFFORT="high"
export CLAUDE_CODE_EFFORT_LEVEL="high"
export CLAUDE_CODE_AUTO_COMPACT_WINDOW="786432"
```

Opsional, untuk mengurangi telemetri/atribusi:

```bash
export CLAUDE_CODE_ATTRIBUTION_HEADER="0"
export CLAUDE_CODE_ENABLE_TELEMETRY="0"
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC="1"
```

Muat ulang shell:

```bash
source ~/.zshrc
```

> `~/.zshrc` berisi API key plaintext. Batasi permission dengan `chmod 600 ~/.zshrc` dan jangan commit file ini ke Git.

## 6. Konfigurasi CLI

Buat atau edit `~/.claude/settings.json`:

```json
{
  "permissions": {
    "defaultMode": "bypassPermissions"
  },
  "model": "haiku",
  "effortLevel": "high",
  "modelSettings": {
    "deepseek-v4-pro": {
      "effortLevel": "xhigh"
    },
    "deepseek-v4-flash": {
      "effortLevel": "xhigh"
    }
  },
  "enabledPlugins": {
    "frontend-design@claude-plugins-official": true,
    "context7@claude-plugins-official": true,
    "superpowers@claude-plugins-official": true
  },
  "skipDangerousModePermissionPrompt": true
}
```

Catatan:

- `model: "haiku"` adalah alias yang diarahkan ke `deepseek-v4-flash` melalui `ANTHROPIC_DEFAULT_HAIKU_MODEL`.
- `bypassPermissions` menjalankan perintah tanpa konfirmasi. Gunakan hanya di environment yang Anda percaya.

## 7. Verifikasi CLI

```bash
claude --version
claude
```

Contoh prompt:

```text
Balas dengan satu kata: OK
```

---

# Bagian 2 — Claude Desktop Mode 3P (Gateway)

## 1. Install Claude Desktop

Unduh dari <https://claude.ai/download> lalu pasang `.dmg`.

Jika macOS memblokir aplikasi:

1. Buka **System Settings → Privacy & Security**.
2. Klik **Open Anyway** untuk Claude.
3. Konfirmasi.

## 2. Aktifkan Developer Mode dan Third-Party Inference

1. Buka Claude Desktop.
2. Masuk ke **Settings → Developer**.
3. Aktifkan **Developer mode**.
4. Buka **Configure Third-Party Inference…**.

## 3. Hubungkan ke DeepSeek

Isi form koneksi:

| Field | Nilai |
|---|---|
| Connection type | Gateway |
| Base URL | `https://api.deepseek.com/anthropic` |
| API key | API key DeepSeek Anda |
| Auth scheme | `x-api-key` |
| Credential kind | Static |
| Model discovery | Nonaktif |

Penting:

- Base URL **tidak boleh** berakhiran `/v1`. Claude Desktop menambahkan sendiri `/v1/messages` dan `/v1/models`.
- Gunakan **API key statis**. Opsi **Sign in with gateway** memakai OIDC/SSO dan tidak didukung DeepSeek.
- Jangan mengaktifkan model discovery. DeepSeek tidak menyediakan `/v1/models`; Claude Desktop akan menampilkan error 404.

## 4. Daftar Model

Claude Desktop memfilter model yang bukan keluarga Anthropic. Model seperti `deepseek-v4-flash` akan dihapus dari daftar. Gunakan alias Claude yang dipetakan DeepSeek:

| Model ID di Claude Desktop | Dipetakan DeepSeek | Label |
|---|---|---|
| `claude-haiku-4-5` | `deepseek-flash` | DeepSeek Flash (Haiku) |
| `claude-sonnet-4-5` | `deepseek-flash` | DeepSeek Flash (Sonnet) |
| `claude-opus-4-5` | `deepseek-v4-pro` | DeepSeek V4 Pro (Opus) |
| `claude-opus-4-5[1m]` | `deepseek-v4-pro` | DeepSeek V4 Pro (Opus, 1M) |

Isi daftar model:

```json
[
  {
    "name": "claude-haiku-4-5",
    "labelOverride": "DeepSeek Flash (Haiku)",
    "anthropicFamilyTier": "haiku"
  },
  {
    "name": "claude-sonnet-4-5",
    "labelOverride": "DeepSeek Flash (Sonnet)",
    "anthropicFamilyTier": "sonnet"
  },
  {
    "name": "claude-opus-4-5",
    "labelOverride": "DeepSeek V4 Pro (Opus)",
    "anthropicFamilyTier": "opus"
  },
  {
    "name": "claude-opus-4-5[1m]",
    "labelOverride": "DeepSeek V4 Pro (Opus, 1M)"
  }
]
```

Set effort default:

```text
high
```

## 5. Terapkan dan Restart

1. Klik **Apply**.
2. Keluar sepenuhnya dari Claude Desktop (`⌘Q`).
3. Buka kembali Claude Desktop.
4. Buka pemilih model. Anda harus melihat empat entri DeepSeek di atas.

## 6. Verifikasi

Log macOS:

```bash
tail -f ~/Library/Logs/Claude-3p/main.log
```

Baris yang menandakan konfigurasi berhasil:

```text
[custom-3p] 3P mode active { provider: 'gateway' }
[custom-3p] Model discovery: not supported by provider; picker = 4 (inferenceModels)
[custom-3p] inference apiHost=https://api.deepseek.com/anthropic
[custom-3p] ConfigHealth recomputed { state: 'healthy', provider: 'gateway' }
```

Jika `picker = 0 (empty)`, daftar model tidak terbaca. Periksa kembali alias model dan JSON.

## 7. Konfigurasi Lokal (Advanced)

Claude Desktop 3P menyimpan konfigurasi lokal di:

```text
~/Library/Application Support/Claude-3p/configLibrary/
```

Struktur:

```text
configLibrary/
├── _meta.json
└── <uuid>.json
```

`_meta.json`:

```json
{
  "appliedId": "<uuid>",
  "entries": [
    {
      "id": "<uuid>",
      "name": "DeepSeek Gateway"
    }
  ]
}
```

`<uuid>.json`:

```json
{
  "inferenceProvider": "gateway",
  "inferenceGatewayBaseUrl": "https://api.deepseek.com/anthropic",
  "inferenceGatewayApiKey": "<API_KEY_DEEPSEEK>",
  "inferenceGatewayAuthScheme": "x-api-key",
  "inferenceCredentialKind": "static",
  "modelDiscoveryEnabled": false,
  "inferenceModels": [
    {
      "name": "claude-haiku-4-5",
      "labelOverride": "DeepSeek Flash (Haiku)",
      "anthropicFamilyTier": "haiku"
    },
    {
      "name": "claude-sonnet-4-5",
      "labelOverride": "DeepSeek Flash (Sonnet)",
      "anthropicFamilyTier": "sonnet"
    },
    {
      "name": "claude-opus-4-5",
      "labelOverride": "DeepSeek V4 Pro (Opus)",
      "anthropicFamilyTier": "opus"
    },
    {
      "name": "claude-opus-4-5[1m]",
      "labelOverride": "DeepSeek V4 Pro (Opus, 1M)"
    }
  ],
  "defaultModelEffort": "high"
}
```

Pastikan `appliedId` sama dengan nama file `<uuid>.json`.

Mode deployment disimpan di:

```text
~/Library/Application Support/Claude-3p/claude_desktop_config.json
```

Nilai yang diperlukan:

```json
{
  "deploymentMode": "3p"
}
```

## 8. Managed Profile (Opsional, Administrator)

Untuk deployment terkelola, macOS membaca managed preferences:

```text
/Library/Managed Preferences/<user>/com.anthropic.claudefordesktop.plist
/Library/Managed Preferences/com.anthropic.claudefordesktop.plist
```

Gunakan hanya jika Anda mengelola banyak mesin. Untuk penggunaan pribadi, konfigurasi lokal di atas lebih sederhana.

---

# Bagian 3 — Cowork

Cowork adalah fitur di dalam Claude Desktop, bukan aplikasi terpisah. Cowork memakai model yang sama dengan Chat melalui konfigurasi 3P.

## Prasyarat Cowork di macOS

- macOS 14 Sonoma atau lebih baru
- Apple silicon atau Intel (x64)
- Claude Desktop resmi (`.dmg`)
- Hardware virtualization tersedia
- Ruang disk cukup. Bundle workspace pertama dapat melebihi 1 GB; sisakan sekitar 20–30 GB
- Koneksi ke `downloads.claude.ai` dan `api.deepseek.com`

## Pemeriksaan Awal

Versi macOS:

```bash
sw_vers
```

Ruang disk:

```bash
df -h /
```

Ukuran data Claude (hanya untuk melihat, jangan langsung dihapus):

```bash
du -sh ~/Library/Application\ Support/Claude 2>/dev/null
du -sh ~/Library/Application\ Support/Claude-3p 2>/dev/null
du -sh ~/Library/Caches/Claude 2>/dev/null
```

> Jangan menghapus `~/Library/Application Support/Claude-3p/configLibrary/`; folder itu berisi konfigurasi gateway Anda. Jika perlu membersihkan, hapus hanya cache yang aman dan pastikan aplikasi sudah ditutup.

## Menjalankan Cowork

1. Buka Claude Desktop dalam mode 3P.
2. Pilih tab **Cowork**.
3. Ikuti wizard persiapan.
4. Pada pemilihan model, pilih salah satu alias DeepSeek.
5. Pada run pertama, aplikasi mengunduh bundle workspace. Proses ini bisa lama.

## Verifikasi Cowork

Pantau log:

```bash
tail -f ~/Library/Logs/Claude-3p/main.log
```

Cari baris yang menyebut Cowork, workspace, VM, atau service. Jika tab Cowork tidak muncul:

- Pastikan macOS 14+.
- Pastikan aplikasi dijalankan dari `.dmg` resmi, bukan salinan yang dimodifikasi.
- Periksa koneksi ke `downloads.claude.ai`.
- Periksa ruang disk.
- Restart aplikasi dan coba lagi.

Jika Cowork berhasil berjalan tetapi model gagal:

- Periksa `ConfigHealth` di log.
- Pastikan API key masih valid.
- Pastikan model yang dipilih adalah alias Claude, bukan `deepseek-*`.

---

# Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| `zsh: command not found: claude` | `PATH` npm belum dimuat | Tambahkan `export PATH="$HOME/.npm-global/bin:$PATH"` ke `~/.zshrc`, lalu `source ~/.zshrc` |
| CLI 401 | API key salah/kedaluwarsa | Perbarui `ANTHROPIC_API_KEY` |
| CLI 404 | Base URL salah | Gunakan `https://api.deepseek.com/anthropic` tanpa `/v1` |
| Desktop menampilkan login Claude.ai | Mode 3P belum aktif | Pastikan `deploymentMode: "3p"` dan konfigurasi gateway terbaca |
| Model picker kosong | Model `deepseek-*` difilter | Gunakan alias `claude-haiku-4-5`, `claude-sonnet-4-5`, `claude-opus-4-5` |
| `ConfigHealth: invalid` | API key/base URL tidak valid | Periksa ulang form koneksi dan log |
| Error 404 pada `/v1/models` | Model discovery aktif | Nonaktifkan model discovery |
| Cowork tidak muncul | macOS terlalu lama / bundle belum terunduh / disk kurang | Perbarui macOS, cek disk, cek koneksi, restart |
| Cowork jalan tetapi model error | Model bukan alias Claude | Pilih model DeepSeek melalui alias Claude |
| Perubahan tidak terbaca | Aplikasi masih berjalan | Keluar dengan `⌘Q`, lalu buka kembali |

## Perbedaan CLI dan Desktop

CLI membaca environment variable dari shell. Claude Desktop **tidak** otomatis membaca `~/.zshrc`. Konfigurasi Desktop harus diisi melalui form 3P atau config library. Jika CLI berhasil tetapi Desktop gagal, fokus pada konfigurasi Desktop, bukan environment shell.

---

# Rollback

## Kembali ke mode 1P

1. Keluar dari Claude Desktop (`⌘Q`).
2. Edit:

```text
~/Library/Application Support/Claude-3p/claude_desktop_config.json
```

3. Ubah:

```json
{
  "deploymentMode": "1p"
}
```

4. Buka kembali Claude Desktop.

## Hapus konfigurasi gateway

Cadangkan dulu:

```bash
mv ~/Library/Application\ Support/Claude-3p/configLibrary \
   ~/Library/Application\ Support/Claude-3p/configLibrary.bak
```

Untuk mengembalikan:

```bash
mv ~/Library/Application\ Support/Claude-3p/configLibrary.bak \
   ~/Library/Application\ Support/Claude-3p/configLibrary
```

## Uninstall CLI

```bash
npm uninstall -g @anthropic-ai/claude-code
```

Hapus baris `ANTHROPIC_*`, `CLAUDE_*` dari `~/.zshrc` jika tidak lagi dipakai. Folder `~/.claude` berisi konfigurasi dan riwayat; hapus hanya jika Anda yakin.

## Uninstall Claude Desktop

1. Keluar dari aplikasi.
2. Hapus `Claude.app` dari `/Applications`.
3. Opsional, hapus data aplikasi. **Jangan lakukan jika masih membutuhkan konfigurasi atau riwayat.**

```bash
# opsional, hanya jika yakin
rm -rf ~/Library/Application\ Support/Claude
rm -rf ~/Library/Application\ Support/Claude-3p
rm -rf ~/Library/Caches/Claude
rm -rf ~/Library/Logs/Claude-3p
```

---

# Keamanan

- Jangan commit API key ke Git.
- File tutorial di repo ini memakai placeholder `<API_KEY_DEEPSEEK>`.
- Jika API key pernah terekspos, revoke di dashboard DeepSeek dan buat yang baru.
- Batasi permission file konfigurasi:

```bash
chmod 600 ~/.zshrc
chmod 700 ~/Library/Application\ Support/Claude-3p/configLibrary
chmod 600 ~/Library/Application\ Support/Claude-3p/configLibrary/*.json
```

- `bypassPermissions` dan `skipDangerousModePermissionPrompt` mengurangi pengamanan. Aktifkan hanya di mesin yang Anda percaya.

---

# Referensi

- DeepSeek — Anthropic API compatibility: <https://api-docs.deepseek.com/guides/anthropic_api>
- Claude Code: <https://docs.claude.com/en/docs/claude-code>
- Claude Desktop: <https://claude.ai/download>
- Claude Support: <https://support.claude.com>
- Panduan Windows: [Readme.MD](Readme.MD)
