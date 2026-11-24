# CustomADBTOOL



```powershell
#requires -Version 5.1
<#
  =====================================================================
   ADB MULTI TOOL  -  Setup / Builder  (einmal ausfuehren)
  =====================================================================
   Erstellt unter <Root>:
     platform-tools\           offizielles Google-Paket (adb, fastboot)
     Start-MultiTool.ps1       das Tool (Konsolen-Menue)
     ADB-MultiTool.cmd         Doppelklick-Starter
     lib\Common.ps1            adb-Wrapper, Geraete, Config
     lib\Logger.ps1            Logger-Engine (alle Input-/System-Streams)
     lib\CallTrace.ps1         Anruf-Rueckverfolgung + Nummern-Aufloesung
     lib\SimTool.ps1           SIM/Netz All-in-One (USSD, STK, APN, IMEI)
     logs\                     Logs pro Geraet und Sitzung
     config\settings.json      Einstellungen

   Start:  powershell -ExecutionPolicy Bypass -File "<Root>\Start-MultiTool.ps1"

   EHRLICHE GRENZEN (damit du nicht mehr erwartest als moeglich ist):
   1) "Jeder Tastenanschlag": ohne Root liefert Android Input-Events des
      Touch-/Hardkey-Devices ueber getevent plus InputReader-Logs, dazu
      Fokus-, Activity-, Clipboard-, Notification- und Media-Aenderungen.
      Getippter Text erscheint als Key-Code, nicht als fertiges Wort -
      komplette Woerter nur, wo Android den Feldinhalt hergibt.
      Mehr geht nur mit Root (su) - das Tool versucht den Root-Fallback.
   2) Standort des ANRUFERS: GSM uebertraegt keinen GPS-Punkt. Du bekommst
      Nummer -> Land/Vorwahl/Netz/Typ, optional Online-Lookup per API-Key,
      plus Funkzellen-Daten (Cell-ID/LAC) deines eigenen Geraets.
      Echte GPS-Position des Anrufers gibt es nur bei Apps wie WhatsApp/Signal
      oder wenn der Anrufer sie teilt.
   3) SIM: Android exponiert nur, was die Telefonie-Schicht freigibt
      (ICCID/IMSI/IMEI je nach Hersteller und Root, Netz, Zellen, USSD, STK,
      APN). Vollzugriff auf SIM-EF-Files braucht einen PC/SC-Kartenleser.
  =====================================================================
#>
[CmdletBinding()]
param(
    [string]$Root = "$env:USERPROFILE\ADBMultiTool"
)

$ErrorActionPreference = 'Stop'
$ProgressPreference    = 'SilentlyContinue'

function Write-Text {
    param([string]$Path, [string]$Content)
    $dir = Split-Path -Parent $Path
    if (-not (Test-Path $dir)) { New-Item -ItemType Directory -Force -Path $dir | Out-Null }
    Set-Content -LiteralPath $Path -Value $Content -Encoding UTF8
}

Write-Host "`n=== ADB Multi Tool Setup ===" -ForegroundColor Cyan
Write-Host "Ziel: $Root`n"

foreach ($d in @($Root, (Join-Path $Root 'lib'), (Join-Path $Root 'logs'), (Join-Path $Root 'config'))) {
    if (-not (Test-Path $d)) { New-Item -ItemType Directory -Force -Path $d | Out-Null }
}

# ---------------------------------------------------------------- platform-tools
$PtDir = Join-Path $Root 'platform-tools'
$AdbExe = Join-Path $PtDir 'adb.exe'
if (-not (Test-Path $AdbExe)) {
    Write-Host 'Lade offizielle Android platform-tools (Google)...' -ForegroundColor Yellow
    $zip = Join-Path $env:TEMP 'platform-tools-latest-windows.zip'
    Invoke-WebRequest -Uri 'https://dl.google.com/android/repository/platform-tools-latest-windows.zip' -OutFile $zip -UseBasicParsing
    Write-Host 'Entpacke...' -ForegroundColor Yellow
    Expand-Archive -LiteralPath $zip -DestinationPath $Root -Force
    Remove-Item $zip -Force -ErrorAction SilentlyContinue
}
if (-not (Test-Path $AdbExe)) { throw 'adb.exe wurde nicht gefunden - Download/Entpacken fehlgeschlagen.' }
Write-Host 'platform-tools OK.' -ForegroundColor Green

# ---------------------------------------------------------------- config
Write-Text (Join-Path $Root 'config\settings.json') @'
{
  "PhoneLookupKey": "",
  "PollMs": 1000,
  "ScreenshotSec": 0,
  "MergeOnStop": false
}
'@

# ================================================================ Common.ps1
Write-Text (Join-Path $Root 'lib\Common.ps1') @'
$script:Root    = Split-Path -Parent $PSScriptRoot
$script:Adb     = Join-Path $script:Root 'platform-tools\adb.exe'
$script:Config  = Join-Path $script:Root 'config\settings.json'
$script:LogRoot = Join-Path $script:Root 'logs'
$script:Serial  = $null
$script:LogDir  = $null
$script:LogJobs = @()

if (-not (Test-Path $script:LogRoot)) { New-Item -ItemType Directory -Force -Path $script:LogRoot | Out-Null }

function Get-Config {
    if (Test-Path $script:Config) {
        try { return (Get-Content $script:Config -Raw | ConvertFrom-Json) } catch { }
    }
    return [pscustomobject]@{ PhoneLookupKey = ''; PollMs = 1000; ScreenshotSec = 0; MergeOnStop = $false }
}

function Save-Config {
    param($Cfg)
    $Cfg | ConvertTo-Json -Depth 5 | Set-Content -LiteralPath $script:Config -Encoding UTF8
}

function Invoke-Adb {
    param(
        [string[]]$Arguments = @(),
        [switch]$NoSerial,
        [switch]$Show
    )
    $a = @()
    if ($script:Serial -and -not $NoSerial) { $a += @('-s', $script:Serial) }
    $a += $Arguments
    try { $out = (& $script:Adb @a 2>&1 | Out-String) }
    catch { $out = "ADB-Fehler: $($_.Exception.Message)" }
    if ($Show) { Write-Host $out }
    return $out
}

function Assert-Device {
    if (-not $script:Serial) { Write-Host 'Kein Geraet gewaehlt (Punkt 1/2).' -ForegroundColor Yellow; return $false }
    $list = Invoke-Adb -Arguments @('devices') -NoSerial
    if ($list -match ([regex]::Escape($script:Serial) + '\s+device')) { return $true }
    Write-Host 'Geraet nicht (mehr) verbunden.' -ForegroundColor Yellow
    return $false
}

function Show-Devices {
    $list = Invoke-Adb -Arguments @('devices', '-l') -NoSerial
    Write-Host "`n--- adb devices ---" -ForegroundColor Cyan
    Write-Host $list
    Write-Host '--------------------'
}

function Select-Device {
    $list = Invoke-Adb -Arguments @('devices') -NoSerial
    $serials = @()
    foreach ($line in ($list -split "`r?`n")) {
        if ($line -match '^(\S+)\s+device\s*$') { $serials += $Matches[1] }
    }
    if ($serials.Count -eq 0) { Write-Host 'Kein verbundenes Geraet. Erst Punkt 1 (connect) oder USB.' -ForegroundColor Yellow; return }
    if ($serials.Count -eq 1) { $script:Serial = $serials[0]; Write-Host "Gewaehlt: $script:Serial" -ForegroundColor Green; return }
    for ($i = 0; $i -lt $serials.Count; $i++) { Write-Host ("[{0}] {1}" -f ($i + 1), $serials[$i]) }
    $idx = Read-Host 'Nummer waehlen'
    if ($idx -match '^\d+$' -and [int]$idx -ge 1 -and [int]$idx -le $serials.Count) {
        $script:Serial = $serials[[int]$idx - 1]
        Write-Host "Gewaehlt: $script:Serial" -ForegroundColor Green
    }
}

function Connect-Wireless {
    $target = Read-Host 'IP:Port (z.B. 192.168.1.50:5555)'
    if (-not $target) { return }
    Invoke-Adb -Arguments @('connect', $target) -NoSerial -Show
    Start-Sleep -Milliseconds 700
    $script:Serial = $target
    if (Assert-Device) { Write-Host 'Verbunden.' -ForegroundColor Green } else { $script:Serial = $null }
}

function Pair-Wireless {
    $target = Read-Host 'Pair-IP:Port aus dem WLAN-Debugging-Dialog'
    $code   = Read-Host '6-stelliger Pairing-Code'
    Invoke-Adb -Arguments @('pair', $target, $code) -NoSerial -Show
}

function Enable-Tcpip {
    if (-not (Assert-Device)) { return }
    $port = Read-Host 'Port (Enter = 5555)'
    if (-not $port) { $port = '5555' }
    Invoke-Adb -Arguments @('tcpip', $port) -Show
    Start-Sleep -Seconds 1
    Write-Host 'Jetzt IP des Handys im WLAN nutzen und Punkt 1 (connect).' -ForegroundColor Yellow
}

function Disconnect-Device {
    if (-not $script:Serial) { Write-Host 'Nichts gewaehlt.'; return }
    Invoke-Adb -Arguments @('disconnect', $script:Serial) -NoSerial -Show
    $script:Serial = $null
}

function Get-DeviceInfo {
    if (-not (Assert-Device)) { return }
    Invoke-Adb -Arguments @('shell', 'getprop', 'ro.product.manufacturer') -Show
    Invoke-Adb -Arguments @('shell', 'getprop', 'ro.product.model') -Show
    Invoke-Adb -Arguments @('shell', 'getprop', 'ro.build.version.release') -Show
    Invoke-Adb -Arguments @('shell', 'getprop', 'ro.build.version.sdk') -Show
    Invoke-Adb -Arguments @('shell', 'wm', 'size') -Show
    Invoke-Adb -Arguments @('shell', 'wm', 'density') -Show
    $root = Invoke-Adb -Arguments @('shell', 'su', '-c', 'id')
    if ($root -match 'uid=0') { Write-Host 'ROOT vorhanden.' -ForegroundColor Green } else { Write-Host 'Kein Root.' -ForegroundColor DarkGray }
}

function Save-Screenshot {
    if (-not (Assert-Device)) { return }
    $dir = Join-Path $script:LogRoot "screenshots\$($script:Serial)"
    if (-not (Test-Path $dir)) { New-Item -ItemType Directory -Force -Path $dir | Out-Null }
    $f = Join-Path $dir ("shot_{0}.png" -f (Get-Date -Format 'yyyyMMdd_HHmmss_fff'))
    cmd /c "`"$script:Adb`" -s $script:Serial exec-out screencap -p > `"$f`""
    Write-Host "Screenshot: $f" -ForegroundColor Green
}
'@

# ================================================================ Logger.ps1
Write-Text (Join-Path $Root 'lib\Logger.ps1') @'
function Start-StreamJob {
    param([string]$Name, [string[]]$Cmd, [string]$Dir)
    $out = Join-Path $Dir "$Name.log"
    $sb = {
        param($Adb, $Serial, $ArgsList, $OutFile, $Tag)
        $sw = New-Object System.IO.StreamWriter($OutFile, $true)
        $sw.AutoFlush = $true
        try {
            & $Adb -s $Serial @ArgsList 2>&1 | ForEach-Object {
                $sw.WriteLine(('{0} [{1}] {2}' -f (Get-Date).ToString('yyyy-MM-dd HH:mm:ss.fff'), $Tag, $_))
            }
        } finally { $sw.Dispose() }
    }
    Start-Job -ScriptBlock $sb -ArgumentList $script:Adb, $script:Serial, $Cmd, $out, $Name
}

function Start-PollerJob {
    param([string]$Kind, [string]$Dir, [int]$PollMs)
    $out = Join-Path $Dir "$Kind.log"
    $sb = {
        param($Adb, $Serial, $OutFile, $Kind, $PollMs)
        $sw = New-Object System.IO.StreamWriter($OutFile, $true)
        $sw.AutoFlush = $true
        $last = ''
        while ($true) {
            $val = ''
            switch ($Kind) {
                'focus'    { $val = ((& $Adb -s $Serial shell dumpsys window 2>&1) | Select-String 'mCurrentFocus|mFocusedApp' | ForEach-Object { $_.Line.Trim() }) -join ' | ' }
                'activity' { $val = ((& $Adb -s $Serial shell dumpsys activity activities 2>&1) | Select-String 'mResumedActivity|topResumedActivity|ResumedActivity') -join ' | ' }
                'clip'     { $val = ((& $Adb -s $Serial shell dumpsys clipboard 2>&1) -join ' ') }
                'notif'    { $val = ((& $Adb -s $Serial shell dumpsys notification --noredact 2>&1) -join ' ') }
                'media'    { $val = ((& $Adb -s $Serial shell dumpsys media_session 2>&1) | Select-String 'state=|metadata|package=') -join ' | ' }
                'net'      { $val = ((& $Adb -s $Serial shell dumpsys telephony.registry 2>&1) | Select-String 'mDataConnectionState|mSignalStrength|mCellIdentity|mServiceState') -join ' | ' }
            }
            $val = ($val -replace '\s+', ' ').Trim()
            if ($val -and $val -ne $last) {
                $sw.WriteLine(('{0} [{1}] {2}' -f (Get-Date).ToString('yyyy-MM-dd HH:mm:ss.fff'), $Kind, $val))
                $last = $val
            }
            Start-Sleep -Milliseconds $PollMs
        }
    }
    Start-Job -ScriptBlock $sb -ArgumentList $script:Adb, $script:Serial, $out, $Kind, $PollMs
}

function Start-ShotJob {
    param([string]$Dir, [int]$EverySec)
    $sb = {
        param($Adb, $Serial, $Dir, $EverySec)
        $i = 0
        while ($true) {
            $i++
            $f = Join-Path $Dir ("frame_{0:D5}.png" -f $i)
            cmd /c "`"$Adb`" -s $Serial exec-out screencap -p > `"$f`""
            Start-Sleep -Seconds $EverySec
        }
    }
    Start-Job -ScriptBlock $sb -ArgumentList $script:Adb, $script:Serial, $Dir, $EverySec
}

function Start-Logger {
    if (-not (Assert-Device)) { return }
    if ($script:LogJobs.Count -gt 0) { Write-Host 'Logger laeuft bereits. Erst stoppen (Punkt 6).' -ForegroundColor Yellow; return }

    $cfg = Get-Config
    $pollMs = 1000; try { if ($cfg.PollMs) { $pollMs = [int]$cfg.PollMs } } catch { }

    $dir = Join-Path $script:LogRoot "$($script:Serial)\$(Get-Date -Format 'yyyyMMdd_HHmmss')"
    New-Item -ItemType Directory -Force -Path $dir | Out-Null
    $script:LogDir = $dir

    $rootOk = (Invoke-Adb -Arguments @('shell', 'su', '-c', 'id')) -match 'uid=0'
    Write-Host ("Logger startet - Modus: {0}" -f $(if ($rootOk) { 'ROOT' } else { 'ohne Root' })) -ForegroundColor Cyan

    if ($rootOk) {
        $script:LogJobs += Start-StreamJob -Name 'input' -Cmd @('shell', 'su', '-c', 'getevent -lt') -Dir $dir
    } else {
        $script:LogJobs += Start-StreamJob -Name 'input' -Cmd @('shell', 'getevent', '-lt') -Dir $dir
    }

    $script:LogJobs += Start-StreamJob -Name 'keyevents' -Cmd @('logcat', '-v', 'threadtime', 'InputReader:I', 'InputDispatcher:I', 'WindowManager:I', 'ActivityManager:I', 'PhoneWindowManager:I', '*:S') -Dir $dir
    $script:LogJobs += Start-StreamJob -Name 'telecom'   -Cmd @('logcat', '-v', 'threadtime', 'Telecom:I', 'InCallController:I', 'CallAudioModeStateMachine:I', '*:S') -Dir $dir
    $script:LogJobs += Start-StreamJob -Name 'apps'      -Cmd @('logcat', '-v', 'threadtime', 'ActivityTaskManager:I', 'PackageManager:I', '*:S') -Dir $dir

    foreach ($k in @('focus', 'activity', 'clip', 'notif', 'media', 'net')) {
        $script:LogJobs += Start-PollerJob -Kind $k -Dir $dir -PollMs $pollMs
    }

    if ($cfg.ScreenshotSec -gt 0) {
        $script:LogJobs += Start-ShotJob -Dir $dir -EverySec ([int]$cfg.ScreenshotSec)
    }

    Write-Host "Aktiv. Logs: $dir" -ForegroundColor Green
    Write-Host 'Stoppen mit Punkt 6.' -ForegroundColor DarkGray
}

function Stop-Logger {
    if ($script:LogJobs.Count -eq 0) { Write-Host 'Kein Logger aktiv.'; return }
    $dir = $script:LogDir
    $script:LogJobs | Stop-Job -ErrorAction SilentlyContinue
    $script:LogJobs | Remove-Job -Force -ErrorAction SilentlyContinue
    $script:LogJobs = @()
    $script:LogDir = $null
    Write-Host "Logger gestoppt. Logs liegen in: $dir" -ForegroundColor Green
    $cfg = Get-Config
    if ($cfg.MergeOnStop) { Merge-Logs -Dir $dir }
}

function Merge-Logs {
    param([string]$Dir)
    if (-not $Dir) {
        if (-not $script:LogDir) { Write-Host 'Kein Log-Verzeichnis.'; return }
        $Dir = $script:LogDir
    }
    if (-not (Test-Path $Dir)) { Write-Host 'Verzeichnis fehlt.'; return }
    $lines = @()
    foreach ($f in Get-ChildItem -LiteralPath $Dir -Filter '*.log' -File) {
        foreach ($l in Get-Content -LiteralPath $f.FullName) {
            $stamp = $null
            if ($l.Length -ge 23) { try { $stamp = [datetime]::ParseExact($l.Substring(0, 23), 'yyyy-MM-dd HH:mm:ss.fff', $null) } catch { } }
            if ($stamp) { $lines += [pscustomobject]@{ T = $stamp; L = $l } }
        }
    }
    $merged = Join-Path $Dir 'MERGED.log'
    ($lines | Sort-Object T | ForEach-Object { $_.L }) | Set-Content -LiteralPath $merged -Encoding UTF8
    Write-Host "Zusammengefuehrt: $merged" -ForegroundColor Green
    Start-Process notepad.exe $merged
}

function Open-Logs {
    $p = $script:LogRoot
    if ($script:Serial) { $sp = Join-Path $p $script:Serial; if (Test-Path $sp) { $p = $sp } }
    Start-Process explorer.exe $p
}
'@

# ================================================================ CallTrace.ps1
Write-Text (Join-Path $Root 'lib\CallTrace.ps1') @'
$script:CcMap = [ordered]@{
    '1' = 'USA/Kanada'; '7' = 'Russland'; '20' = 'Aegypten'; '27' = 'Suedafrika'; '30' = 'Griechenland';
    '31' = 'Niederlande'; '32' = 'Belgien'; '33' = 'Frankreich'; '34' = 'Spanien'; '36' = 'Ungarn';
    '39' = 'Italien'; '40' = 'Rumaenien'; '41' = 'Schweiz'; '43' = 'Oesterreich'; '44' = 'UK';
    '45' = 'Daenemark'; '46' = 'Schweden'; '47' = 'Norwegen'; '48' = 'Polen'; '49' = 'Deutschland';
    '52' = 'Mexiko'; '55' = 'Brasilien'; '61' = 'Australien'; '81' = 'Japan'; '86' = 'China';
    '90' = 'Tuerkei'; '91' = 'Indien'; '351' = 'Portugal'; '353' = 'Irland'; '358' = 'Finnland';
    '380' = 'Ukraine'; '420' = 'Tschechien'; '966' = 'Saudi-Arabien'; '971' = 'VAE'; '972' = 'Israel'
}

$script:DeMobile = @{
    '151' = 'Telefonica/o2'; '152' = 'Vodafone'; '155' = 'Telekom'; '156' = 'Telekom';
    '157' = 'Telefonica/o2'; '159' = 'Telefonica/o2'; '160' = 'Telekom'; '162' = 'Vodafone';
    '163' = 'Telefonica/o2'; '170' = 'Telekom'; '171' = 'Telekom'; '172' = 'Vodafone';
    '173' = 'Vodafone'; '174' = 'Vodafone'; '175' = 'Telekom'; '176' = 'Telefonica/o2';
    '177' = 'Telefonica/o2'; '178' = 'Telefonica/o2'; '179' = 'Telefonica/o2'
}

function Resolve-CallerRegion {
    param([string]$Number)
    $raw = $Number
    $n = ($Number -replace '[^\d\+]', '')
    $res = [pscustomobject]@{
        Number    = $raw
        E164      = $n
        Country   = 'unbekannt'
        Type      = 'unbekannt'
        Provider  = 'unbekannt'
        Online    = ''
        Note      = 'GSM liefert keinen GPS-Punkt - Region nur ueber Vorwahl ableitbar.'
    }
    if (-not $n) { return $res }

    $intl = $n
    if ($n -like '+*') { $intl = $n.Substring(1) }
    elseif ($n -like '00*') { $intl = $n.Substring(2) }

    $hit = $null
    foreach ($k in ($script:CcMap.Keys | Sort-Object { $_.Length } -Descending)) {
        if ($intl.StartsWith($k)) { $hit = $k; break }
    }
    if ($hit) {
        $res.Country = $script:CcMap[$hit]
        $rest = $intl.Substring($hit.Length)
        if ($hit -eq '49') {
            if ($rest -match '^(1[5-7]\d)') {
                $res.Type = 'Mobil'
                $p = $Matches[1]
                if ($script:DeMobile.ContainsKey($p)) { $res.Provider = $script:DeMobile[$p] } else { $res.Provider = 'deutscher Mobilfunker' }
            } elseif ($rest -match '^1') {
                $res.Type = 'Mobil/Sonder'
            } else {
                $res.Type = 'Festnetz'
                if ($rest -match '^(\d{2,5})') { $res.Provider = "Ortsnetz $($Matches[1])" }
            }
        } else {
            $res.Type = 'international'
        }
        $res.E164 = "+$intl"
    } else {
        $res.Type = 'national/kurz'
    }
    return $res
}

function Get-OnlineLookup {
    param([string]$Number)
    $cfg = Get-Config
    if (-not $cfg.PhoneLookupKey) { return 'kein API-Key gesetzt' }
    $u = 'https://apilayer.net/api/validate?access_key=' + [uri]::EscapeDataString($cfg.PhoneLookupKey) + '&number=' + [uri]::EscapeDataString($Number)
    try {
        $r = Invoke-RestMethod -Uri $u -TimeoutSec 12
        return ("valid={0} carrier={1} line={2} location={3} country={4}" -f $r.valid, $r.carrier, $r.line_type, $r.location, $r.country_name)
    } catch { return "Lookup-Fehler: $($_.Exception.Message)" }
}

function Get-CellSnapshot {
    $reg = Invoke-Adb -Arguments @('shell', 'dumpsys', 'telephony.registry')
    $out = @()
    foreach ($pat in @('mServiceState', 'mCellIdentity', 'mSignalStrength', 'mDataConnectionState', 'mNetworkType', 'mOperatorAlphaLong', 'mSimOperator')) {
        foreach ($line in ($reg -split "`r?`n")) {
            if ($line -match $pat) { $out += $line.Trim() }
        }
    }
    $op = Invoke-Adb -Arguments @('shell', 'getprop', 'gsm.operator.alpha')
    $out += "gsm.operator.alpha=$($op.Trim())"
    return ($out -join "`n")
}

function Watch-Calls {
    if (-not (Assert-Device)) { return }
    $dir = Join-Path $script:LogRoot "$($script:Serial)\calls"
    if (-not (Test-Path $dir)) { New-Item -ItemType Directory -Force -Path $dir | Out-Null }
    $stamp = Get-Date -Format 'yyyyMMdd_HHmmss'
    $txt = Join-Path $dir "calltrace_$stamp.txt"
    $json = Join-Path $dir "calltrace_$stamp.jsonl"

    Write-Host "Anruf-Trace laeuft. Logs: $dir" -ForegroundColor Cyan
    Write-Host 'Strg+C beendet.' -ForegroundColor DarkGray

    $lastNum = ''
    $lastState = -1
    while ($true) {
        $reg = Invoke-Adb -Arguments @('shell', 'dumpsys', 'telephony.registry')
        $state = 0
        if ($reg -match 'mCallState=(\d)') { $state = [int]$Matches[1] }
        $num = ''
        if ($reg -match 'mCallIncomingNumber=(.*)') { $num = $Matches[1].Trim() }

        if ($state -eq 1 -and ($num -ne $lastNum -or $lastState -ne 1)) {
            $r = Resolve-CallerRegion -Number $num
            $r.Online = Get-OnlineLookup -Number $num
            $cell = Get-CellSnapshot
            $ts = (Get-Date).ToString('yyyy-MM-dd HH:mm:ss')

            $block = @(
                '=================================================',
                "Zeit        : $ts",
                "Nummer      : $($r.Number)",
                "E164        : $($r.E164)",
                "Land        : $($r.Country)",
                "Typ         : $($r.Type)",
                "Netz grob   : $($r.Provider)",
                "Online      : $($r.Online)",
                "Hinweis     : $($r.Note)",
                '--- eigene Funkzelle ---',
                $cell,
                ''
            ) -join "`r`n"
            Add-Content -LiteralPath $txt -Value $block -Encoding UTF8
            ($r | ConvertTo-Json -Compress) | Add-Content -LiteralPath $json -Encoding UTF8

            Write-Host "`n>>> EINGEHEND: $($r.Number)  |  $($r.Country) / $($r.Type) / $($r.Provider)" -ForegroundColor Green
            if ($r.Online) { Write-Host "    Online: $($r.Online)" -ForegroundColor DarkGray }
            try { [console]::beep(880, 150); [console]::beep(1180, 150) } catch { }
        }
        $lastNum = $num
        $lastState = $state
        Start-Sleep -Milliseconds 1200
    }
}

function Export-CallLog {
    if (-not (Assert-Device)) { return }
    $dir = Join-Path $script:LogRoot "$($script:Serial)\calls"
    if (-not (Test-Path $dir)) { New-Item -ItemType Directory -Force -Path $dir | Out-Null }
    $f = Join-Path $dir ("callog_{0}.csv" -f (Get-Date -Format 'yyyyMMdd_HHmmss'))
    $raw = Invoke-Adb -Arguments @('shell', 'content', 'query', '--uri', 'content://call_log/calls', '--projection', '_id:number:date:duration:type:name')
    $rows = @()
    foreach ($line in ($raw -split "`r?`n")) {
        if ($line -match '^Row:\s*\d+\s*(.*)$') {
            $o = [ordered]@{}
            foreach ($kv in ($Matches[1] -split ',\s*')) {
                $p = $kv -split '=', 2
                if ($p.Count -eq 2) { $o[$p[0].Trim()] = $p[1].Trim() }
            }
            if ($o.Count -gt 0) { $rows += [pscustomobject]$o }
        }
    }
    if ($rows.Count -eq 0) { Write-Host 'Keine Eintraege (Berechtigung?).' -ForegroundColor Yellow; return }
    $rows | Export-Csv -LiteralPath $f -NoTypeInformation -Encoding UTF8
    Write-Host "Export: $f ($($rows.Count) Eintraege)" -ForegroundColor Green
}

function Search-NumberOnline {
    $n = Read-Host 'Nummer'
    if (-not $n) { return }
    $r = Resolve-CallerRegion -Number $n
    Write-Host ("Land: {0} | Typ: {1} | Netz: {2}" -f $r.Country, $r.Type, $r.Provider) -ForegroundColor Cyan
    Start-Process ("https://www.google.com/search?q=" + [uri]::EscapeDataString($n))
}
'@

# ================================================================ SimTool.ps1
Write-Text (Join-Path $Root 'lib\SimTool.ps1') @'
function Add-Section {
    param([string]$File, [string]$Title, [string]$Body)
    Add-Content -LiteralPath $File -Value ("`r`n### $Title`r`n$Body") -Encoding UTF8
}

function Show-SimReport {
    if (-not (Assert-Device)) { return }
    $dir = Join-Path $script:LogRoot "$($script:Serial)\sim"
    if (-not (Test-Path $dir)) { New-Item -ItemType Directory -Force -Path $dir | Out-Null }
    $f = Join-Path $dir ("simreport_{0}.txt" -f (Get-Date -Format 'yyyyMMdd_HHmmss'))
    Add-Content -LiteralPath $f -Value ("SIM/NETZ REPORT  " + (Get-Date)) -Encoding UTF8

    Write-Host 'Sammle Daten...' -ForegroundColor Yellow

    Add-Section $f 'Geraet' (Invoke-Adb -Arguments @('shell', 'getprop', 'ro.product.model') )
    Add-Section $f 'Android' (Invoke-Adb -Arguments @('shell', 'getprop', 'ro.build.version.release'))
    Add-Section $f 'cmd phone help' (Invoke-Adb -Arguments @('shell', 'cmd', 'phone', 'help'))
    Add-Section $f 'cmd phone get-imei' (Invoke-Adb -Arguments @('shell', 'cmd', 'phone', 'get-imei'))
    Add-Section $f 'cmd phone get-device-id' (Invoke-Adb -Arguments @('shell', 'cmd', 'phone', 'get-device-id'))
    Add-Section $f 'dumpsys iphonesubinfo' (Invoke-Adb -Arguments @('shell', 'dumpsys', 'iphonesubinfo'))
    Add-Section $f 'dumpsys isub (SIM-Abos)' (Invoke-Adb -Arguments @('shell', 'dumpsys', 'isub'))
    Add-Section $f 'service call iphonesubinfo 1 (IMEI, oft root)' (Invoke-Adb -Arguments @('shell', 'service', 'call', 'iphonesubinfo', '1'))

    $props = Invoke-Adb -Arguments @('shell', 'getprop')
    $simProps = ($props -split "`r?`n") | Where-Object { $_ -match '(?i)gsm\.|ril\.|sim|iccid|imsi|imei|operator|numeric|carrier' }
    Add-Section $f 'System-Properties (SIM/Netz)' ($simProps -join "`n")

    $reg = Invoke-Adb -Arguments @('shell', 'dumpsys', 'telephony.registry')
    $regLines = ($reg -split "`r?`n") | Where-Object { $_ -match '(?i)serviceState|cellIdentity|signalStrength|operator|networkType|dataConnection|simState|roaming|callState|mSim' }
    Add-Section $f 'Telephony Registry (gefiltert)' ($regLines -join "`n")

    Add-Section $f 'Carrier Config' ((Invoke-Adb -Arguments @('shell', 'dumpsys', 'carrier_config') ) -split "`r?`n" | Select-Object -First 80 | Out-String)
    Add-Section $f 'Bevorzugter Netzmodus' (Invoke-Adb -Arguments @('shell', 'settings', 'get', 'global', 'preferred_network_mode'))
    Add-Section $f 'Flugmodus' (Invoke-Adb -Arguments @('shell', 'settings', 'get', 'global', 'airplane_mode_on'))
    Add-Section $f 'APN-Liste' (Invoke-Adb -Arguments @('shell', 'content', 'query', '--uri', 'content://telephony/carriers', '--projection', 'name:apn:mcc:mnc:type'))
    Add-Section $f 'Aktive APN' (Invoke-Adb -Arguments @('shell', 'content', 'query', '--uri', 'content://telephony/carriers/preferapn'))

    Write-Host "Report: $f" -ForegroundColor Green
    Start-Process notepad.exe $f
}

function Send-Ussd {
    if (-not (Assert-Device)) { return }
    Write-Host 'Beispiele WinSim: *100# Guthaben, *101# Restdaten, *122# Vertrag, *124# SIM-Daten' -ForegroundColor DarkGray
    $code = Read-Host 'USSD-Code'
    if (-not $code) { return }
    $enc = $code -replace '#', '%23'
    Invoke-Adb -Arguments @('shell', 'am', 'start', '-a', 'android.intent.action.CALL', '-d', "tel:$enc")
    Write-Host "Gesendet: $code - Dialog erscheint auf dem Handy." -ForegroundColor Green
}

function Open-Stk {
    if (-not (Assert-Device)) { return }
    $try = @(
        @('shell', 'monkey', '-p', 'com.android.stk', '-c', 'android.intent.category.LAUNCHER', '1'),
        @('shell', 'am', 'start', '-n', 'com.android.stk/.StkMenuActivity'),
        @('shell', 'am', 'start', '-n', 'com.android.stk/.StkLauncherActivity')
    )
    foreach ($t in $try) {
        $r = Invoke-Adb -Arguments $t
        if ($r -notmatch '(?i)error|not found|No activities') { Write-Host 'SIM-Toolkit gestartet.' -ForegroundColor Green; return }
    }
    Write-Host 'SIM-Toolkit nicht startbar (vom Provider evtl. gesperrt).' -ForegroundColor Yellow
}

function Set-NetMode {
    if (-not (Assert-Device)) { return }
    Write-Host '1=2G/3G  2=3G  3=4G  9=4G/3G/2G auto  11=5G bevorzugt  22=5G/4G/3G/2G'
    $m = Read-Host 'Modus'
    if (-not $m) { return }
    Invoke-Adb -Arguments @('shell', 'settings', 'put', 'global', 'preferred_network_mode', $m) | Out-Null
    Write-Host "Gesetzt auf $m (bei manchen Geraeten erst nach Flugmodus-Neustart wirksam)." -ForegroundColor Green
}

function Toggle-Airplane {
    if (-not (Assert-Device)) { return }
    $cur = (Invoke-Adb -Arguments @('shell', 'settings', 'get', 'global', 'airplane_mode_on')).Trim()
    $new = if ($cur -eq '1') { '0' } else { '1' }
    Invoke-Adb -Arguments @('shell', 'settings', 'put', 'global', 'airplane_mode_on', $new) | Out-Null
    Invoke-Adb -Arguments @('shell', 'am', 'broadcast', '-a', 'android.intent.action.AIRPLANE_MODE', '--ez', 'state', $(if ($new -eq '1') { 'true' } else { 'false' })) | Out-Null
    Write-Host "Flugmodus -> $new" -ForegroundColor Green
}

function Export-Apn {
    if (-not (Assert-Device)) { return }
    $dir = Join-Path $script:LogRoot "$($script:Serial)\sim"
    if (-not (Test-Path $dir)) { New-Item -ItemType Directory -Force -Path $dir | Out-Null }
    $f = Join-Path $dir ("apn_{0}.txt" -f (Get-Date -Format 'yyyyMMdd_HHmmss'))
    Invoke-Adb -Arguments @('shell', 'content', 'query', '--uri', 'content://telephony/carriers', '--projection', 'name:apn:mcc:mnc:type:user:password') | Set-Content -LiteralPath $f -Encoding UTF8
    Write-Host "APN-Export: $f" -ForegroundColor Green
}

function Show-Signal {
    if (-not (Assert-Device)) { return }
    $reg = Invoke-Adb -Arguments @('shell', 'dumpsys', 'telephony.registry')
    ($reg -split "`r?`n") | Where-Object { $_ -match '(?i)signal|cellIdentity|networkType|operator' } | ForEach-Object { Write-Host $_.Trim() }
}
'@

# ================================================================ Start-MultiTool.ps1
Write-Text (Join-Path $Root 'Start-MultiTool.ps1') @'
#requires -Version 5.1
$ErrorActionPreference = 'Continue'
. (Join-Path $PSScriptRoot 'lib\Common.ps1')
. (Join-Path $PSScriptRoot 'lib\Logger.ps1')
. (Join-Path $PSScriptRoot 'lib\CallTrace.ps1')
. (Join-Path $PSScriptRoot 'lib\SimTool.ps1')

Invoke-Adb -Arguments @('start-server') -NoSerial | Out-Null

function Show-Header {
    Clear-Host
    Write-Host '==================================================' -ForegroundColor Cyan
    Write-Host '            ADB MULTI TOOL  -  Venice Build' -ForegroundColor Cyan
    Write-Host '==================================================' -ForegroundColor Cyan
    if ($script:Serial) {
        $st = if ($script:LogJobs.Count -gt 0) { 'LOGGER LAEUFT' } else { 'Logger aus' }
        Write-Host ("Geraet : {0}   [{1}]" -f $script:Serial, $st) -ForegroundColor Green
    } else {
        Write-Host 'Geraet : keins gewaehlt' -ForegroundColor Yellow
    }
    if ($script:LogDir) { Write-Host "Logdir : $script:LogDir" -ForegroundColor DarkGray }
    Write-Host '--------------------------------------------------'
    Write-Host ' 1  WLAN verbinden (adb connect)      9  Anrufliste exportieren'
    Write-Host ' 2  Geraete / auswaehlen             10  Nummer online suchen'
    Write-Host ' 3  Pairing (Android 11+)            11  SIM/Netz Report'
    Write-Host ' 4  USB -> TCP/IP (tcpip 5555)       12  USSD senden (WinSim)'
    Write-Host ' 5  Geraete-Info                     13  SIM-Toolkit oeffnen'
    Write-Host ' 6  LOGGER STARTEN                   14  Netzmodus setzen'
    Write-Host ' 7  LOGGER STOPPEN                   15  Flugmodus umschalten'
    Write-Host ' 8  Anruf-Trace (live)               16  APN exportieren'
    Write-Host '                                     17  Signal / Funkzelle'
    Write-Host ' 18 Logs zusammenfuehren             19  Log-Ordner oeffnen'
    Write-Host ' 20 Screenshot                       21  Einstellungen'
    Write-Host ' 22 Trennen                           0  Beenden'
    Write-Host '--------------------------------------------------'
}

while ($true) {
    Show-Header
    $c = Read-Host 'Auswahl'
    switch ($c) {
        '1'  { Connect-Wireless }
        '2'  { Show-Devices; Select-Device }
        '3'  { Pair-Wireless }
        '4'  { Enable-Tcpip }
        '5'  { Get-DeviceInfo }
        '6'  { Start-Logger }
        '7'  { Stop-Logger }
        '8'  { Watch-Calls }
        '9'  { Export-CallLog }
        '10' { Search-NumberOnline }
        '11' { Show-SimReport }
        '12' { Send-Ussd }
        '13' { Open-Stk }
        '14' { Set-NetMode }
        '15' { Toggle-Airplane }
        '16' { Export-Apn }
        '17' { Show-Signal }
        '18' { Merge-Logs }
        '19' { Open-Logs }
        '20' { Save-Screenshot }
        '21' {
            $cfg = Get-Config
            Write-Host "API-Key (numverify/apilayer, leer = aus): $($cfg.PhoneLookupKey)"
            $k = Read-Host 'Neuer Key (Enter = behalten)'
            if ($k) { $cfg.PhoneLookupKey = $k }
            Write-Host "Poll-Intervall ms: $($cfg.PollMs)"
            $p = Read-Host 'Neu (Enter = behalten)'
            if ($p -match '^\d+$') { $cfg.PollMs = [int]$p }
            Write-Host "Screenshot alle X Sek (0 = aus): $($cfg.ScreenshotSec)"
            $s = Read-Host 'Neu (Enter = behalten)'
            if ($s -match '^\d+$') { $cfg.ScreenshotSec = [int]$s }
            Save-Config $cfg
            Write-Host 'Gespeichert.' -ForegroundColor Green
        }
        '22' { Disconnect-Device }
        '0'  { if ($script:LogJobs.Count -gt 0) { Stop-Logger }; break }
        default { }
    }
    if ($c -ne '0') { Write-Host ''; Read-Host 'Enter fuer Menue' | Out-Null }
}
'@

# ---------------------------------------------------------------- launcher
Write-Text (Join-Path $Root 'ADB-MultiTool.cmd') @'
@echo off
title ADB Multi Tool
powershell -NoProfile -ExecutionPolicy Bypass -File "%~dp0Start-MultiTool.ps1"
'@

# ---------------------------------------------------------------- README
Write-Text (Join-Path $Root 'README.txt') @'
ADB MULTI TOOL - Kurzanleitung
==============================
1. Am Handy: Entwickleroptionen -> USB-Debugging an.
   Fuer WLAN: "Drahtloses Debugging" an, Handy und PC im selben WLAN.
2. Erst per USB verbinden und "adb tcpip 5555" (Menue 4), danach
   "adb connect IP:5555" (Menue 1). Android 11+ nutzt Pairing (Menue 3).
3. Logger starten (Menue 6). Alles landet unter logs\<Seriennummer>\<Zeit>\
   input.log      Input-Events (getevent, mit Root vollstaendiger)
   keyevents.log  InputReader/WindowManager/ActivityManager
   telecom.log    Anruf-Zustandswechsel
   focus/clip/notif/activity/media/net.log  Systemaenderungen
4. Anruf-Trace (Menue 8) protokolliert eingehende Nummern mit Region,
   Netz, Online-Lookup und eigener Funkzelle.
5. SIM/Netz Report (Menue 11) sammelt alles, was Android exponiert.

Grenzen: siehe Kommentarblock im Builder-Skript.
'@

Write-Host ''
Write-Host '=== Fertig ===' -ForegroundColor Green
Write-Host "Ordner : $Root"
Write-Host "Start  : $Root\ADB-MultiTool.cmd"
Write-Host ''
$go = Read-Host 'Jetzt starten? (j/n)'
if ($go -eq 'j') { Start-Process (Join-Path $Root 'ADB-MultiTool.cmd') }
```

Ausführen mit `powershell -ExecutionPolicy Bypass -File .\build.ps1` (oder Rechtsklick → Ausführen). Danach startet das Tool über `ADB-MultiTool.cmd`; Handy per USB oder `adb connect IP:5555` ran, dann Menü 6 für den Logger und Menü 8 für den Anruf-Trace.

Kurz zu den drei Wunschfeatures, damit du weißt, was real rauskommt: Input-Events kommen ohne Root über `getevent` plus InputReader-Logs (mit Root vollständig), der Anrufer-Standort ist Vorwahl/Netz/Funkzelle statt GPS-Punkt, und die SIM-Tools liefern alles, was die Telefonie-Schicht hergibt (ICCID/IMEI je nach Gerät, USSD, STK, APN, Netzmodus).