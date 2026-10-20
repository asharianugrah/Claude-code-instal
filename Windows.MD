# Panduan Claude Code + Claude Desktop dengan DeepSeek (Windows)

Panduan memasang dan mengonfigurasi **Claude Code CLI** serta **Claude Desktop mode 3P (gateway)** dengan **DeepSeek** sebagai backend model.

Panduan macOS tersedia di [macOS.md](macOS.md).

Panduan menghubungkan add-in **Claude for Microsoft Office (Word & Excel)** ke gateway yang sama tersedia di [office-addin-gateway.md](office-addin-gateway.md).

> Tutorial ini disusun dari implementasi pada Windows 11 dengan Claude Desktop `2.42304.2.0` dan API DeepSeek yang kompatibel dengan Anthropic Messages API. Nama menu dan tampilan dapat berubah pada versi Claude Desktop yang lebih baru.

---

## Prasyarat

- Windows 10 build 19041 (2004) atau Windows 11
- PowerShell
- Node.js LTS dan Git
- API Key DeepSeek dari <https://platform.deepseek.com>
- Untuk Cowork:
  - Claude Desktop versi **MSIX** (bukan installer `.exe` lama)
  - Hardware virtualization aktif
  - Fitur **Virtual Machine Platform** aktif
  - Ruang disk yang cukup (bundle workspace > 1 GB; sisakan sekitar 20–30 GB)

---

# Bagian 1 — Claude Code CLI

## 1. Install Node.js

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
winget install OpenJS.NodeJS.LTS
```

Verifikasi:

```powershell
node -v
npm -v
npx -v
```

## 2. Install Git

```powershell
winget install -e --id Git.Git
```

## 3. Install Claude Code

```powershell
npm install -g @anthropic-ai/claude-code
```

## 4. Environment variable DeepSeek

Jalankan di PowerShell. Ganti `<API_KEY_DEEPSEEK>` dengan API key milik Anda.

```powershell
[System.Environment]::SetEnvironmentVariable('ANTHROPIC_API_KEY', '<API_KEY_DEEPSEEK>', 'User')
[System.Environment]::SetEnvironmentVariable('ANTHROPIC_BASE_URL', 'https://api.deepseek.com/anthropic', 'User')

[System.Environment]::SetEnvironmentVariable('ANTHROPIC_MODEL', 'deepseek-v4-flash', 'User')
[System.Environment]::SetEnvironmentVariable('ANTHROPIC_DEFAULT_OPUS_MODEL', 'deepseek-v4-pro[1m]', 'User')
[System.Environment]::SetEnvironmentVariable('ANTHROPIC_DEFAULT_SONNET_MODEL', 'deepseek-v4-flash-vision-exp', 'User')
[System.Environment]::SetEnvironmentVariable('ANTHROPIC_DEFAULT_HAIKU_MODEL', 'deepseek-v4-flash', 'User')
[System.Environment]::SetEnvironmentVariable('CLAUDE_CODE_SUBAGENT_MODEL', 'deepseek-v4-flash', 'User')

[System.Environment]::SetEnvironmentVariable('CLAUDE_EFFORT', 'high', 'User')
[System.Environment]::SetEnvironmentVariable('CLAUDE_CODE_EFFORT_LEVEL', 'high', 'User')
[System.Environment]::SetEnvironmentVariable('CLAUDE_CODE_AUTO_COMPACT_WINDOW', '786432', 'User')
```

Opsional, untuk mengurangi telemetri/atribusi:

```powershell
[System.Environment]::SetEnvironmentVariable('CLAUDE_CODE_ATTRIBUTION_HEADER', '0', 'User')
[System.Environment]::SetEnvironmentVariable('CLAUDE_CODE_ENABLE_TELEMETRY', '0', 'User')
[System.Environment]::SetEnvironmentVariable('CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC', '1', 'User')
```

Tutup dan buka kembali terminal agar environment variable terbaca.

## 5. Konfigurasi CLI

Buat atau edit `%USERPROFILE%\.claude\settings.json`. Isi yang dipakai pada implementasi ini juga tersedia di [Setting.txt](Setting.txt).

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

- `model: "haiku"` adalah alias. Dengan `ANTHROPIC_DEFAULT_HAIKU_MODEL=deepseek-v4-flash`, alias tersebut diarahkan ke DeepSeek Flash.
- `bypassPermissions` membuat Claude Code menjalankan perintah tanpa konfirmasi. Gunakan hanya di environment yang Anda percaya.
- Jangan pernah commit API key ke repository publik.

## 6. Verifikasi CLI

```powershell
claude --version
claude
```

Contoh prompt:

```text
Balas dengan satu kata: OK
```

---

# Bagian 2 — Claude Desktop mode 3P + Cowork

## Penting: arti “Sign in with gateway”

Di Claude Desktop, opsi **Gateway sign-in (OIDC)** / **“Sign in with gateway”** berarti aplikasi melakukan login OIDC ke identity provider (Entra ID, Okta, Keycloak, dan sejenisnya), lalu mengirim token JWT ke gateway.

**DeepSeek tidak menyediakan OIDC dan tidak memvalidasi JWT.** DeepSeek hanya menerima API key. Karena itu, untuk DeepSeek gunakan:

- **Inference provider:** Gateway
- **Credential kind:** Static API key
- **Gateway auth scheme:** `x-api-key`

Jika Anda benar-benar membutuhkan tombol “Sign in with gateway”, itu adalah proyek terpisah: siapkan IdP OIDC + gateway yang memvalidasi JWT (misalnya LiteLLM/Envoy/Kong/Portkey), lalu letakkan DeepSeek di belakangnya.

## 1. Install Claude Desktop

Unduh dan pasang Claude Desktop dari <https://claude.ai/download>.

Untuk Cowork di Windows, gunakan paket **MSIX**. Installer `.exe` lama tidak menyertakan Cowork.

## 2. Aktifkan Developer Mode

1. Buka Claude Desktop.
2. Klik menu aplikasi **☰** di kiri atas.
3. Pilih **Help → Troubleshooting → Enable Developer Mode**.
4. Buka **Developer → Configure Third-Party Inference…**.

## 3. Isi bagian Connection

| Field | Nilai |
|---|---|
| Inference provider | `Gateway` |
| Gateway base URL | `https://api.deepseek.com/anthropic` |
| Gateway auth scheme | `x-api-key` |
| Credential kind | `Static API key` |
| Gateway API key | API key DeepSeek Anda |
| Model discovery | **Off** |

Base URL **tidak** memakai akhiran `/v1`. Claude Desktop menambahkan sendiri `/v1/messages` dan `/v1/models`.

## 4. Isi bagian Models — gunakan alias Claude

Claude Desktop memfilter model ID yang terlihat non-Anthropic. Model seperti `deepseek-v4-flash` akan **dihapus** dari daftar dan picker menjadi kosong.

Gunakan alias Claude yang dipetakan oleh DeepSeek:

| Alias yang dikirim | Model DeepSeek |
|---|---|
| `claude-opus-*` | `deepseek-v4-pro` |
| `claude-sonnet-*` | `deepseek-flash` |
| `claude-haiku-*` | `deepseek-flash` |

Tambahkan model berikut. Entri pertama menjadi default.

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

Catatan:

- `claude-opus-4-5[1m]` adalah varian 1M. DeepSeek mengabaikan header `anthropic-beta`, jadi aman untuk dicoba. Model `claude-opus*` tetap dipetakan ke `deepseek-v4-pro`.
- `defaultModelEffort` dapat diisi `high` (atau `xhigh`) bila ingin menyamakan dengan CLI.
- `anthropicFamilyTier` membantu aplikasi menyelesaikan alias seperti `sonnet`/`opus`/`haiku`.
- Label dapat berubah pada picker karena aplikasi juga membaca model catalog Anthropic. `labelOverride` tetap dipakai untuk model yang tidak ada di catalog.

## 5. Test dan Apply

1. Klik **Test connection** jika tersedia.
2. Klik **Apply Changes**.
3. Aplikasi akan relaunch ke mode 3P.
4. Di layar login, pilih opsi **gateway/third-party inference**, bukan “Sign in with Claude.ai”.

## 6. Verifikasi

Buka log:

```text
%LOCALAPPDATA%\Claude-3p\logs\main.log
```

Baris yang menandakan konfigurasi berhasil:

```text
[custom-3p] 3P mode active { provider: 'gateway' }
[custom-3p] Model discovery: not supported by provider; picker = 4 (inferenceModels)
[custom-3p] inference apiHost=https://api.deepseek.com/anthropic
[custom-3p] ConfigHealth recomputed { state: 'healthy', provider: 'gateway' }
```

Jika muncul:

```text
inferenceModels: "..." is not an Anthropic model and was removed from the list
[custom-3p] Model discovery: ...; picker = 0 (empty)
```

berarti Anda masih memakai model ID DeepSeek. Ganti dengan alias `claude-*` di atas.

## 7. Advanced: local config library (Windows)

In-app configuration adalah cara yang disarankan. Jika perlu otomatisasi atau key `HKCU\SOFTWARE\Policies\Claude` terkunci (read-only), Claude Desktop juga membaca konfigurasi lokal dari:

```text
%LOCALAPPDATA%\Claude-3p\configLibrary\
```

Struktur:

```text
configLibrary\
  _meta.json
  <uuid>.json
```

Contoh `_meta.json`:

```json
{
  "appliedId": "11111111-1111-1111-1111-111111111111",
  "entries": [
    {
      "id": "11111111-1111-1111-1111-111111111111",
      "name": "DeepSeek Gateway"
    }
  ]
}
```

Contoh `<uuid>.json`:

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

Pastikan `deploymentMode` di `%LOCALAPPDATA%\Claude-3p\claude_desktop_config.json` bernilai `"3p"`. Tutup dan buka kembali Claude Desktop setelah mengubah file.

> API key tersimpan plaintext di file ini. Lindungi akses ke akun Windows Anda dan jangan membagikan folder `Claude-3p`.

### Script PowerShell (opsional)

Script berikut membuat config library dari environment variable, tanpa menuliskan key ke dalam script:

```powershell
$ErrorActionPreference = 'Stop'
if (-not $env:ANTHROPIC_API_KEY) { throw 'ANTHROPIC_API_KEY belum diset.' }

$dir = Join-Path $env:LOCALAPPDATA 'Claude-3p\configLibrary'
New-Item -ItemType Directory -Path $dir -Force | Out-Null
if (Test-Path (Join-Path $dir '_meta.json')) {
    Copy-Item (Join-Path $dir '_meta.json') (Join-Path $dir '_meta.json.bak') -Force
}

$id = [guid]::NewGuid().ToString()
$models = @(
    @{ name = 'claude-haiku-4-5';  labelOverride = 'DeepSeek Flash (Haiku)';  anthropicFamilyTier = 'haiku' },
    @{ name = 'claude-sonnet-4-5'; labelOverride = 'DeepSeek Flash (Sonnet)'; anthropicFamilyTier = 'sonnet' },
    @{ name = 'claude-opus-4-5';   labelOverride = 'DeepSeek V4 Pro (Opus)';   anthropicFamilyTier = 'opus' },
    @{ name = 'claude-opus-4-5[1m]'; labelOverride = 'DeepSeek V4 Pro (Opus, 1M)' }
)
$cfg = [ordered]@{
    inferenceProvider          = 'gateway'
    inferenceGatewayBaseUrl    = 'https://api.deepseek.com/anthropic'
    inferenceGatewayApiKey     = $env:ANTHROPIC_API_KEY
    inferenceGatewayAuthScheme = 'x-api-key'
    inferenceCredentialKind    = 'static'
    modelDiscoveryEnabled      = $false
    inferenceModels            = $models
    defaultModelEffort         = 'high'
}
$meta = [ordered]@{
    appliedId = $id
    entries   = @(@{ id = $id; name = 'DeepSeek Gateway' })
}

$utf8 = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText((Join-Path $dir ($id + '.json')), ($cfg | ConvertTo-Json -Depth 20), $utf8)
[System.IO.File]::WriteAllText((Join-Path $dir '_meta.json'), ($meta | ConvertTo-Json -Depth 20), $utf8)

Write-Host "Config ditulis ke $dir. Tutup dan buka kembali Claude Desktop."
```

## 8. Cowork

Cowork **bukan aplikasi terpisah**. Cowork adalah tab dan service (`CoworkVMService` / `cowork-svc.exe`) di dalam Claude Desktop. Konfigurasi gateway yang sama otomatis dipakai oleh Chat, Code, dan Cowork.

Persyaratan Windows:

- Windows 10 build 19041+ atau Windows 11
- Claude Desktop MSIX
- Hardware virtualization aktif
- Fitur **Virtual Machine Platform** aktif
- `CoworkVMService` berjalan
- Akses ke `downloads.claude.ai` dan `api.deepseek.com`
- Ruang disk cukup

Cek kapasitas disk:

```powershell
Get-PSDrive C
```

Jangan menghapus folder `%LOCALAPPDATA%\Claude`, `%LOCALAPPDATA%\Claude-3p`, atau `%PROGRAMDATA%\Claude` secara agresif. Hapus hanya file yang Anda yakin aman, atau gunakan Settings → System → Storage.

Cek service:

```powershell
Get-Service CoworkVMService
```

Jika `rootfs.vhdx` belum ada di log Cowork, itu normal sebelum Cowork pertama kali dijalankan. Run pertama akan mengunduh workspace bundle (> 1 GB) dan Claude CLI dari `downloads.claude.ai`.

Jika muncul error kebijakan keamanan Windows, pastikan grup `NT VIRTUAL MACHINE\Virtual Machines` (SID `S-1-5-83-0`) memiliki hak **Log on as a service**.

Log Cowork:

```text
%LOCALAPPDATA%\Claude\logs\cowork_vm_node.log
%PROGRAMDATA%\Claude\Logs\cowork-service.log
```

## 9. Troubleshooting

| Gejala | Penyebab / solusi |
|---|---|
| “Sign in with gateway” gagal | DeepSeek tidak mendukung OIDC. Gunakan Static API key. |
| Picker model kosong | Model ID `deepseek-*` difilter. Gunakan alias `claude-*`. |
| `GET /v1/models` 404 | DeepSeek tidak menyediakan endpoint models. Matikan model discovery dan isi `inferenceModels`. |
| `401` | API key salah atau tidak terkirim sebagai `x-api-key`. |
| `402` / `429` | Kuota atau rate limit DeepSeek. |
| Aplikasi tetap menampilkan login Claude.ai | Konfigurasi belum terbaca. Periksa `configLibrary`, policy registry, dan `deploymentMode`. |
| Chat/Cowork gagal start | Cek log, koneksi ke `downloads.claude.ai`, dan ruang disk. |
| Web search tidak berjalan | Gateway/DeepSeek tidak meneruskan Anthropic server tools. Fitur ini butuh konfigurasi tambahan. |
| Prompt caching tidak terlihat | DeepSeek mengabaikan `cache_control`; `cache_read_input_tokens` bisa tetap 0. Ini tidak menghalangi chat. |

Catatan keamanan:

- API key tersimpan plaintext di config library atau registry.
- Gunakan key khusus dengan kuota terbatas dan rotasi bila perlu.
- Jangan commit API key ke Git.

## 10. Rollback

- In-app: kembali ke opsi “Sign in with Claude.ai” atau hapus konfigurasi third-party.
- Local config library: hapus `%LOCALAPPDATA%\Claude-3p\configLibrary\*` dan set `deploymentMode` ke `"1p"` di `%LOCALAPPDATA%\Claude-3p\claude_desktop_config.json`.
- Registry: hapus nilai di `HKCU\SOFTWARE\Policies\Claude` atau `HKLM\SOFTWARE\Policies\Claude` bila Anda memakai policy.
- Restart Claude Desktop.

---

## Referensi

- Dokumentasi DeepSeek Anthropic API: <https://api-docs.deepseek.com/guides/anthropic_api>
- Dokumentasi Claude Desktop 3P: <https://claude.com/docs/third-party/claude-desktop>
- Claude Desktop download: <https://claude.ai/download>
