# Panduan Instalasi Claude Code (Windows)

Panduan singkat memasang dan mengonfigurasi Claude Code di Windows, termasuk penggunaan API DeepSeek sebagai backend model.

## Prasyarat

- Windows 10/11
- PowerShell
- Koneksi internet
- API Key DeepSeek (daftar di <https://platform.deepseek.com>)

## Langkah Instalasi

### 1. Install Node.JS

```powershell
winget install OpenJS.NodeJS.LTS
```

Verifikasi instalasi:

```powershell
node -v
npm -v
npx -v
```

### 2. Install Git

```powershell
winget install -e --id Git.Git
```

### 3. Install Claude Code

```powershell
npm install -g @anthropic-ai/claude-code
```

### 4. Set Environment Variable untuk API DeepSeek

Jalankan perintah berikut di PowerShell. Ganti `API_KEY` dengan API Key DeepSeek milik Anda.

```powershell
[System.Environment]::SetEnvironmentVariable('ANTHROPIC_API_KEY', 'API_KEY', 'User')
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "https://api.deepseek.com/anthropic", "User")
```

Pemilihan model:

```powershell
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", "deepseek-v4-flash", "User")
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_OPUS_MODEL", "deepseek-v4-pro[1m]", "User")
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_SONNET_MODEL", "deepseek-v4-flash-vision-exp", "User")
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_HAIKU_MODEL", "deepseek-v4-flash", "User")
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_SUBAGENT_MODEL", "deepseek-v4-flash", "User")
```

Perilaku dan telemetri:

```powershell
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_EFFORT_LEVEL", "high", "User")
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_AUTO_COMPACT_WINDOW", "786432", "User")
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_SIMPLE", "1", "User")
```

Matikan atribusi dan telemetri:

```powershell
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_ATTRIBUTION_HEADER", "0", "User")
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_ENABLE_TELEMETRY", "0", "User")
[System.Environment]::SetEnvironmentVariable("CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC", "1", "User")
```

> Tutup dan buka kembali terminal agar environment variable terbaca.

### 5. Konfigurasi CLI

Buat atau edit file `%USERPROFILE%\.claude\settings.json`, lalu isi dengan konfigurasi berikut (tersedia juga di [Setting.txt](Setting.txt)):

```json
{
  "permissions": {
    "defaultMode": "bypassPermissions"
  },
  "model": "opus",
  "enabledPlugins": {
    "frontend-design@claude-plugins-official": true,
    "context7@claude-plugins-official": true,
    "superpowers@claude-plugins-official": true
  },
  "skipDangerousModePermissionPrompt": true
}
```

## Verifikasi

```powershell
claude --version
claude
```

## Catatan

- `bypassPermissions` membuat Claude Code menjalankan perintah tanpa konfirmasi. Gunakan dengan hati-hati dan hanya di environment yang Anda percaya.
- Pastikan `ANTHROPIC_API_KEY` tidak pernah di-commit ke repository publik.
