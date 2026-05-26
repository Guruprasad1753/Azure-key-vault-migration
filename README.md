# 🔐 Azure Key Vault Migration Guide

[![Azure](https://img.shields.io/badge/Azure-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)](https://azure.microsoft.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](https://opensource.org/licenses/MIT)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=for-the-badge)](http://makeapullrequest.com)
[![Made with Markdown](https://img.shields.io/badge/Made%20with-Markdown-1f425f.svg?style=for-the-badge)](https://commonmark.org)

> **A complete beginner-friendly guide to migrate Azure Key Vault secrets from one environment to another**

---

## 📋 Table of Contents

- [What is Azure Key Vault?](#-what-is-azure-key-vault)
- [Why Migrate Key Vault?](#-why-migrate-key-vault)
- [Before You Start](#-before-you-start)
- [Method 1: GUI Method (1–5 Secrets)](#️-method-1-gui-method-15-secrets)
- [Method 2: CLI Method (5+ Secrets)](#-method-2-cli-method-5-secrets)
- [Migrating Keys & Certificates](#-migrating-keys--certificates)
- [Troubleshooting](#-troubleshooting)
- [FAQ](#-faq)
- [Migration Checklist](#-migration-checklist)
- [Method Comparison](#-method-comparison)

---

## 🎯 What is Azure Key Vault?

Think of Azure Key Vault as a **digital safe** in the cloud. It securely stores:

| Type | Example |
|------|---------|
| 🔑 **Secrets** | Database passwords, API keys, connection strings |
| 🔐 **Keys** | Encryption keys for sensitive data |
| 📜 **Certificates** | SSL/TLS certificates for websites |

**Without Key Vault:** Passwords are stored in code files (insecure)  
**With Key Vault:** Apps request secrets from the vault at runtime (secure)

---

## 🚀 Why Migrate Key Vault?

Common scenarios where you need to migrate:

- Moving from **Development → Production** environment
- Moving from **Testing → Staging** environment
- Reorganizing Azure subscriptions
- Disaster recovery setup
- Region migration

> ⚠️ **Important:** You **cannot** directly "move" a Key Vault between resource groups or subscriptions while keeping the same URL. You must create a new vault and copy the contents. This guide shows you exactly how.

---

## 📋 Before You Start

### Prerequisites

| Item | Required | Notes |
|------|----------|-------|
| Azure subscription | ✅ Yes | With access to both vaults |
| Old Key Vault name | ✅ Yes | Source vault |
| New Key Vault name | ✅ Yes | Must be pre-created |
| `Key Vault Administrator` role | ✅ Yes | On **both** vaults |
| Azure CLI 2.50.0+ | ⭐ Recommended | Required for Method 2 |

> 💡 **RBAC vs Access Policies:** Azure now recommends using **Azure Role-Based Access Control (RBAC)** instead of the legacy Vault Access Policies. If your new vault uses RBAC, assign the `Key Vault Administrator` role via **Access Control (IAM)** on the vault resource.

### ⚠️ Critical: Check Soft-Delete & Purge Protection

Azure Key Vault has **soft-delete enabled by default** (secrets are retained for 7–90 days after deletion). If you deleted a secret recently, it may still exist in a "deleted" state and block re-creation with the same name.

```bash
# List soft-deleted secrets in the OLD vault
az keyvault secret list-deleted --vault-name $OLD_VAULT -o table

# Recover a deleted secret if needed
az keyvault secret recover --vault-name $OLD_VAULT --name "SecretName"

# Or permanently purge it (only if purge protection is NOT enabled)
az keyvault secret purge --vault-name $OLD_VAULT --name "SecretName"
```

### Find Your Vault Information

```bash
# List all your Key Vaults
az keyvault list --query "[].{Name:name, ResourceGroup:resourceGroup, Location:location}" -o table
```

Example output:

```
Name                ResourceGroup         Location
------------------  --------------------  ----------
DevAppSecrets       Development-RG        eastus
ProdAppSecrets      Production-RG         eastus2
```

📝 Write these down before proceeding:

- **Old Vault Name:** ___________
- **New Vault Name:** ___________
- **Resource Groups:** ___________

---

## 🖥️ Method 1: GUI Method (1–5 Secrets)

**Best for:** Beginners, small migrations, 1–5 secrets  
**Time:** 5–15 minutes

### Step 1: Open Both Vaults

1. Go to [Azure Portal](https://portal.azure.com)
2. Search for **Key Vaults**
3. Open your **OLD** vault in one browser tab
4. Open your **NEW** vault in another browser tab

### Step 2: Copy Secrets

**From OLD Vault:**

```
Secrets → Click secret name → Current Version → Show Secret Value → Copy value
```

**To NEW Vault:**

```
Secrets → + Generate/Import → Manual → Paste name → Paste value → Create
```

> ⚠️ Secret names are **case-sensitive**. Copy the name exactly as shown.

### Step 3: Copy Access Policies (or RBAC Roles)

**Legacy Access Policies:**

- OLD Vault → **Access policies** → Note each application/identity and its permissions
- NEW Vault → **Access policies** → **+ Create** → Recreate each policy

**Azure RBAC (recommended):**

- OLD Vault → **Access control (IAM)** → **Role assignments** → Note each assignment
- NEW Vault → **Access control (IAM)** → **+ Add** → **Add role assignment** → Recreate each

### Step 4: Verify

Check that all secrets appear in the new vault with the correct names and values.

✅ **GUI Method Complete!**

---

## 💻 Method 2: CLI Method (5+ Secrets)

**Best for:** Large migrations, DevOps/IT professionals  
**Time:** 2–10 minutes for 100+ secrets

### Part A: Install Azure CLI

<details>
<summary><b>Click to expand installation commands</b></summary>

**Windows (PowerShell — run as Administrator):**

```powershell
iex (New-Object Net.WebClient).DownloadString('https://aka.ms/installazurecliwindows')
```

**macOS (Terminal):**

```bash
brew install azure-cli
# Or without Homebrew:
/bin/bash -c "$(curl -fsSL https://aka.ms/installazureclimac)"
```

**Linux (Ubuntu/Debian):**

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

**Verify installation:**

```bash
az --version
```

</details>

---

### Part B: Login to Azure

```bash
az login
```

A browser window will open. Sign in with your Azure credentials. After login, set your target subscription if you have multiple:

```bash
# List subscriptions
az account list --query "[].{Name:name, ID:id}" -o table

# Set the correct subscription
az account set --subscription "Your-Subscription-Name-or-ID"
```

---

### Part C: Set Your Variables

Replace with **your actual vault names**:

<details>
<summary><b>Bash / macOS / Linux</b></summary>

```bash
export OLD_VAULT="DevAppSecrets"
export NEW_VAULT="ProdAppSecrets"
```

</details>

<details>
<summary><b>Windows PowerShell</b></summary>

```powershell
$env:OLD_VAULT = "DevAppSecrets"
$env:NEW_VAULT = "ProdAppSecrets"
```

</details>

<details>
<summary><b>Windows Command Prompt</b></summary>

```cmd
set OLD_VAULT=DevAppSecrets
set NEW_VAULT=ProdAppSecrets
```

</details>

---

### Part D: List Secrets in the Old Vault

Before copying, review what you're migrating:

```bash
# List all ENABLED secrets
az keyvault secret list --vault-name $OLD_VAULT --query "[?attributes.enabled==\`true\`].name" -o tsv

# List ALL secrets including disabled ones
az keyvault secret list --vault-name $OLD_VAULT --query "[].{Name:name, Enabled:attributes.enabled}" -o table
```

> 💡 The bulk-copy scripts below only migrate **enabled** secrets. If you need disabled secrets too, remove the `enabled` filter from the list command.

---

### Part E: Test Copy One Secret

Always test with one secret before running the bulk migration:

```bash
# Replace 'DatabasePassword' with any secret name from your vault
TEST_SECRET="DatabasePassword"

SECRET_VALUE=$(az keyvault secret show \
  --vault-name $OLD_VAULT \
  --name $TEST_SECRET \
  --query "value" -o tsv)

az keyvault secret set \
  --vault-name $NEW_VAULT \
  --name $TEST_SECRET \
  --value "$SECRET_VALUE" \
  --only-show-errors

echo "✅ Test copy successful"
```

---

### Part F: Copy ALL Secrets (Bulk)

<details>
<summary><b>🪟 Windows — Command Prompt</b></summary>

```cmd
for /f "tokens=*" %i in ('az keyvault secret list --vault-name %OLD_VAULT% --query "[].name" -o tsv') do (
    echo Copying secret: %i
    for /f "tokens=*" %v in ('az keyvault secret show --vault-name %OLD_VAULT% --name %i --query "value" -o tsv') do (
        az keyvault secret set --vault-name %NEW_VAULT% --name %i --value "%v" --only-show-errors
    )
    echo Done: %i
)
```

</details>

<details>
<summary><b>🪟 Windows — PowerShell</b></summary>

```powershell
$secrets = az keyvault secret list --vault-name $env:OLD_VAULT --query "[].name" -o tsv

foreach ($secretName in $secrets) {
    Write-Host "Copying: $secretName" -ForegroundColor Yellow
    $secretValue = az keyvault secret show `
        --vault-name $env:OLD_VAULT `
        --name $secretName `
        --query "value" -o tsv
    az keyvault secret set `
        --vault-name $env:NEW_VAULT `
        --name $secretName `
        --value $secretValue `
        --only-show-errors | Out-Null
    Write-Host "✓ Copied: $secretName" -ForegroundColor Green
}

Write-Host "`n✅ All secrets copied!" -ForegroundColor Cyan
```

</details>

<details>
<summary><b>🍎🐧 macOS / Linux — Bash</b></summary>

```bash
az keyvault secret list --vault-name $OLD_VAULT --query "[].name" -o tsv | while read secret_name; do
    echo "Copying: $secret_name"
    secret_value=$(az keyvault secret show \
        --vault-name $OLD_VAULT \
        --name "$secret_name" \
        --query "value" -o tsv)
    az keyvault secret set \
        --vault-name $NEW_VAULT \
        --name "$secret_name" \
        --value "$secret_value" \
        --only-show-errors > /dev/null
    echo "✓ Copied: $secret_name"
done

echo ""
echo "✅ All secrets copied!"
```

</details>

---

### Part G: Verify Migration

```bash
# Count secrets in old vault
echo "Old vault count:"
az keyvault secret list --vault-name $OLD_VAULT --query "length([])"

# Count secrets in new vault
echo "New vault count:"
az keyvault secret list --vault-name $NEW_VAULT --query "length([])"
```

✅ Both numbers should match!

If they don't, find missing secrets:

```bash
# macOS/Linux only
comm -23 \
  <(az keyvault secret list --vault-name $OLD_VAULT --query "[].name" -o tsv | sort) \
  <(az keyvault secret list --vault-name $NEW_VAULT --query "[].name" -o tsv | sort)
```

---

## 📜 Migrating Keys & Certificates

### Keys

> ⚠️ **RSA and EC private keys cannot be exported** from Azure Key Vault unless the key was created as **exportable** (`--exportable true`). If your key is non-exportable, you must generate a new key in the destination vault and re-encrypt your data.

```bash
# Check if a key is exportable
az keyvault key show --vault-name $OLD_VAULT --name "MyKey" --query "attributes"

# Export an exportable key (requires Key Vault Crypto Officer role)
az keyvault key download --vault-name $OLD_VAULT --name "MyKey" --file mykey.pem --encoding PEM

# Import into new vault
az keyvault key import --vault-name $NEW_VAULT --name "MyKey" --pem-file mykey.pem
```

### Certificates

Certificates **cannot** be bulk-copied via the secret method. Use the export/import flow:

```bash
# Export certificate as PFX (includes private key)
az keyvault certificate download \
  --vault-name $OLD_VAULT \
  --name "MyCert" \
  --file mycert.pfx \
  --encoding PFX

# Import into new vault
az keyvault certificate import \
  --vault-name $NEW_VAULT \
  --name "MyCert" \
  --file mycert.pfx
```

> 💡 For certificates managed by a CA (like DigiCert or GlobalSign), re-issue from the CA in the new vault instead of exporting.

---

## 🔧 Troubleshooting

### ❌ `az: command not found` / `'az' is not recognized`

Azure CLI is not installed or not in your PATH.  
→ [Download Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)

---

### ❌ `(Forbidden)` / `Permission denied`

You don't have sufficient permissions on one or both vaults.  
→ Ask your Azure Admin to assign you the **Key Vault Administrator** role on both vaults.

```bash
# Check your current role assignments on a vault
az role assignment list --scope $(az keyvault show --name $OLD_VAULT --query id -o tsv) -o table
```

---

### ❌ `(VaultNotFound)` / `Vault not found`

The vault name is wrong or you're in the wrong subscription.

```bash
# List all vaults in current subscription
az keyvault list --query "[].name" -o tsv

# Switch subscription
az account set --subscription "Your-Subscription-ID"
```

---

### ❌ `(Conflict)` — Secret already exists in deleted state

The secret was recently deleted and is in soft-delete limbo.

```bash
# List deleted secrets
az keyvault secret list-deleted --vault-name $NEW_VAULT -o table

# Purge the deleted secret to allow recreation (irreversible)
az keyvault secret purge --vault-name $NEW_VAULT --name "SecretName"
```

---

### ❌ Secret counts don't match after migration

```bash
# Find missing secrets (macOS/Linux)
comm -23 \
  <(az keyvault secret list --vault-name $OLD_VAULT --query "[].name" -o tsv | sort) \
  <(az keyvault secret list --vault-name $NEW_VAULT --query "[].name" -o tsv | sort)
```

Manually copy any secrets listed in the output.

---

## ❓ FAQ

**Can I just "move" the Key Vault to a new resource group?**  
Yes, but moving a vault between resource groups does **not** change its URL — so applications keep working. This guide is for copying secrets to a **different vault** (e.g., new subscription or new vault name). Use `az resource move` for a simple resource group change.

**Will my applications break during migration?**  
Your old vault stays fully active until you delete it. Applications only break if you update their vault URL **before** confirming the new vault is working. Update applications after verifying the migration, then delete the old vault.

**Do certificates migrate the same way as secrets?**  
No. Use the `az keyvault certificate import/download` commands shown in the [Migrating Keys & Certificates](#-migrating-keys--certificates) section. For CA-issued certificates, re-issue from the CA directly in the new vault.

**How long does migration take?**
| Volume | GUI Method | CLI Method |
|--------|-----------|-----------|
| 5 secrets | ~10 min | ~1 min |
| 50 secrets | 45+ min | ~2 min |
| 200 secrets | Not recommended | ~5 min |

**Can I automate this in a pipeline?**  
Yes. Use the CLI commands in Azure DevOps or GitHub Actions. Authenticate using a Service Principal with the `Key Vault Administrator` role:

```bash
az login --service-principal -u $APP_ID -p $CLIENT_SECRET --tenant $TENANT_ID
```

**What happens to secret versions during migration?**  
Only the **latest enabled version** of each secret is copied. Historical versions are not migrated. If you need all versions, export them individually using `az keyvault secret show --version <version-id>`.

---

## ✅ Migration Checklist

### Before Migration
- [ ] Old Key Vault name and resource group documented
- [ ] New Key Vault already created
- [ ] `Key Vault Administrator` role confirmed on **both** vaults
- [ ] Correct Azure subscription selected (`az account show`)
- [ ] Soft-delete / purge-protection status checked on new vault
- [ ] Azure CLI installed and version verified (`az --version`)
- [ ] All secrets listed and reviewed in old vault

### During Migration
- [ ] Test copy performed with 1 secret ✅
- [ ] Bulk copy script completed without errors
- [ ] Secret counts match between old and new vault
- [ ] Missing secrets (if any) manually copied
- [ ] Access policies / RBAC roles recreated on new vault
- [ ] Keys exported/imported (if applicable)
- [ ] Certificates exported/imported (if applicable)

### After Migration
- [ ] Applications updated with new Vault URL
- [ ] Applications tested end-to-end with new vault
- [ ] Monitoring/alerts configured on new vault
- [ ] 24-hour observation period completed
- [ ] Old Key Vault deleted (optional, after confirmation)

---

## 📊 Method Comparison

| Feature | GUI Method | CLI Method |
|---------|-----------|-----------|
| Best for | 1–5 secrets | 5+ secrets |
| Time for 50 secrets | 45+ minutes | ~2 minutes |
| Technical skill required | Beginner | Intermediate |
| Risk of manual errors | Higher | Lower (automated) |
| Certificate migration | ✅ Easy via portal | ⚠️ Requires export/import |
| Key migration | ✅ Easy via portal | ⚠️ Non-exportable keys cannot be moved |
| CI/CD pipeline support | ❌ No | ✅ Yes |
| Audit trail | ❌ Manual | ✅ Scriptable & repeatable |

---

## 🆘 Getting Help

- 📖 [Microsoft Official Docs — Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/)
- 📖 [Azure CLI Key Vault Reference](https://learn.microsoft.com/en-us/cli/azure/keyvault)
- 💬 [Azure Community Support](https://techcommunity.microsoft.com/t5/azure/ct-p/Azure)
- 🐛 [Report an Issue](../../issues)

---

## 📄 License

This guide is licensed under the [MIT License](LICENSE) — feel free to use, modify, and share!

---

## ⭐ Support

If this guide helped you, please:

- ⭐ Star this repository
- 🔄 Share with your team
- ✏️ Suggest improvements via Pull Request

---

*Made with ❤️ for the Azure community*  
*Last Updated: May 2026 · Azure CLI Version: 2.50.0+*
