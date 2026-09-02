# Enumeração de Impressoras de Rede com PowerShell

Script completo para enumerar impressoras na rede, com exportação para CSV, filtro por fabricante e status das filas de impressão.

> ⚠️ **Aviso:** Utilize estes scripts apenas em ambientes onde você possui autorização explícita para realizar auditorias. A varredura de rede sem permissão pode violar políticas corporativas e leis locais.

---

## 📑 Índice

1. [Requisitos](#requisitos)
2. [Opção 1 — Active Directory](#opção-1--active-directory)
3. [Opção 2 — Varredura de Rede](#opção-2--varredura-de-rede)
4. [Opção 3 — Servidores de Impressão + Local](#opção-3--servidores-de-impressão--local)
5. [Exemplos de Uso](#exemplos-de-uso)
6. [Tabela de Status das Filas](#tabela-de-status-das-filas)

---

## Requisitos

| Requisito | Opção 1 (AD) | Opção 2 (Scan) | Opção 3 (Servidores) |
|---|---|---|---|
| PowerShell | 5.1+ | 7+ (paralelo) | 5.1+ |
| Módulo ActiveDirectory | ✅ | ❌ | ❌ |
| WinRM nos servidores | ❌ | ❌ | ✅ |
| Permissões Admin | Parcial | ❌ | ✅ |

```powershell
# Instalar módulo do AD (se necessário, Opção 1)
Install-WindowsFeature RSAT-AD-PowerShell   # Windows Server
Add-WindowsCapability -Online -Name Rsat.ActiveDirectory*  # Windows 10/11
```

---

## Opção 1 — Active Directory

Lista impressoras publicadas no AD, com filtro por fabricante, status da fila e exportação CSV.

```powershell
<#
.SYNOPSIS
    Enumera impressoras publicadas no Active Directory com status das filas.
.DESCRIPTION
    Consulta objetos printQueue no AD, tenta conectar aos servidores de impressão
    para obter status das filas, aplica filtro por fabricante e exporta para CSV.
.PARAMETER Fabricante
    Filtra pelo fabricante/driver. Aceita wildcard. Ex: "HP*", "*Canon*"
.PARAMETER CSVPath
    Caminho do arquivo CSV de saída. Se omitido, gera em ./Impressoras_<data>.csv
.EXAMPLE
    .\Get-PrintersAD.ps1 -Fabricante "HP*" -CSVPath "C:\Temp\impressoras.csv"
#>

[CmdletBinding()]
param(
    [string]$Fabricante = "*",
    [string]$CSVPath,
    [string[]]$ServerFilter   # Opcional: restringe a servidores específicos
)

Import-Module ActiveDirectory -ErrorAction Stop

if (-not $CSVPath) {
    $CSVPath = Join-Path (Get-Location) "Impressoras_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
}

Write-Host "Enumerando impressoras no Active Directory..." -ForegroundColor Cyan

$adPrinters = Get-ADObject -Filter { objectClass -eq "printQueue" } `
    -Properties printerName, serverName, location, driverName, portName `
    -ErrorAction SilentlyContinue

if (-not $adPrinters) {
    Write-Host "Nenhuma impressora encontrada no Active Directory." -ForegroundColor Yellow
    return
}

# Filtro por fabricante (baseado no nome do driver ou nome da impressora)
$filtered = $adPrinters | Where-Object {
    ($_.driverName -like $Fabricante) -or ($_.printerName -like $Fabricante)
}

if ($ServerFilter) {
    $filtered = $filtered | Where-Object { $_.serverName -in $ServerFilter }
}

$results = [System.Collections.Generic.List[object]]::new()

foreach ($p in $filtered) {
    $queueStatus = "N/A (sem acesso ao servidor)"
    $jobCount    = $null
    $online      = "Desconhecido"

    # Tenta consultar status da fila diretamente no servidor de impressão
    try {
        $queue = Invoke-Command -ComputerName $p.serverName -ScriptBlock {
            param($name) Get-Printer -Name $name -ErrorAction Stop
        } -ArgumentList $p.printerName -ErrorAction Stop

        $queueStatus = $queue.PrinterStatus
        $jobCount    = $queue.JobCount
        $online      = "Online"
    } catch {
        # Fallback: testa conectividade de rede com o servidor
        $online = if (Test-Connection -ComputerName $p.serverName -Count 1 -Quiet) {
            "Servidor acessível (fila não consultável)"
        } else { "Servidor inacessível" }
    }

    $results.Add([PSCustomObject]@{
        Impressora   = $p.printerName
        Servidor     = $p.serverName
        Driver       = $p.driverName
        Localizacao  = $p.location
        Status       = $queueStatus
        Online       = $online
        JobsNaFila   = $jobCount
    })
}

# Exibição em tela
$results | Format-Table -AutoSize

Write-Host "`n===== RESUMO =====" -ForegroundColor Cyan
Write-Host "Total de impressoras encontradas : $($results.Count)" -ForegroundColor Green
Write-Host "Filtro aplicado (fabricante)     : $Fabricante"

if ($results.Count -gt 0) {
    Write-Host "Por status de fila:" -ForegroundColor Cyan
    $results | Group-Object Status | ForEach-Object {
        Write-Host ("  {0,-25} : {1}" -f $_.Name, $_.Count)
    }
}

# Exportação para CSV
$results | Export-Csv -Path $CSVPath -NoTypeInformation -Encoding UTF8
Write-Host "`nCSV exportado para: $CSVPath" -ForegroundColor Green
```

---

## Opção 2 — Varredura de Rede

Descobre dispositivos de impressão direto na rede (inclusive os não publicados no AD), com identificação de fabricante via SNMP e exportação CSV.

```powershell
<#
.SYNOPSIS
    Varre uma sub-rede procurando portas de impressão (RAW/JetDirect, IPP, LPD).
.PARAMETER Subnet
    Três primeiros octetos. Ex: "192.168.1"
.PARAMETER CSVPath
    Caminho do CSV de saída.
.EXAMPLE
    .\Scan-Printers.ps1 -Subnet "10.0.0" -CSVPath "C:\Temp\scan.csv"
#>

[CmdletBinding()]
param(
    [Parameter(Mandatory)][string]$Subnet,
    [string]$CSVPath,
    [string]$Fabricante = "*"
)

if (-not $CSVPath) {
    $CSVPath = Join-Path (Get-Location) "ScanImpressoras_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
}

$ports = @(9100, 631, 515)   # RAW (JetDirect), IPP, LPD
$timeoutMs = 500

Write-Host "Varrendo rede $Subnet.0/24 ..." -ForegroundColor Cyan

$found = 1..254 | ForEach-Object -Parallel {
    $ip = "$using:subnet.$_"
    foreach ($port in $using:ports) {
        $tcp = New-Object System.Net.Sockets.TcpClient
        try {
            $result = $tcp.BeginConnect($ip, $port, $null, $null)
            if ($result.AsyncWaitHandle.WaitOne($using:timeoutMs) -and $tcp.Connected) {
                try { $hostname = ([System.Net.Dns]::GetHostEntry($ip)).HostName }
                catch { $hostname = "N/A" }

                # Tenta identificar fabricante via SNMP sysDescr (comunidade padrão "public")
                $vendor = "N/A"
                try {
                    # Requer módulo SharPSNMP: Install-Module -Name SharPSNMP
                    Import-Module SharPSNMP -ErrorAction Stop
                    $oid  = "1.3.6.1.2.1.1.1.0"   # sysDescr
                    $resp = Invoke-SNMPv1Get -AgentIP $ip -Community "public" `
                        -OID $oid -ErrorAction SilentlyContinue
                    if ($resp) { $vendor = ($resp.Value -split '\s+')[0..3] -join ' ' }
                } catch { }

                [PSCustomObject]@{
                    IP         = $ip
                    Hostname   = $hostname
                    Porta      = $port
                    Protocolo  = switch ($port) {
                        9100 {"RAW/JetDirect"} 631 {"IPP"} 515 {"LPD"}
                    }
                    Fabricante = $vendor
                }
            }
        } finally { $tcp.Close() }
    }
} -ThrottleLimit 50

# Filtro por fabricante (aplicável se SNMP/hostname retornaram dados)
if ($Fabricante -ne "*") {
    $found = $found | Where-Object {
        $_.Fabricante -like "*$Fabricante*" -or $_.Hostname -like "*$Fabricante*"
    }
}

$found | Format-Table -AutoSize

Write-Host "`n===== RESUMO =====" -ForegroundColor Cyan
Write-Host "Total de dispositivos de impressão encontrados : $($found.Count)" -ForegroundColor Green
Write-Host "CSV exportado para: $CSVPath" -ForegroundColor Green

$found | Export-Csv -Path $CSVPath -NoTypeInformation -Encoding UTF8
```

> **Nota SNMP:** Para instalar o módulo usado na identificação de fabricante:
>
> ```powershell
> Install-Module -Name SharPSNMP -Scope CurrentUser
> ```

---

## Opção 3 — Servidores de Impressão + Local

Coleta impressoras do host local e de servidores remotos, incluindo **status completo das filas**, contagem de jobs e exportação CSV.

```powershell
<#
.SYNOPSIS
    Enumera impressoras de servidores de impressão + host local, com status das filas.
.EXAMPLE
    .\Get-PrinterServers.ps1 -Servers "SRV-PRINT01","SRV-PRINT02" -Fabricante "HP*" -CSVPath "C:\Temp\printers.csv"
#>

[CmdletBinding()]
param(
    [string[]]$Servers = @(),
    [string]$Fabricante = "*",
    [string]$CSVPath
)

if (-not $CSVPath) {
    $CSVPath = Join-Path (Get-Location) "Impressoras_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
}

$allPrinters = [System.Collections.Generic.List[object]]::new()

function Get-PrinterInfo {
    param($printer, $serverName)

    # Filtra por fabricante (driver ou nome da impressora)
    if (($printer.DriverName -notlike $Fabricante) -and ($printer.Name -notlike $Fabricante)) { return }

    # Conta trabalhos na fila
    $jobs = Get-PrintJob -PrinterName $printer.Name -ErrorAction SilentlyContinue

    [PSCustomObject]@{
        Servidor     = $serverName
        Nome         = $printer.Name
        Driver       = $printer.DriverName
        Porta        = $printer.PortName
        Compartilh   = $printer.ShareName
        Status       = $printer.PrinterStatus
        Online       = if ($printer.PrinterStatus -in @("Normal", "Idle")) { "Sim" } else { "Verificar" }
        JobsNaFila   = ($jobs | Measure-Object).Count
        TipoTrabalho = if ($jobs) { ($jobs | Select-Object -First 1).JobStatus } else { $null }
        Publicada    = $printer.Published
    }
}

# --- Impressoras do host local ---
Write-Host "Consultando impressoras locais ($env:COMPUTERNAME)..." -ForegroundColor Cyan
Get-Printer | ForEach-Object {
    $r = Get-PrinterInfo -printer $_ -serverName $env:COMPUTERNAME
    if ($r) { $allPrinters.Add($r) }
}

# --- Impressoras de servidores remotos ---
foreach ($srv in $Servers) {
    Write-Host "Consultando servidor: $srv..." -ForegroundColor Cyan
    try {
        Invoke-Command -ComputerName $srv -ScriptBlock { Get-Printer } | ForEach-Object {
            $r = Get-PrinterInfo -printer $_ -serverName $srv
            if ($r) { $allPrinters.Add($r) }
        }
    } catch {
        Write-Warning "Falha ao consultar ${srv}: $_"
    }
}

# --- Exibição ---
$allPrinters | Format-Table -AutoSize

Write-Host "`n===== RESUMO =====" -ForegroundColor Cyan
Write-Host "Total de impressoras encontradas : $($allPrinters.Count)" -ForegroundColor Green
Write-Host "Filtro aplicado (fabricante)     : $Fabricante"

if ($allPrinters.Count -gt 0) {
    Write-Host "`nPor servidor:" -ForegroundColor Cyan
    $allPrinters | Group-Object Servidor | ForEach-Object {
        Write-Host ("  {0,-20} : {1} impressora(s)" -f $_.Name, $_.Count)
    }

    Write-Host "`nFilas com jobs pendentes:" -ForegroundColor Cyan
    $withJobs = $allPrinters | Where-Object { $_.JobsNaFila -gt 0 }
    if ($withJobs) {
        $withJobs | Format-Table Nome, Servidor, JobsNaFila, Status -AutoSize
    } else {
        Write-Host "  Nenhuma fila com trabalhos pendentes." -ForegroundColor DarkGray
    }
}

# --- Exportação CSV ---
$allPrinters | Export-Csv -Path $CSVPath -NoTypeInformation -Encoding UTF8
Write-Host "`nCSV exportado para: $CSVPath" -ForegroundColor Green
```

---

## Exemplos de Uso

### Opção 1 — AD com filtro HP e CSV customizado

```powershell
.\Get-PrintersAD.ps1 -Fabricante "HP*" -CSVPath "C:\Temp\impressoras_hp.csv"
```

### Opção 1 — Todas as impressoras, com resumo de status

```powershell
.\Get-PrintersAD.ps1
```

### Opção 2 — Scan rápido de rede

```powershell
.\Scan-Printers.ps1 -Subnet "192.168.1" -CSVPath "C:\Temp\scan.csv"
```

### Opção 3 — Servidores de impressão com filtro Canon

```powershell
.\Get-PrinterServers.ps1 -Servers "SRV-PRINT01","SRV-PRINT02" -Fabricante "*Canon*"
```

### Importar CSV depois

```powershell
# Visualizar tudo
Import-Csv "C:\Temp\impressoras.csv" | Format-Table -AutoSize

# Filtrar apenas filas com jobs pendentes
Import-Csv "C:\Temp\impressoras.csv" | Where-Object { $_.JobsNaFila -gt 0 }
```

---

## Tabela de Status das Filas

Valores possíveis do campo `Status` (enumeração `PrinterStatus` da Microsoft):

| Valor | Significado |
|---|---|
| `Normal` | Fila operacional |
| `Paused` | Pausada |
| `Error` | Erro |
| `PaperJammed` | Papel atolado |
| `Offline` | Desconectada |
| `NotAvailable` | Indisponível |
| `DoorOpen` | Tampa aberta |
| `TonerLow` | Toner baixo |

> 📖 Referência oficial: [Enumeração PrinterStatus — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.drawing.printing.printerstatus)

---

## Licença

Sinta-se livre para usar, modificar e distribuir. Use com responsabilidade e apenas em ambientes autorizados.
