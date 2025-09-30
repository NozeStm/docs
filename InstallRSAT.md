# Installer les fonctionnalités RSAT en PowerShell
1. Visualiser les fonctionnalités installées
```powershell
Get-WindowsCapability -Name "RSAT*" -Online | Select-Object -Property DisplayName, State
```
2. Visualiser les noms techniques des fonctionalités RSAT
```powershell
Get-WindowsCapability -Name "RSAT*" -Online | Select-Object -Property Name, DisplayName
```
