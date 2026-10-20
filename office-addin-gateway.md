# Claude for Microsoft Office (Word & Excel) dengan Gateway DeepSeek

Panduan menghubungkan add-in resmi **Claude for Microsoft Office** ke akun gateway yang sama dengan Claude Desktop — tanpa langganan Claude terpisah dan tanpa membuat add-in baru.

Panduan Claude Code CLI dan Claude Desktop ada di [Readme.MD](Readme.MD). Panduan macOS ada di [macOS.md](macOS.md).

> Diuji pada Windows 11, Office 365 `16.0.17932.21056`, Node.js 24, add-in **Claude in Microsoft Office** (Excel `1.0.0.8`, Word `1.0.0.1`), dan gateway DeepSeek yang kompatibel dengan Anthropic Messages API.

---

## Ringkasan arsitektur

```
┌──────────────────────────────┐
│ Excel / Word (Office.js)     │
│ add-in Claude                │
└──────────────┬───────────────┘
               │ HTTP ke 127.0.0.1:8789
               │ (CORS origin https://pivot.claude.ai)
┌──────────────▼───────────────┐
│ Proxy lokal gateway.mjs      │
│ - GET  /v1/models            │  ← dilayani sendiri
│ - POST /v1/messages          │  ← diteruskan ke upstream
│ - normalisasi tools          │
└──────────────┬───────────────┘
               │ HTTPS + x-api-key
┌──────────────▼───────────────┐
│ https://api.deepseek.com     │
│ /anthropic/v1/messages       │
└──────────────────────────────┘
```

Kenapa perlu proxy kecil di antaranya:

1. **`GET /v1/models` tidak ada di jalur `/anthropic`.** Add-in memakai endpoint ini untuk uji koneksi. Tanpa endpoint itu, muncul error *"Gateway returned an error — HTTP 404"*.
2. **Tool dari add-in memakai `"type": "custom"`.** Parser Anthropic-compatible DeepSeek hanya mengenal tool klien tanpa `type` dan tool server `web_search_*`, sehingga request ditolak **HTTP 422**. Proxy membuang `type` dari tool klien dan menyaring tool server yang tidak dikenal.
3. **CORS dan Private Network Access.** Add-in berjalan di origin `https://pivot.claude.ai` dan memanggil `127.0.0.1`; proxy menambahkan header CORS dan `Access-Control-Allow-Private-Network`.

---

## Prasyarat

- Claude Desktop sudah berjalan di mode gateway DeepSeek (lihat [Readme.MD](Readme.MD)).
- Add-in **Claude for Microsoft Office** sudah terpasang di Excel dan Word.
- Node.js LTS dan PowerShell.
- API Key DeepSeek yang sama dengan yang dipakai Claude Desktop.
- Excel/Word dan proxy berjalan di komputer yang sama (proxy hanya mendengarkan `127.0.0.1`).

---

## Bagian 1 — Siapkan folder runtime

Semua berkas gateway disimpan di satu folder tetap:

```powershell
$dir = "$env:USERPROFILE\claude-office-gateway"
New-Item -ItemType Directory -Force -Path $dir | Out-Null
```

> **Jangan** menaruh folder ini di `%LOCALAPPDATA%`. Pada sebagian konfigurasi, proses Task Scheduler ditolak saat mengakses folder tersebut (gejalanya: `LastTaskResult` `2147942667` / `1` dan log tidak pernah dibuat). `%USERPROFILE%` aman.

---

## Bagian 2 — Buat proxy `gateway.mjs`

Simpan berkas berikut sebagai `%USERPROFILE%\claude-office-gateway\gateway.mjs`.

```js
import http from 'node:http'

const HOST = '127.0.0.1'
const PORT = Number(process.env.PORT || 8789)
const UPSTREAM = process.env.UPSTREAM || 'https://api.deepseek.com/anthropic'
const ORIGIN = 'https://pivot.claude.ai'

const MODELS = {
  data: [
    { type: 'model', id: 'claude-opus-4-5', display_name: 'Claude Opus 4.5', created_at: '2025-11-24T00:00:00Z' },
    { type: 'model', id: 'claude-sonnet-4-5', display_name: 'Claude Sonnet 4.5', created_at: '2025-09-29T00:00:00Z' },
    { type: 'model', id: 'claude-haiku-4-5', display_name: 'Claude Haiku 4.5', created_at: '2025-10-01T00:00:00Z' },
  ],
  has_more: false,
}
MODELS.first_id = MODELS.data[0].id
MODELS.last_id = MODELS.data.at(-1).id

function cors(req, res) {
  res.setHeader('Access-Control-Allow-Origin', ORIGIN)
  res.setHeader('Vary', 'Origin')
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, OPTIONS')
  res.setHeader('Access-Control-Allow-Headers', req.headers['access-control-request-headers'] || 'x-api-key, authorization, content-type, anthropic-version, anthropic-beta')
  res.setHeader('Access-Control-Expose-Headers', 'request-id, x-request-id, anthropic-ratelimit-requests-remaining')
  res.setHeader('Access-Control-Allow-Private-Network', 'true')
  res.setHeader('Access-Control-Max-Age', '86400')
}

function json(req, res, status, body) {
  cors(req, res)
  res.writeHead(status, { 'content-type': 'application/json; charset=utf-8' })
  res.end(JSON.stringify(body))
}

async function readBody(req) {
  const chunks = []
  for await (const chunk of req) chunks.push(chunk)
  return Buffer.concat(chunks)
}

// Parser DeepSeek hanya mengenal tool klien (tanpa "type") dan dua tool server
// web_search. Add-in menandai tool klien dengan "type":"custom".
function adaptPayload(raw) {
  let payload
  try {
    payload = JSON.parse(raw.toString('utf8'))
  } catch {
    return raw
  }
  if (Array.isArray(payload.tools)) {
    const kept = []
    for (const tool of payload.tools) {
      if (!tool || typeof tool !== 'object') continue
      if (tool.type === 'custom' || tool.type === undefined) {
        const { type: _type, ...clientTool } = tool
        if (clientTool.name) {
          if (!clientTool.input_schema) clientTool.input_schema = { type: 'object', properties: {} }
          kept.push(clientTool)
        }
      } else if (tool.type === 'web_search_20250305' || tool.type === 'web_search_20260209') {
        kept.push(tool)
      }
    }
    if (kept.length) payload.tools = kept
    else delete payload.tools
  }
  if (Array.isArray(payload.mcp_servers)) delete payload.mcp_servers
  if (payload.container) delete payload.container
  return Buffer.from(JSON.stringify(payload), 'utf8')
}

async function forwardMessages(req, res) {
  const body = adaptPayload(await readBody(req))
  const headers = {
    'content-type': 'application/json',
    'anthropic-version': req.headers['anthropic-version'] || '2023-06-01',
    accept: req.headers.accept || 'application/json',
  }
  if (req.headers['x-api-key']) headers['x-api-key'] = req.headers['x-api-key']
  else if (req.headers.authorization) headers.authorization = req.headers.authorization
  if (req.headers['anthropic-beta']) headers['anthropic-beta'] = req.headers['anthropic-beta']

  let upstream
  try {
    upstream = await fetch(UPSTREAM + '/v1/messages', { method: 'POST', headers, body })
  } catch (err) {
    json(req, res, 502, { type: 'error', error: { type: 'api_error', message: 'Upstream unreachable: ' + err.message } })
    return
  }

  const outHeaders = {}
  for (const name of ['content-type', 'request-id', 'x-request-id', 'anthropic-ratelimit-requests-remaining']) {
    const value = upstream.headers.get(name)
    if (value) outHeaders[name] = value
  }
  cors(req, res)
  res.writeHead(upstream.status, outHeaders)

  if (!upstream.body) {
    res.end()
    return
  }
  try {
    for await (const chunk of upstream.body) res.write(chunk)
  } catch {
    // klien memutus koneksi atau upstream berhenti
  }
  res.end()
}

const server = http.createServer(async (req, res) => {
  const url = new URL(req.url, 'http://localhost')

  if (req.method === 'OPTIONS') {
    cors(req, res)
    res.writeHead(204)
    res.end()
    return
  }

  if (req.method === 'GET' && (url.pathname === '/' || url.pathname === '/healthz')) {
    cors(req, res)
    res.writeHead(200, { 'content-type': 'text/plain' })
    res.end('claude-office-gateway ok')
    return
  }

  if (req.method === 'GET' && url.pathname === '/v1/models') {
    json(req, res, 200, MODELS)
    return
  }

  if (req.method === 'GET' && url.pathname.startsWith('/v1/models/')) {
    const id = decodeURIComponent(url.pathname.slice('/v1/models/'.length))
    const model = MODELS.data.find((m) => m.id === id)
    if (model) json(req, res, 200, model)
    else json(req, res, 404, { type: 'error', error: { type: 'not_found_error', message: 'model not found' } })
    return
  }

  if (req.method === 'POST' && url.pathname === '/v1/messages') {
    await forwardMessages(req, res)
    return
  }

  json(req, res, 404, { type: 'error', error: { type: 'not_found_error', message: 'not found: ' + url.pathname } })
})

server.listen(PORT, HOST, () => {
  console.log(`claude-office-gateway listening on http://${HOST}:${PORT} -> ${UPSTREAM}`)
})
```

Catatan:

- Proxy **tidak menyimpan API key**. Key yang dikirim add-in diteruskan apa adanya ke upstream.
- Alias model di atas adalah alias yang diterima gateway. Nama model asli DeepSeek juga bisa dipakai.
- Port dapat diubah lewat variabel lingkungan `PORT`; upstream lewat `UPSTREAM`.

---

## Bagian 3 — Jalankan dan uji proxy

Jalankan manual untuk uji cepat:

```powershell
node "$env:USERPROFILE\claude-office-gateway\gateway.mjs"
```

Di terminal lain:

```powershell
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:8789/healthz
```

Hasil yang diharapkan:

```
claude-office-gateway ok
```

Uji inference dengan tool `type: "custom"` (persis seperti yang dikirim add-in):

```powershell
$key = Read-Host "DeepSeek API Key"
$body = @{
  model      = "claude-haiku-4-5"
  max_tokens = 64
  messages   = @(@{ role = "user"; content = "Say OK." })
  tools      = @(
    @{ type = "custom"; name = "get_selected_cells"; description = "Read selected cells."; input_schema = @{ type = "object"; properties = @{} } }
  )
} | ConvertTo-Json -Depth 10

Invoke-RestMethod -Uri http://127.0.0.1:8789/v1/messages -Method Post `
  -Headers @{ "x-api-key" = $key; "anthropic-version" = "2023-06-01" } `
  -ContentType "application/json" -Body $body
```

Jika berhasil, respons berisi `"type": "message"` dan `"text": "OK"` — bukan `422`.

---

## Bagian 4 — Hubungkan add-in Excel dan Word

Lakukan di **masing-masing** aplikasi (konfigurasi Excel dan Word terpisah).

1. Buka Excel atau Word, lalu buka panel **Claude** (tab *Home* → *Claude*).
2. Pada layar masuk, pilih **Cloud provider or gateway**.
3. Pilih **Gateway**, lalu isi:

   | Kolom | Nilai |
   |---|---|
   | Gateway URL | `http://127.0.0.1:8789` |
   | API Key / Token | API Key DeepSeek Anda |
   | Auth header | `x-api-key` |
   | API format | `anthropic` |
   | Model | `claude-haiku-4-5`, `claude-sonnet-4-5`, atau `claude-opus-4-5` |

4. Klik **Connect / Sign in**.
5. Setelah masuk, kirim pesan uji, misalnya: *"Tulis di sel A1: Halo dari Claude"*.

Konfigurasi ini **tersimpan di Office**. Saat komputer di-restart, Anda tidak perlu mengisi ulang URL, token, atau format — yang harus hidup hanya proses proxy (lihat Bagian 5).

> Jika add-in menanyakan model dan daftar otomatis kosong, isi ID model secara manual dengan salah satu alias di atas.

---

## Bagian 5 — Jalankan otomatis saat login

Agar proxy selalu hidup setelah restart, daftarkan sebagai Scheduled Task yang berjalan tersembunyi dan otomatis diulang jika crash.

### 5.1 Buat launcher

Buat `%USERPROFILE%\claude-office-gateway\start-gateway.cmd`:

```bat
@echo off
"C:\Program Files\nodejs\node.exe" "C:\Users\<NAMA_ANDA>\claude-office-gateway\gateway.mjs" >> "C:\Users\<NAMA_ANDA>\claude-office-gateway\gateway.log" 2>&1
```

Ganti `<NAMA_ANDA>` dengan nama folder profil Anda.

Buat `%USERPROFILE%\claude-office-gateway\start-gateway.vbs` (menjalankan launcher tanpa jendela dan menunggu proses selesai):

```vbscript
Set sh = CreateObject("WScript.Shell")
code = sh.Run("cmd /c ""C:\Users\<NAMA_ANDA>\claude-office-gateway\start-gateway.cmd""", 0, True)
WScript.Quit code
```

### 5.2 Daftarkan task

Jalankan PowerShell:

```powershell
$user = "$env:USERDOMAIN\$env:USERNAME"
$launcher = "$env:USERPROFILE\claude-office-gateway\start-gateway.vbs"

$action = New-ScheduledTaskAction -Execute 'C:\WINDOWS\System32\wscript.exe' -Argument ('"' + $launcher + '"')
$trigger = New-ScheduledTaskTrigger -AtLogOn -User $user
$settings = New-ScheduledTaskSettingsSet `
  -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries `
  -ExecutionTimeLimit ([TimeSpan]::Zero) `
  -RestartCount 10 -RestartInterval (New-TimeSpan -Minutes 1) `
  -MultipleInstances IgnoreNew -StartWhenAvailable
$principal = New-ScheduledTaskPrincipal -UserId $user -LogonType Interactive -RunLevel Limited

Register-ScheduledTask -TaskName 'ClaudeOfficeGateway' `
  -Action $action -Trigger $trigger -Settings $settings -Principal $principal `
  -Description 'Gateway lokal Anthropic-compatible untuk Claude for Microsoft Office' -Force

Start-ScheduledTask -TaskName 'ClaudeOfficeGateway'
Start-Sleep -Seconds 3
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:8789/healthz
```

### 5.3 Verifikasi

```powershell
Get-ScheduledTask -TaskName ClaudeOfficeGateway | Select-Object TaskName, State
Get-ScheduledTaskInfo -TaskName ClaudeOfficeGateway | Select-Object LastRunTime, LastTaskResult
Get-Content "$env:USERPROFILE\claude-office-gateway\gateway.log"
```

- `State` harus `Running`.
- Log berisi `claude-office-gateway listening on http://127.0.0.1:8789`.
- `LastTaskResult` `0` berarti normal (task memang berjalan terus; hasil baru terisi setelah proses berhenti).

### 5.4 Menghentikan / menghapus otomatis

```powershell
Stop-ScheduledTask -TaskName ClaudeOfficeGateway
```

```powershell
Unregister-ScheduledTask -TaskName ClaudeOfficeGateway -Confirm:$false
```

---

## Bagian 6 — Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| *"Gateway returned an error — HTTP 404"* saat Connect | URL diisi `https://api.deepseek.com/anthropic`; `/v1/models` tidak ada di jalur itu | Isi URL dengan `http://127.0.0.1:8789` |
| **HTTP 422**: `tools[0]: unknown variant 'custom'` | Add-in mengirim `"type":"custom"`; parser DeepSeek menolak | Pastikan `gateway.mjs` versi terbaru (ada fungsi `adaptPayload`) dan restart proxy |
| *"Unable to connect"* / connection refused | Proxy tidak berjalan | Jalankan manual atau `Start-ScheduledTask -TaskName ClaudeOfficeGateway` |
| Error CORS di console browser | Origin bukan `https://pivot.claude.ai` atau header CORS hilang | Gunakan proxy ini; jangan arahkan langsung ke upstream |
| Task `LastTaskResult` `2147942667` (`0x8007010B`) atau `1`, log tidak dibuat | Folder runtime di `%LOCALAPPDATA%` ditolak proses Task Scheduler | Pindahkan folder ke `%USERPROFILE%\claude-office-gateway` |
| HTTP 401 | Token salah atau bukan API key gateway | Salin ulang API key DeepSeek |
| Model error / model not found | ID model tidak dikenal gateway | Pakai `claude-haiku-4-5`, `claude-sonnet-4-5`, atau `claude-opus-4-5` |
| Login berhasil tetapi pesan gagal | Tool/field yang tidak didukung upstream | Periksa log proxy; pastikan normalisasi tools aktif |

Tips diagnostik:

```powershell
# proses yang memegang port
Get-NetTCPConnection -LocalPort 8789 -State Listen | Select-Object OwningProcess

# hentikan proxy yang menempel
Get-NetTCPConnection -LocalPort 8789 -State Listen | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
```

---

## Bagian 7 — Keamanan

- Proxy hanya mendengarkan `127.0.0.1`, sehingga tidak dapat diakses dari jaringan lain.
- API key **tidak** ditulis di `gateway.mjs`, log, atau repo. Key hanya tersimpan di konfigurasi add-in dan dikirim per request.
- Jangan membagikan API key. Jika bocor, rotasi key di dashboard DeepSeek.
- Siapa pun yang punya akses ke sesi Windows Anda dapat memanggil proxy, tetapi tetap harus menyertakan API key yang valid.
- Add-in menyimpan konfigurasi koneksi. Jika komputer dipakai bergantian, putuskan koneksi dari panel add-in saat selesai.

---

## Bagian 8 — Rollback

1. Hapus task:

   ```powershell
   Unregister-ScheduledTask -TaskName ClaudeOfficeGateway -Confirm:$false
   ```

2. Hentikan proses yang masih memegang port:

   ```powershell
   Get-NetTCPConnection -LocalPort 8789 -State Listen -ErrorAction SilentlyContinue | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
   ```

3. Hapus folder `%USERPROFILE%\claude-office-gateway`.
4. Di add-in Excel/Word, pilih keluar dari akun gateway, lalu sambungkan kembali ke akun Claude atau biarkan kosong.

---

## Referensi

- Dokumentasi Claude for M365 dengan third-party platform: <https://claude.com/docs/office-agents/third-party-platforms.md>
- Panduan Claude Desktop + DeepSeek di repo ini: [Readme.MD](Readme.MD)
- Panduan macOS: [macOS.md](macOS.md)
