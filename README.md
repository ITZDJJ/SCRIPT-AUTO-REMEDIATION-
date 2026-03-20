#$PVWA = "https://EC2AMAZ-L6EO09G.Cyberarkdj.com/PasswordVault"
$User = "Administrator"
$Safe = "ApikeyDevops"

# Login
$Password = Read-Host "Enter CyberArk Password" -AsSecureString
$bstr = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($Password)
$Plain = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($bstr)

$body = @{
    username = $User
    password = $Plain
} | ConvertTo-Json

$token = Invoke-RestMethod -Method Post `
    -Uri "$PVWA/API/Auth/CyberArk/Logon" `
    -Body $body `
    -ContentType "application/json"

# Get accounts
$accounts = Invoke-RestMethod -Method Get `
    -Uri "$PVWA/API/Accounts?filter=safeName eq $Safe" `
    -Headers @{ Authorization = $token }

$fixed = 0
$skipped = 0

foreach ($acct in $accounts.value) {

    $needsFix = $false

    if ($acct.secretManagement.automaticManagementEnabled -ne $true) {
        $needsFix = $true
    }

    if ($acct.userName -notmatch '^api_') {
        $needsFix = $true
    }

    if ($needsFix) {
        Write-Host "🔧 Fixing: $($acct.userName)"

        $updateBody = @{
            secretManagement = @{
                automaticManagementEnabled = $true
            }
        } | ConvertTo-Json -Depth 5

        try {
            Invoke-RestMethod -Method Patch `
                -Uri "$PVWA/API/Accounts/$($acct.id)" `
                -Headers @{ Authorization = $token } `
                -Body $updateBody `
                -ContentType "application/json"

            Write-Host "✅ Fixed: $($acct.userName)" -ForegroundColor Green
            $fixed++
        }
        catch {
            Write-Host "❌ Failed: $($acct.userName)" -ForegroundColor Red
        }
    }
    else {
        $skipped++
    }
}

Write-Host ""
Write-Host "🔥 AUTO-REMEDIATION SUMMARY"
Write-Host "Fixed: $fixed"
Write-Host "Already Compliant: $skipped"
