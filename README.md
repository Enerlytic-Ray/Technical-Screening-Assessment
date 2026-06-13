# BI Mini Challenge Submission

This repository contains solutions to the mini technical assessment covering:

## Task 1: Advanced DAX (Dynamic RLS)
Implemented dynamic Row-Level Security (RLS) using:
- `USERPRINCIPALNAME()`
- `LOOKUPVALUE()`
- `PATHCONTAINS()`

This allows managers to view:
- Their own records
- Direct reports
- Indirect reports in the hierarchy

**SQL Solution:**
```sql
VAR CurrentEmployeeKey =
    LOOKUPVALUE(
        dim_employee[EmployeeKey],
        dim_employee[EmployeeEmail],
        USERPRINCIPALNAME()
    )
RETURN
    PATHCONTAINS(
        dim_employee[ParentPath],
        CurrentEmployeeKey
    )
    || dim_employee[EmployeeKey] = CurrentEmployeeKey`

```
---

## Task 2: T-SQL Pipeline Logic (SCD Type 2)
Implemented a Slowly Changing Dimension (SCD Type 2) process for an Asset Register including:

- New record insertion
- Change detection
- Historical versioning
- `EffectiveFrom`
- `EffectiveTo`
- `IsCurrent`

**SQL Solution:**
```sql

BEGIN TRANSACTION;

BEGIN TRY

    ---------------------------------------------------------
    -- STEP 1: Insert brand-new assets
    ---------------------------------------------------------
    INSERT INTO dim_AssetRegister
    (
        AssetID,
        AssetName,
        AssetCategory,
        AssetLocation,
        AssetOwner,
        PurchaseValue,
        AssetStatus,
        EffectiveFrom,
        EffectiveTo,
        IsCurrent
    )
    SELECT
        s.AssetID,
        s.AssetName,
        s.AssetCategory,
        s.AssetLocation,
        s.AssetOwner,
        s.PurchaseValue,
        s.AssetStatus,
        GETDATE() AS EffectiveFrom,
        '9999-12-31' AS EffectiveTo,
        1 AS IsCurrent
    FROM stg_AssetRegister s
    LEFT JOIN dim_AssetRegister d
        ON s.AssetID = d.AssetID
        AND d.IsCurrent = 1
    WHERE d.AssetID IS NULL;


    ---------------------------------------------------------
    -- STEP 2: Expire changed records
    ---------------------------------------------------------
    UPDATE d
    SET
        d.EffectiveTo = GETDATE(),
        d.IsCurrent = 0
    FROM dim_AssetRegister d
    INNER JOIN stg_AssetRegister s
        ON d.AssetID = s.AssetID
    WHERE d.IsCurrent = 1
      AND (
            ISNULL(d.AssetName, '') <> ISNULL(s.AssetName, '')
         OR ISNULL(d.AssetCategory, '') <> ISNULL(s.AssetCategory, '')
         OR ISNULL(d.AssetLocation, '') <> ISNULL(s.AssetLocation, '')
         OR ISNULL(d.AssetOwner, '') <> ISNULL(s.AssetOwner, '')
         OR ISNULL(d.PurchaseValue, 0) <> ISNULL(s.PurchaseValue, 0)
         OR ISNULL(d.AssetStatus, '') <> ISNULL(s.AssetStatus, '')
      );


    ---------------------------------------------------------
    -- STEP 3: Insert new version of changed records
    ---------------------------------------------------------
    INSERT INTO dim_AssetRegister
    (
        AssetID,
        AssetName,
        AssetCategory,
        AssetLocation,
        AssetOwner,
        PurchaseValue,
        AssetStatus,
        EffectiveFrom,
        EffectiveTo,
        IsCurrent
    )
    SELECT
        s.AssetID,
        s.AssetName,
        s.AssetCategory,
        s.AssetLocation,
        s.AssetOwner,
        s.PurchaseValue,
        s.AssetStatus,
        GETDATE() AS EffectiveFrom,
        '9999-12-31' AS EffectiveTo,
        1 AS IsCurrent
    FROM stg_AssetRegister s
    INNER JOIN dim_AssetRegister d
        ON s.AssetID = d.AssetID
    WHERE d.IsCurrent = 0
      AND d.EffectiveTo = GETDATE();


    COMMIT TRANSACTION;

END TRY
BEGIN CATCH

    ROLLBACK TRANSACTION;

    THROW;

END CATCH;

```
---

## Task 3: BI Platform REST API Deployment
Provided a PowerShell-based pseudo-script demonstrating:

- REST API authentication
- Artifact promotion from `/Dev` to `/Test`
- Configuration refresh

---

**SQL Solution:**
```sql

# -------------------------------------------------------
# Configuration
# -------------------------------------------------------

$BaseUrl   = "https://bi-platform.company.com/api/v1"
$Username  = "svc_bi_deployment"
$Password  = "********"

$SourcePath = "/Dev"
$TargetPath = "/Test"

# -------------------------------------------------------
# Step 1: Authenticate and Get Access Token
# -------------------------------------------------------

$AuthBody = @{
    username = $Username
    password = $Password
} | ConvertTo-Json

$TokenResponse = Invoke-RestMethod `
    -Uri "$BaseUrl/auth/login" `
    -Method Post `
    -ContentType "application/json" `
    -Body $AuthBody

$AccessToken = $TokenResponse.access_token

$Headers = @{
    Authorization = "Bearer $AccessToken"
    ContentType   = "application/json"
}

# -------------------------------------------------------
# Step 2: Retrieve Artifact(s) from /Dev
# -------------------------------------------------------

$Artifacts = Invoke-RestMethod `
    -Uri "$BaseUrl/artifacts?path=$SourcePath" `
    -Method Get `
    -Headers $Headers

# -------------------------------------------------------
# Step 3: Deploy Artifact(s) to /Test
# -------------------------------------------------------

foreach ($Artifact in $Artifacts.items)
{
    $DeployBody = @{
        sourcePath      = "$SourcePath/$($Artifact.name)"
        destinationPath = "$TargetPath/$($Artifact.name)"
        overwrite       = $true
    } | ConvertTo-Json

    Invoke-RestMethod `
        -Uri "$BaseUrl/deployment/copy" `
        -Method Post `
        -Headers $Headers `
        -Body $DeployBody

    Write-Host "Successfully deployed $($Artifact.name) from Dev to Test"
}

# -------------------------------------------------------
# Step 4: Refresh Configuration (Optional)
# -------------------------------------------------------

Invoke-RestMethod `
    -Uri "$BaseUrl/configuration/refresh" `
    -Method Post `
    -Headers $Headers

Write-Host "Test environment configuration refreshed successfully."

```
