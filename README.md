## 📊 Teams Governance Dashboard

### 📌 Problem It Solves
Organizations often lose track of orphaned Teams, excessive guest users, and inactive channels. This toolkit audits your entire Teams environment and generates a compliance report.

### 🛠️ Technologies Used
- Microsoft Graph PowerShell SDK
- Microsoft Teams Admin
- Teams Governance & Compliance

### 📂 Repository Structure
```
Teams-Governance-Dashboard/
├── README.md
├── scripts/
│   ├── 01-GetAllTeams.ps1
│   ├── 02-AuditGuestUsers.ps1
│   ├── 03-FindOrphanedTeams.ps1
│   └── 04-GenerateGovernanceReport.ps1
└── reports/
    └── (generated reports appear here)
```

### ⚙️ Prerequisites
```powershell
# Install required modules
Install-Module Microsoft.Graph -Scope CurrentUser -Force
Install-Module MicrosoftTeams -Scope CurrentUser -Force

# Connect
Connect-MgGraph -Scopes "Group.Read.All", "User.Read.All", "TeamMember.Read.All"
Connect-MicrosoftTeams
```

### 📄 Script 1 — Get All Teams Inventory
```powershell
# 01-GetAllTeams.ps1
# Description: Gets a full inventory of all Teams in the tenant

Connect-MgGraph -Scopes "Group.Read.All", "TeamMember.Read.All"

$AllTeams = Get-MgGroup -Filter "resourceProvisioningOptions/Any(x:x eq 'Team')" -All `
    -Property "Id,DisplayName,Description,CreatedDateTime,Visibility,Mail"

$TeamInventory = @()

foreach ($Team in $AllTeams) {
    # Get member count
    $Members = Get-MgGroupMember -GroupId $Team.Id -All
    $Owners  = Get-MgGroupOwner  -GroupId $Team.Id -All

    $TeamInventory += [PSCustomObject]@{
        TeamName        = $Team.DisplayName
        TeamId          = $Team.Id
        Visibility      = $Team.Visibility
        MemberCount     = $Members.Count
        OwnerCount      = $Owners.Count
        CreatedDate     = $Team.CreatedDateTime
        Email           = $Team.Mail
    }
}

$TeamInventory | Export-Csv -Path ".\reports\Teams-Inventory.csv" -NoTypeInformation
Write-Host "✅ Teams inventory exported: $($TeamInventory.Count) teams found" -ForegroundColor Green
```

### 📄 Script 2 — Audit Guest Users in Teams
```powershell
# 02-AuditGuestUsers.ps1
# Description: Reports all guest users across all Teams

Connect-MgGraph -Scopes "Group.Read.All", "User.Read.All"

$AllTeams = Get-MgGroup -Filter "resourceProvisioningOptions/Any(x:x eq 'Team')" -All
$GuestReport = @()

foreach ($Team in $AllTeams) {
    $Members = Get-MgGroupMember -GroupId $Team.Id -All

    foreach ($Member in $Members) {
        $User = Get-MgUser -UserId $Member.Id -Property "DisplayName,UserPrincipalName,UserType,Mail" -ErrorAction SilentlyContinue

        if ($User.UserType -eq "Guest") {
            $GuestReport += [PSCustomObject]@{
                TeamName          = $Team.DisplayName
                GuestName         = $User.DisplayName
                GuestEmail        = $User.Mail
                GuestUPN          = $User.UserPrincipalName
            }
        }
    }
}

$GuestReport | Export-Csv -Path ".\reports\Teams-GuestUsers.csv" -NoTypeInformation
Write-Host "✅ Found $($GuestReport.Count) guest users across all Teams" -ForegroundColor Yellow
```

### 📄 Script 3 — Find Orphaned Teams (No Owners)
```powershell
# 03-FindOrphanedTeams.ps1
# Description: Identifies Teams that have no owners (orphaned)

Connect-MgGraph -Scopes "Group.Read.All"

$AllTeams = Get-MgGroup -Filter "resourceProvisioningOptions/Any(x:x eq 'Team')" -All
$OrphanedTeams = @()

foreach ($Team in $AllTeams) {
    $Owners = Get-MgGroupOwner -GroupId $Team.Id -All

    if ($Owners.Count -eq 0) {
        $OrphanedTeams += [PSCustomObject]@{
            TeamName    = $Team.DisplayName
            TeamId      = $Team.Id
            CreatedDate = $Team.CreatedDateTime
            Status      = "ORPHANED - No Owners"
        }
        Write-Host "⚠️  Orphaned Team: $($Team.DisplayName)" -ForegroundColor Red
    }
}

$OrphanedTeams | Export-Csv -Path ".\reports\Teams-Orphaned.csv" -NoTypeInformation
Write-Host "`n📊 Found $($OrphanedTeams.Count) orphaned Teams" -ForegroundColor Cyan
```

### 📄 Script 4 — Full Governance Report
```powershell
# 04-GenerateGovernanceReport.ps1
# Description: Generates a consolidated HTML governance report for all Teams

Connect-MgGraph -Scopes "Group.Read.All", "User.Read.All"

$AllTeams = Get-MgGroup -Filter "resourceProvisioningOptions/Any(x:x eq 'Team')" -All
$ReportData = @()

foreach ($Team in $AllTeams) {
    $Members = Get-MgGroupMember -GroupId $Team.Id -All
    $Owners  = Get-MgGroupOwner  -GroupId $Team.Id -All
    $Guests  = $Members | Where-Object { $_.AdditionalProperties["userType"] -eq "Guest" }

    $ReportData += [PSCustomObject]@{
        TeamName    = $Team.DisplayName
        Visibility  = $Team.Visibility
        Members     = $Members.Count
        Owners      = $Owners.Count
        Guests      = $Guests.Count
        IsOrphaned  = ($Owners.Count -eq 0)
        HasGuests   = ($Guests.Count -gt 0)
        CreatedDate = $Team.CreatedDateTime
    }
}

# Generate HTML Report
$HTML = @"
<!DOCTYPE html>
<html>
<head><title>Teams Governance Report</title>
<style>
  body { font-family: Arial; padding: 20px; }
  table { border-collapse: collapse; width: 100%; }
  th { background-color: #0078d4; color: white; padding: 10px; }
  td { border: 1px solid #ddd; padding: 8px; }
  tr:nth-child(even) { background-color: #f2f2f2; }
  .orphaned { background-color: #ffcccc !important; }
</style></head>
<body>
<h1>Microsoft Teams Governance Report</h1>
<p>Generated: $(Get-Date -Format 'yyyy-MM-dd HH:mm')</p>
<p>Total Teams: $($ReportData.Count)</p>
<table>
<tr><th>Team Name</th><th>Visibility</th><th>Members</th><th>Owners</th><th>Guests</th><th>Orphaned?</th></tr>
"@

foreach ($Row in $ReportData) {
    $CssClass = if ($Row.IsOrphaned) { ' class="orphaned"' } else { '' }
    $HTML += "<tr$CssClass><td>$($Row.TeamName)</td><td>$($Row.Visibility)</td><td>$($Row.Members)</td><td>$($Row.Owners)</td><td>$($Row.Guests)</td><td>$($Row.IsOrphaned)</td></tr>`n"
}

$HTML += "</table></body></html>"
$HTML | Out-File -FilePath ".\reports\Teams-GovernanceReport.html" -Encoding UTF8

Write-Host "✅ HTML Report saved to .\reports\Teams-GovernanceReport.html" -ForegroundColor Green
```
