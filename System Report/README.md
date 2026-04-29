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
