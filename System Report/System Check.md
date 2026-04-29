PowerShell is essentially a Swiss Army knife for Windows, you can hook into almost any part of the operating system.
## CPU and Performance

| Metric                              | Command                                                                                             |
| ----------------------------------- | --------------------------------------------------------------------------------------------------- |
| Total vs. Visible Free Memory.      | `Get-CimInstance Win32_OperatingSystem \| Select-Object TotalVisibleMemorySize, FreePhysicalMemory` |
| Average CPU Load Percentage.        | `Get-CimInstance Win32_Processor \| Select-Object LoadPercentage`                                   |
| System Uptime                       | `(Get-CimInstance Win32_OperatingSystem).LastBootUpTime`                                            |
| Interface Alias, Status, and Speed. | `Get-NetAdapter \| Select-Object Name, Status, LinkSpeed`                                           |
| Windows Updates                     | `Get-HotFix \| Sort-Object InstalledOn -Descending \| Select-Object -First 1`                       |
### Q&A
- **Why `(Get-CimInstance Win32_OperatingSystem).LastBootUpTime` show Strange time such as days ago?** 
	- ==Answer== : Even though you are clicking Shut down every night, Windows isn't actually performing a cold boot when you turn it back on. This is a feature designed for fast startup. Windows closes your apps then saves their core  state to a file on your hard drive and then turns off the power. Because the kernel never actually stopped running, The Last Bootup Time counter doesn't reset.
## Pattern
Pattern: **Collect** → **Format** → **Report**.

| Category | WMI/CIM Class               | Use Case                                 |
| -------- | --------------------------- | ---------------------------------------- |
| Hardware | `Win32_PhysicalMemory`      | Identifying if a RAM stick has failed.   |
| Battery  | `Win32_Battery`             | Checking health/charge on laptops.       |
| BIOS     | `Win32_BIOS`                | Checking for outdated firmware versions. |
| Security | `Win32_QuickFixEngineering` | Ensuring security patches are present.   |

## Example

### System Check

This one generate HTML report.

```powershell
# 1. Gather System Uptime & OS Info
$OS = Get-CimInstance Win32_OperatingSystem
$Uptime = (Get-Date) - $OS.LastBootUpTime
$UptimeString = "$($Uptime.Days) Days, $($Uptime.Hours) Hours, $($Uptime.Minutes) Minutes"

# 2. Gather Hardware (CPU & RAM)
$CPU = Get-CimInstance Win32_Processor | Select-Object Name, NumberOfCores, NumberOfLogicalProcessors | Select-Object -First 1
$RAM = Get-CimInstance Win32_OperatingSystem | Select-Object `
    @{Name="Total RAM (GB)"; Expression={[math]::Round($_.TotalVisibleMemorySize / 1MB, 2)}},
    @{Name="Free RAM (GB)"; Expression={[math]::Round($_.FreePhysicalMemory / 1MB, 2)}}

$Hardware = [PSCustomObject]@{
    "Processor" = $CPU.Name
    "Cores/Threads" = "$($CPU.NumberOfCores) / $($CPU.NumberOfLogicalProcessors)"
    "Total RAM (GB)" = $RAM.'Total RAM (GB)'
    "Free RAM (GB)" = $RAM.'Free RAM (GB)'
}

# 3. Gather BIOS Info
$BIOS = Get-CimInstance Win32_BIOS | Select-Object Manufacturer, Name, Version

# 4. Gather Battery Info
$Battery = Get-CimInstance Win32_Battery -ErrorAction SilentlyContinue | Select-Object Name, EstimatedChargeRemaining, BatteryStatus

# 5. Gather Security (Last 5 Updates)
$Security = Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 5 HotFixID, Description, InstalledOn

# 6. Gather Disk Space
$DiskSpace = Get-CimInstance Win32_LogicalDisk -Filter "DriveType=3" | Select-Object DeviceID, 
    @{Name="Total(GB)"; Expression={[math]::Round($_.Size / 1GB, 2)}},
    @{Name="Free(GB)";  Expression={[math]::Round($_.FreeSpace / 1GB, 2)}},
    @{Name="Used(%)";   Expression={[math]::Round((1 - ($_.FreeSpace / $_.Size)) * 100, 1)}}

# 7. Gather Stopped Services
$StoppedServices = Get-Service | Where-Object { $_.Status -eq "Stopped" -and $_.StartType -eq "Automatic" } | 
    Select-Object Name, DisplayName

# 8. Define HTML
$Header = @"
<html>
<head>
    <style>
        body { font-family: 'Segoe UI', Arial, sans-serif; margin: 40px; background-color: #f4f7f6; color: #333; }
        table { border-collapse: collapse; width: 100%; background: white; margin-bottom: 30px; border-radius: 8px; overflow: hidden; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        th, td { padding: 10px 15px; text-align: left; border-bottom: 1px solid #eee; }
        th { background-color: #0078d4; color: white; text-transform: uppercase; font-size: 13px; }
        tr:hover { background-color: #f9f9f9; }
        h2 { color: #004578; border-left: 5px solid #0078d4; padding-left: 15px; margin-top: 30px; }
        .success { color: #28a745; font-weight: bold; padding: 10px; background: #e6ffed; border: 1px solid #b7eb8f; }
        .info-box { background: #e1f0fa; padding: 15px; border-left: 5px solid #0078d4; margin-bottom: 20px; }
    </style>
</head>
<body>
    <h1>System Health Dashboard</h1>
    <div class="info-box"><strong>Current Uptime:</strong> $UptimeString</div>
"@

# 9. Generate Fragments
$HardwareHtml = $Hardware | ConvertTo-Html -Fragment
$BiosHtml     = $BIOS | ConvertTo-Html -Fragment
$DiskHtml     = $DiskSpace | ConvertTo-Html -Fragment
$BatteryHtml  = if ($Battery) { $Battery | ConvertTo-Html -Fragment } else { "<p>No battery detected.</p>" }
$SecurityHtml = if ($Security) { $Security | ConvertTo-Html -Fragment } else { "<p>No recent updates found.</p>" }
$ServiceHtml  = if ($StoppedServices) { $StoppedServices | ConvertTo-Html -Fragment } else { "<p class='success'>OK: All automatic services are running.</p>" }

# 10. Final Assembly
$FinalHtml = $Header + 
    "<h2>Hardware Overview</h2>" + $HardwareHtml + 
    "<h2>BIOS Information</h2>" + $BiosHtml + 
    "<h2>Battery Status</h2>" + $BatteryHtml + 
    "<h2>Storage Overview</h2>" + $DiskHtml + 
    "<h2>Recent Security Patches</h2>" + $SecurityHtml + 
    "<h2>Stopped Automatic Services</h2>" + $ServiceHtml + 
    "</body></html>"

$ReportPath = "$env:USERPROFILE\Desktop\SystemReport.html"

$FinalHtml | Out-File $ReportPath -Encoding utf8
Invoke-Item $ReportPath
```
