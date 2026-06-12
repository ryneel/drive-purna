# 🗂️ Windows Server → Google Drive Otomatik Yedekleme

Masaüstündeki `dark-partner` klasörünü her saat başı ve sunucu açılışında otomatik olarak Google Drive'a yedekler. `node_modules` dahil edilmez. Drive'da her zaman yalnızca **1 güncel yedek** bulunur.

---

## 📋 Gereksinimler

- Windows Server 2019
- PowerShell 5.1+
- Google hesabı
- Google Cloud Console erişimi

---

## 🚀 Kurulum

### 1. Google Cloud Console

1. [console.cloud.google.com](https://console.cloud.google.com) → Yeni proje oluştur
2. **APIs & Services** → **Enable APIs** → **Google Drive API** etkinleştir
3. **APIs & Services** → **Credentials** → **Create Credentials** → **OAuth 2.0 Client ID**
4. Application type: **Web Application** seç
5. **Authorized redirect URIs** kısmına ekle:
   ```
   https://developers.google.com/oauthplayground
   ```
6. **Create** bas → `Client ID` ve `Client Secret`'ı kopyala
7. Sol menüden **Audience** → **Test users** → Gmail adresini ekle → Save

---

### 2. Refresh Token Alma

Aşağıdaki scripti `get_token.ps1` olarak kaydet, bilgileri doldur ve çalıştır:

```powershell
$CLIENT_ID     = "BURAYA_CLIENT_ID"
$CLIENT_SECRET = "BURAYA_CLIENT_SECRET"

$authUrl = "https://accounts.google.com/o/oauth2/auth?" +
    "client_id=$CLIENT_ID" +
    "&redirect_uri=urn:ietf:wg:oauth:2.0:oob" +
    "&response_type=code" +
    "&scope=https://www.googleapis.com/auth/drive" +
    "&access_type=offline"

Write-Host "Tarayicida su URL'yi ac:"
Write-Host $authUrl
Start-Process $authUrl

$authCode = Read-Host "Tarayicidan gelen kodu buraya yapistir"

$body = @{
    code          = $authCode
    client_id     = $CLIENT_ID
    client_secret = $CLIENT_SECRET
    redirect_uri  = "urn:ietf:wg:oauth:2.0:oob"
    grant_type    = "authorization_code"
}

$resp = Invoke-RestMethod -Uri "https://oauth2.googleapis.com/token" -Method Post -Body $body

Write-Host ""
Write-Host "=== REFRESH TOKEN (bunu kopyala) ==="
Write-Host $resp.refresh_token
```

```powershell
powershell -ExecutionPolicy Bypass -File "C:\Users\Administrator\Desktop\get_token.ps1"
```

> Tarayıcı açılır → Google hesabına giriş yap → İzin ver → Ekrandaki kodu PowerShell'e yapıştır → **Refresh Token** ekrana gelir, kopyala.

---

### 3. Drive Klasör ID'si

Google Drive'da yedeklerin gideceği klasörü aç, URL'deki son kısmı kopyala:

```
https://drive.google.com/drive/folders/BURAYA_FOLDER_ID
                                        ^^^^^^^^^^^^^^^^
                                        bu kısım = FOLDER_ID
```

---

### 4. Kurulum Scriptini Hazırla

Aşağıdaki scripti `backup_setup.ps1` olarak kaydet, üstteki **4 değişkeni** doldur:

```powershell
# backup_setup.ps1 - Yönetici olarak bir kez çalıştır

$scriptDir = "C:\BackupSystem"
New-Item -ItemType Directory -Force -Path $scriptDir | Out-Null

$backupScript = @'
$CLIENT_ID     = "BURAYA_CLIENT_ID"
$CLIENT_SECRET = "BURAYA_CLIENT_SECRET"
$REFRESH_TOKEN = "BURAYA_REFRESH_TOKEN"
$FOLDER_ID     = "BURAYA_FOLDER_ID"

$logFile   = "C:\BackupSystem\backup.log"
$zipFile   = "C:\BackupSystem\dark_partner_backup.zip"
$sourceDir = [Environment]::GetFolderPath("Desktop") + "\dark-partner"
$tempDir   = "C:\BackupSystem\temp_backup"

function Log($msg) {
    $line = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') - $msg"
    Add-Content -Path $logFile -Value $line
    Write-Host $line
}

function Get-AccessToken {
    $body = @{
        client_id     = $CLIENT_ID
        client_secret = $CLIENT_SECRET
        refresh_token = $REFRESH_TOKEN
        grant_type    = "refresh_token"
    }
    $resp = Invoke-RestMethod -Uri "https://oauth2.googleapis.com/token" -Method Post -Body $body
    return $resp.access_token
}

function Delete-OldBackups($token) {
    Log "Eski yedekler siliniyor..."
    $headers = @{ Authorization = "Bearer $token" }
    $query   = [uri]::EscapeDataString("'$FOLDER_ID' in parents and name contains 'dark_partner_backup' and trashed=false")
    $listUri = "https://www.googleapis.com/drive/v3/files?q=$query&fields=files(id,name)"
    $files   = Invoke-RestMethod -Uri $listUri -Headers $headers
    foreach ($file in $files.files) {
        Invoke-RestMethod -Uri "https://www.googleapis.com/drive/v3/files/$($file.id)" `
            -Method Delete -Headers $headers | Out-Null
        Log "Silindi: $($file.name)"
    }
}

function Copy-WithoutNodeModules($source, $dest) {
    if (Test-Path $dest) { Remove-Item $dest -Recurse -Force }
    New-Item -ItemType Directory -Force -Path $dest | Out-Null

    Get-ChildItem -Path $source -Recurse | Where-Object {
        $_.FullName -notmatch "\\node_modules\\" -and
        $_.FullName -notmatch "\\node_modules$"
    } | ForEach-Object {
        $targetPath = $_.FullName.Replace($source, $dest)
        if ($_.PSIsContainer) {
            New-Item -ItemType Directory -Force -Path $targetPath | Out-Null
        } else {
            $parentDir = Split-Path $targetPath -Parent
            if (-not (Test-Path $parentDir)) {
                New-Item -ItemType Directory -Force -Path $parentDir | Out-Null
            }
            Copy-Item -Path $_.FullName -Destination $targetPath -Force
        }
    }
}

function Upload-Backup($token, $zipPath) {
    Log "Drive'a yukleniyor..."
    $fileName  = "dark_partner_backup_$(Get-Date -Format 'yyyyMMdd_HHmmss').zip"
    $fileBytes = [System.IO.File]::ReadAllBytes($zipPath)

    $meta = @{
        name    = $fileName
        parents = @($FOLDER_ID)
    } | ConvertTo-Json

    $headers = @{ Authorization = "Bearer $token" }

    $createResp = Invoke-RestMethod `
        -Uri "https://www.googleapis.com/drive/v3/files" `
        -Method Post -Headers $headers `
        -ContentType "application/json" `
        -Body $meta

    $fileId = $createResp.id

    $uploadHeaders = @{
        Authorization  = "Bearer $token"
        "Content-Type" = "application/zip"
    }
    Invoke-RestMethod `
        -Uri "https://www.googleapis.com/upload/drive/v3/files/${fileId}?uploadType=media" `
        -Method Patch -Headers $uploadHeaders `
        -Body $fileBytes | Out-Null

    Log "Yuklendi: $fileName (ID: $fileId)"
}

Log "===== YEDEKLEME BASLADI ====="
try {
    if (-not (Test-Path $sourceDir)) {
        Log "HATA: Kaynak klasor bulunamadi: $sourceDir"
        exit 1
    }

    Log "dark-partner kopyalaniyor (node_modules haric)..."
    Copy-WithoutNodeModules $sourceDir $tempDir

    if (Test-Path $zipFile) { Remove-Item $zipFile -Force }
    Log "Sikistiriliyor..."
    Compress-Archive -Path "$tempDir\*" -DestinationPath $zipFile -Force
    $sizeMB = [math]::Round((Get-Item $zipFile).Length / 1MB, 2)
    Log "ZIP hazir: $sizeMB MB"

    Remove-Item $tempDir -Recurse -Force

    $token = Get-AccessToken
    Delete-OldBackups $token
    Upload-Backup $token $zipFile

    Log "===== YEDEKLEME TAMAMLANDI ====="
} catch {
    Log "HATA: $_"
}
'@

Set-Content -Path "$scriptDir\run_backup.ps1" -Value $backupScript -Encoding UTF8

$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
           -Argument "-NonInteractive -WindowStyle Hidden -ExecutionPolicy Bypass -File `"$scriptDir\run_backup.ps1`""

$triggers = @(
    $(New-ScheduledTaskTrigger -AtStartup),
    $(New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Hours 1) -Once -At (Get-Date).Date)
)

$settings  = New-ScheduledTaskSettingsSet -ExecutionTimeLimit (New-TimeSpan -Hours 1) `
             -RestartCount 3 -RestartInterval (New-TimeSpan -Minutes 5)
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -RunLevel Highest

Register-ScheduledTask -TaskName "DarkPartnerBackup" `
    -Action $action -Trigger $triggers -Settings $settings `
    -Principal $principal -Force | Out-Null

Write-Host "=== KURULUM TAMAMLANDI ==="
Write-Host "Log: C:\BackupSystem\backup.log"
```

---

### 5. Kurulumu Başlat

```powershell
# Yönetici PowerShell ile çalıştır
powershell -ExecutionPolicy Bypass -File "C:\Users\Administrator\Desktop\backup_setup.ps1"
```

İlk yedeği hemen al:

```powershell
powershell -ExecutionPolicy Bypass -File "C:\BackupSystem\run_backup.ps1"
```

---

## ✅ Doğrulama

```powershell
Get-ScheduledTask -TaskName "DarkPartnerBackup"
```

`State: Ready` görünüyorsa kurulum tamamdır.

---

## 📁 Dosya Yapısı

```
C:\BackupSystem\
├── run_backup.ps1       # Ana yedekleme scripti (backup_setup.ps1 tarafından oluşturulur)
└── backup.log           # Tüm işlemlerin logu
```

---

## 📄 Log Örneği

```
2026-06-12 12:46:00 - ===== YEDEKLEME BASLADI =====
2026-06-12 12:46:00 - dark-partner kopyalaniyor (node_modules haric)...
2026-06-12 12:46:03 - Sikistiriliyor...
2026-06-12 12:46:30 - ZIP hazir: 1.63 MB
2026-06-12 12:46:31 - Eski yedekler siliniyor...
2026-06-12 12:46:32 - Silindi: dark_partner_backup_20260612_114600.zip
2026-06-12 12:46:32 - Drive'a yukleniyor...
2026-06-12 12:46:36 - Yuklendi: dark_partner_backup_20260612_124600.zip
2026-06-12 12:46:36 - ===== YEDEKLEME TAMAMLANDI =====
```

---

## ⚙️ Nasıl Çalışır

| Adım | Açıklama |
|------|----------|
| 1 | `dark-partner` klasörü `node_modules` hariç temp klasöre kopyalanır |
| 2 | ZIP olarak sıkıştırılır |
| 3 | Drive'daki eski yedek silinir |
| 4 | Yeni ZIP Drive klasörüne yüklenir |
| 🔁 | Her saat başı ve sunucu açılışında tekrarlanır |
