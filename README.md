# DISA STIG ID: WN10-UR-000090

## Synopsis
This PowerShell script ensures that the account lockout duration is set to 15 minutes.

## Notes
- **Author**: Dion Alexander
- **LinkedIn**: https://www.linkedin.com/in/infosecdion
- **GitHub**: https://github.com/InfoSecDion
- **Date Created**: 2025-11-20
- **Last Modified**: 2025-11-20
- **Version**: 1.0
- **CVEs**: N/A
- **Plugin IDs**: N/A
- **STIG-ID**: WN10-AC-000005
  
## Tested On
- **Date(s) Tested**: 
- **Tested By**: 
- **Systems Tested**: 
- **PowerShell Ver.**: 

## Usage
Put any usage instructions here.

Example syntax:
```powershell
# STIG: Deny log on through Remote Desktop Services
# This script configures the SeDenyRemoteInteractiveLogonRight user right
# to include the specified accounts/groups.

# Requires administrative privileges
if (-not ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()
    ).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {

    Write-Host "This script must be run as Administrator." -ForegroundColor Red
    exit 1
}

# Logical user right name used in security templates
$RightName = "SeDenyRemoteInteractiveLogonRight"

# Groups/accounts to apply (edit as needed)
$DomainGroups     = @("Enterprise Admins", "Domain Admins", "Local account")
$AllSystemsGroups = @("Guests")

$AssignedGroups = $DomainGroups + $AllSystemsGroups

# Security template format uses commas, spaces replaced by underscores
$RightValue = ($AssignedGroups -join ",").Replace(" ", "_")

# Temp paths
$tempDir    = "C:\Temp"
$configPath = Join-Path $tempDir "secpol.cfg"
$logPath    = Join-Path $tempDir "secpol.log"

if (-not (Test-Path $tempDir)) {
    New-Item -ItemType Directory -Path $tempDir -Force | Out-Null
}

Write-Host "Exporting current security policy to $configPath..." -ForegroundColor Cyan
secedit /export /cfg $configPath /log $logPath | Out-Null

if (-not (Test-Path $configPath)) {
    Write-Error "Failed to export security policy. Check permissions."
    exit 1
}

# Read and adjust the template
Write-Host "Updating $RightName in security template..." -ForegroundColor Cyan
$config = Get-Content $configPath

if ($config -notmatch "^$RightName") {
    # Add line if it does not exist
    $config += "$RightName = $RightValue"
}
else {
    $config = $config -replace "^(?$RightName\s*=\s*).*$", "$RightName = $RightValue"
}

$config | Set-Content $configPath -Encoding Unicode

# Apply updated policy (USER_RIGHTS only)
Write-Host "Applying updated user rights via secedit..." -ForegroundColor Cyan
secedit /configure /db "$env:WINDIR\security\Database\secedit.sdb" /cfg $configPath /areas USER_RIGHTS /log $logPath /quiet

if ($LASTEXITCODE -ne 0) {
    Write-Error "secedit returned exit code $LASTEXITCODE. See $logPath for details."
    exit 1
}

# Cleanup
Remove-Item $configPath -ErrorAction SilentlyContinue
Remove-Item $logPath    -ErrorAction SilentlyContinue

Write-Host "Policy '$RightName' has been configured successfully." -ForegroundColor Green
Write-Host "Run 'gpupdate /force' or reboot to ensure the change is fully applied."
```
