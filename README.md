# Azure AD Automation — GitHub Actions + Entra ID

Automating common IT administration tasks using **GitHub Actions** and **Microsoft Entra ID** via the Microsoft Graph API.  
No servers. No scripts running on a VM. Just cloud-native automation triggered directly from GitHub.

---

## What this project does

This project automates four of the most common IT helpdesk tasks:

| Workflow | What it does | Trigger |
|----------|-------------|---------|
| `reset-password.yml` | Resets a user's Entra ID password and forces change on next sign-in | Manual |
| `unlock-account.yml` | Re-enables a locked/disabled Entra ID account | Manual |
| `create-user.yml` | Creates a new user in Entra ID (onboarding) | Manual |
| `disable-user.yml` | Disables account and revokes all active sessions (offboarding) | Manual |

All workflows are triggered manually via **GitHub Actions → workflow_dispatch**, meaning an admin fills in a form in the GitHub UI and clicks **Run workflow** — no code changes required.

---

## Architecture

```
Admin (GitHub UI)
       │
       ▼
GitHub Actions (workflow_dispatch form)
       │
       ▼
Azure Login (Service Principal — stored in GitHub Secrets)
       │
       ▼
Microsoft Graph API
       │
       ▼
Entra ID (user management)
```

**Key design decisions:**
- Credentials are never hardcoded — stored in GitHub Encrypted Secrets
- Service principal follows least-privilege (User Administrator role only)
- All workflow runs are logged in GitHub Actions for audit purposes
- Session revocation on offboarding ensures immediate access removal

---

## Tech stack

- **GitHub Actions** — workflow automation and CI/CD
- **Microsoft Entra ID** — cloud identity platform (formerly Azure AD)
- **Microsoft Graph API** — REST API for managing Microsoft 365 and Entra ID
- **Azure CLI** — used inside workflows to authenticate and call Graph API
- **GitHub Encrypted Secrets** — secure credential storage

---

## Prerequisites

- An Azure subscription with an Entra ID tenant
- A GitHub account and repository
- Azure CLI installed locally (for initial setup only)
- Sufficient permissions to create service principals in Entra ID

---

## Setup guide

### Step 1 — Create the service principal

> **Note:** "User Administrator" is an Entra ID role, not an Azure RBAC role, so it cannot be assigned directly via the `--role` flag. We create the service principal first, then assign the role separately via the Graph API.

Run this in your terminal:

```bash
az ad sp create-for-rbac \
  --name "github-ad-automation" \
  --sdk-auth
```

Copy the **entire JSON output** — you will need it in Step 3.

---

### Step 2 — Assign the User Administrator role

Run these commands in PowerShell:

```powershell
# Get the service principal Object ID
$SP_OBJECT_ID = az ad sp list --display-name "github-ad-automation" --query "[0].id" -o tsv

# Write the role assignment to a temp file (avoids PowerShell JSON quoting issues)
@'
{
  "roleDefinitionId": "fe930be7-5e62-47db-91af-98c3a49a38b1",
  "principalId": "YOUR-SP-OBJECT-ID",
  "directoryScopeId": "/"
}
'@ | Out-File -FilePath "$env:TEMP\role.json" -Encoding utf8

# Assign the User Administrator role
az rest --method POST `
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleAssignments" `
  --headers "Content-Type=application/json" `
  --body "@$env:TEMP\role.json"
```

> The `roleDefinitionId` `fe930be7-5e62-47db-91af-98c3a49a38b1` is the permanent built-in ID for the User Administrator role and never changes across tenants.

**Verify the role was assigned:**

Go to **Azure Portal → Entra ID → Roles and administrators → User Administrator** — you should see `github-ad-automation` listed as a member.

![alt text](image.png)

---

### Step 3 — Add GitHub Secrets

In your GitHub repository go to **Settings → Secrets and Variables → Actions → New repository secret** and add:

| Secret name | Value |
|-------------|-------|
| `AZURE_CREDENTIALS` | The full JSON output from Step 1 |
| `AZURE_TENANT_ID` | Your Azure tenant ID |

---

### Step 4 — Clone and push this repo

```bash
git clone https://github.com/danqiu-dev/azure-ad-automation.git
cd azure-ad-automation
git add .
git commit -m "Initial commit — AD automation workflows"
git push
```

---

### Step 5 — Run a workflow

1. Go to your repository on GitHub
2. Click the **Actions** tab
3. Select a workflow (e.g. "Create new user")
4. Click **Run workflow**
5. Fill in the form fields
6. Click the green **Run workflow** button
7. Verify the result in **Azure Portal → Entra ID → Users**

---

## Testing & verification

Each workflow below includes exact steps to run it and verify the result in the Azure Portal.

---

### Test 1 — Create new user (onboarding)

**Run the workflow:**

1. Go to **Actions → Create new user (onboarding) → Run workflow**
2. Fill in the form:

   | Field | Test value |
   |-------|-----------|
   | Full name | `Apple Lee` |
   | Email | `apple.lee@yourdomain.com` |
   | Department | `IT` |

3. Click **Run workflow**
4. Wait for the green checkmark ✅

**Verify in GitHub Actions:**

Click the completed run → expand the **"Create user via Graph API"** step → you should see a `201 Created` response with the new user's details in JSON format.

![Create user - GitHub Actions success](images/create-user-actions.png)

**Verify in Azure Portal:**

1. Go to **portal.azure.com → Entra ID → Users**
2. Search for `Apple Lee`
3. You should see the account with:
   - Display name: `Apple Lee`
   - User principal name: `apple.lee@yourdomain.com`
   - Department: `IT`
   - Account status: **Enabled**

![Apple Lee created in Entra ID](images/create-user-azure.png)

---

### Test 2 — Disable user (offboarding)

**Run the workflow:**

1. Go to **Actions → Disable user (offboarding) → Run workflow**
2. Fill in the form:

   | Field | Test value |
   |-------|-----------|
   | User email | `apple.lee@yourdomain.com` |

3. Click **Run workflow**
4. Wait for the green checkmark ✅

**Verify in GitHub Actions:**

Click the completed run → you should see two steps succeed:
- **"Disable account"** — returns `204 No Content` (success, no body)
- **"Revoke all active sessions"** — returns `200 OK`

![Disable user - GitHub Actions success](images/disable-user-actions.png)

**Verify in Azure Portal:**

1. Go to **portal.azure.com → Entra ID → Users**
2. Search for `Apple Lee`
3. You should see:
   - Account status: **Account disabled** (shown in red)
   - All active sessions immediately terminated

![Apple Lee disabled in Entra ID](images/disable-user-azure.png)

---

### Test 3 — Unlock account

**Run the workflow:**

1. Go to **Actions → Unlock user account → Run workflow**
2. Fill in the form:

   | Field | Test value |
   |-------|-----------|
   | User email | `apple.lee@yourdomain.com` |

3. Click **Run workflow**
4. Wait for the green checkmark ✅

**Verify in GitHub Actions:**

Click the completed run → expand **"Unlock account via Graph API"** → should return `204 No Content`.

![Unlock account - GitHub Actions success](images/unlock-account-actions.png)

**Verify in Azure Portal:**

1. Go to **portal.azure.com → Entra ID → Users**
2. Search for `Apple Lee`
3. Account status should now show **Enabled** again

![Apple Lee unlocked in Entra ID](images/unlock-account-azure.png)

---

### Test 4 — Reset password

**Run the workflow:**

1. Go to **Actions → Reset user password → Run workflow**
2. Fill in the form:

   | Field | Test value |
   |-------|-----------|
   | User email | `apple.lee@yourdomain.com` |

3. Click **Run workflow**
4. Wait for the green checkmark ✅

**Verify in GitHub Actions:**

Click the completed run → expand **"Reset password via Graph API"** → should return `204 No Content`.

![Reset password - GitHub Actions success](images/reset-password-actions.png)

**Verify in Azure Portal:**

1. Go to **portal.azure.com → Entra ID → Users**
2. Search for `Apple Lee` → click her account
3. Go to **Authentication methods** tab
4. You will see the password was updated — last password change timestamp will reflect the current time

![Password reset confirmed in Entra ID](images/reset-password-azure.png)

> **Tip:** To fully confirm the password reset, sign in as Apple Lee at [myaccount.microsoft.com](https://myaccount.microsoft.com) using the temporary password `TempPass@2026!` — you will be immediately prompted to set a new password.

---

### Test 5 — Full lifecycle test (recommended)

Run all four workflows in sequence to simulate a complete employee lifecycle:

```
1. Create user    →  Apple Lee account created
2. Reset password →  Password changed to temporary
3. Unlock account →  Account re-enabled
4. Disable user   →  Account disabled, sessions revoked
```

All four runs will appear in the **Actions** tab with timestamps, inputs, and results — this serves as a complete audit trail.

![Full lifecycle test in GitHub Actions](images/full-lifecycle-actions.png)

---

## Workflows in detail

### Reset password (`reset-password.yml`)

**Inputs:**
- `user_email` — the UPN of the user (e.g. `john.smith@yourdomain.com`)

**What it does:**
1. Authenticates to Azure using the service principal
2. Calls Graph API `PATCH /users/{email}` to set a temporary password
3. Forces the user to change their password on next sign-in

**Use case:** User forgot their password or is locked out.

---

### Unlock account (`unlock-account.yml`)

**Inputs:**
- `user_email` — the UPN of the user

**What it does:**
1. Authenticates to Azure
2. Calls Graph API `PATCH /users/{email}` with `accountEnabled: true`

**Use case:** Account was disabled due to too many failed sign-in attempts.

---

### Create user — onboarding (`create-user.yml`)

**Inputs:**
- `display_name` — user's full name (e.g. `Apple Lee`)
- `email` — new UPN (e.g. `apple.lee@yourdomain.com`)
- `department` — user's department (e.g. `IT`, `Finance`, `HR`)

**What it does:**
1. Authenticates to Azure
2. Calls Graph API `POST /users` to create a new Entra ID account
3. Sets a temporary password with forced change on first sign-in

**Use case:** New employee starting — create their account before day one.

---

### Disable user — offboarding (`disable-user.yml`)

**Inputs:**
- `user_email` — the UPN of the user to offboard

**What it does:**
1. Authenticates to Azure
2. Disables the account via `PATCH /users/{email}` with `accountEnabled: false`
3. Revokes all active sign-in sessions via `POST /users/{email}/revokeSignInSessions`

**Use case:** Employee leaving the company — immediately cut off all access.

---

## Security considerations

- **No plaintext credentials** — all secrets stored in GitHub Encrypted Secrets
- **Least privilege** — service principal only has `User Administrator` role, nothing broader
- **Audit trail** — every workflow run is permanently logged in GitHub Actions with timestamp, trigger, and inputs
- **Session revocation** — offboarding immediately invalidates all active tokens, not just disabling the account
- **Forced password change** — all password resets require the user to set a new password on next sign-in

---

## Lessons learned during setup

- The `--role "User Administrator"` flag in `az ad sp create-for-rbac` does not work because User Administrator is an Entra ID directory role, not an Azure RBAC role. It must be assigned separately via the Graph API `roleManagement` endpoint.
- PowerShell multiline JSON in `az rest --body` causes Bad Request errors due to quote escaping. Writing the JSON to a temp file with `Out-File` solves this cleanly.
- The `--sdk-auth` flag in `az ad sp create-for-rbac` shows a deprecation warning but still works as of March 2026.

---

## Future improvements

- [ ] Add email notification to user after password reset
- [ ] Add Teams notification to manager after onboarding/offboarding
- [ ] Add Terraform to provision the service principal as Infrastructure as Code
- [ ] Add Pester tests to verify each workflow completed successfully
- [ ] Add approval gate — require a second admin to approve sensitive actions
- [ ] Extend to M365 license assignment on onboarding

---

## About this project

Built as a portfolio project to demonstrate practical skills in:
- **Azure / Entra ID** — cloud identity management and Microsoft Graph API
- **GitHub Actions** — workflow automation, secrets management, and manual triggers
- **Cloud security** — service principals, least privilege, audit logging, Zero Trust principles

This project is based on real IT helpdesk tasks performed daily as an IT Service Technician, automated and moved entirely to the cloud.

---

## Author

**Dan Qiu**  
IT Service Technician → aspiring Cloud / DevOps Engineer  
[LinkedIn](https://linkedin.com/in/dan-qiu-725558209) · [GitHub](https://github.com/danqiu-dev)

---

## License

MIT — free to use, fork, and build on.
# github-ad-automation
