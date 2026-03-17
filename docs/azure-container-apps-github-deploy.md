# Deploy This App From GitHub to Azure Container Apps

This guide walks through a clean setup from scratch for deploying this repository to Azure Container Apps with GitHub Actions.

It is written for this repo specifically:

- The container listens on port `3000`.
- The image exposes port `3000`.
- The app writes persistent data under `/data/docuseal`.
- A production deployment should not keep credentials hardcoded in workflow files.

## 1. What You Are Building

Recommended deployment flow:

1. GitHub Actions builds the image from this repository.
2. GitHub Actions pushes the image to Azure Container Registry (ACR).
3. Azure Container Apps pulls the image from ACR.
4. Azure Container Apps runs the app with external HTTPS ingress.
5. Azure Files is mounted to `/data/docuseal` so uploads, SQLite data, and generated files survive restarts.

## 1.5. Script vs UI: Quick Comparison

| Aspect | Script (PowerShell CLI) | UI (Azure Portal) |
|--------|----------------------|------------------|
| **Speed** | Fast after initial learning | Slower but straightforward |
| **Learning curve** | Steeper | Shallow, visual |
| **Errors** | Cryptic CLI messages | Clear GUI feedback |
| **Automation** | Easy to automate/repeat | Manual each time |
| **Prerequisites** | Need Azure CLI installed | Just need a browser |
| **Best for** | Experienced DevOps/developers | Learning Azure, first-time users |
| **Can mix approaches?** | Yes! Use both in same workflow |

Each major section has **Option 1 (Script)** and **Option 2 (UI)** steps. Choose whichever you prefer, or mix them.

## 1.6. Key Values Reference (CLI vs Portal)

This table shows where to find critical values depending on your approach:

| Value | PowerShell CLI Command | Azure Portal Location |
|-------|----------------------|----------------------|
| **ACR Login Server** | `az acr show --name $AcrName --query loginServer` | Container Registry → Overview → **Login server** |
| **ACR Username** | `az acr credential show --name $AcrName --query username` | Container Registry → Access keys → **Username** |
| **ACR Password** | `az acr credential show --name $AcrName --query "passwords[0].value"` | Container Registry → Access keys → **Password (1 or 2)** |
| **App URL** | `az containerapp show --name $AppName --query properties.configuration.ingress.fqdn` | Container App → Overview → **Application URL** |
| **Container App Identity** | `az containerapp identity show --name $AppName --query principalId` | Container App → Identity → System assigned → **Object (principal) ID** |
| **Subscription ID** | `az account show --query id` | Subscriptions → Your subscription → Copy **Subscription ID** |
| **Tenant ID** | `az account show --query tenantId` | Azure AD → Overview → **Tenant ID** |
| **Storage Account Key** | `az storage account keys list --account-name $StorageAccount` | Storage account → Access keys → **Key** |

## 2. Prerequisites

### For Script-Based Approach (CLI)

You need:

- An Azure subscription.
- Owner or Contributor access to the target subscription or resource group.
- A GitHub repository containing this app.
- GitHub repo admin access so you can add secrets.
- **Azure CLI installed locally.**
- **The Azure Container Apps CLI extension installed.**

PowerShell setup:

```powershell
az login
az extension add --name containerapp --upgrade
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
```

### For UI-Based Approach (Portal)

You need:

- An Azure subscription.
- Owner or Contributor access to the target subscription or resource group.
- A GitHub repository containing this app.
- GitHub repo admin access so you can add secrets.
- A web browser (no CLI installation needed).

### Choosing Your Approach

- **Script-Based (CLI/PowerShell)**: Faster for experienced users, easier to automate, repeatable on other subscriptions
- **UI-Based (Portal)**: Better for learning, visual feedback at each step, no CLI commands to learn

You can **mix both approaches**—for example, create resources via the Portal and manage secrets via CLI, or vice versa. Each section indicates steps for both.

## 3. Repo Facts You Must Match in Azure

These settings come from the app code and container config:

- App port: `3000`
- Persistent data path: `/data/docuseal`
- Default working directory inside container: `/data/docuseal`

You can verify those in this repo:

- [Dockerfile](../Dockerfile)
- [config/puma.rb](../config/puma.rb)
- [docker-compose.yml](../docker-compose.yml)

## 4. Choose One Deployment Branch

Pick one Git branch and use it consistently everywhere.

Recommended:

- Use `main` as the deployment branch.

If your repo still uses `master`, either:

- Rename the default branch to `main`, or
- Update the workflow to deploy from `master`

Do not mix both. A branch mismatch is one of the most common reasons Azure keeps showing the placeholder container page.

## 5. Create Azure Resources

### Option 1: Using PowerShell Script

Set variables first:

```powershell
$Location = "eastus"
$ResourceGroup = "documust-rg"
$AcrName = "documustacr12345"
$EnvironmentName = "documust-env"
$AppName = "documust-app"
$StorageAccount = "documuststorage12345"
$FileShare = "docuseal-data"
$StorageMountName = "docusealfiles"
```

Create a resource group:

```powershell
az group create --name $ResourceGroup --location $Location
```

Create Azure Container Registry.

This guide enables the ACR admin account initially because it simplifies first-time bootstrap and GitHub configuration. You can tighten this later.

```powershell
az acr create `
  --name $AcrName `
  --resource-group $ResourceGroup `
  --location $Location `
  --sku Basic `
  --admin-enabled true
```

Create the Container Apps environment:

```powershell
az containerapp env create `
  --name $EnvironmentName `
  --resource-group $ResourceGroup `
  --location $Location
```

### Option 2: Using Azure Portal UI

**Create a Resource Group:**
1. Go to [portal.azure.com](https://portal.azure.com)
2. Search for **Resource groups** in the search bar
3. Click **Create**
4. Fill in:
   - **Subscription**: Select your subscription
   - **Resource group name**: `documust-rg` (or your chosen name)
   - **Region**: `East US` (or your preferred region)
5. Click **Review + Create**, then **Create**

**Create Azure Container Registry:**
1. Search for **Container registries** in the portal search bar
2. Click **Create**
3. Fill in:
   - **Subscription**: Select your subscription
   - **Resource group**: Select `documust-rg`
   - **Registry name**: `documustacr12345` (must be globally unique, lowercase alphanumeric)
   - **Location**: `East US` (match your resource group location)
   - **SKU**: `Basic`
4. Click **Review + Create**, then **Create**

**After creation, enable admin user:**
1. Open your ACR resource
2. Go to **Access keys** (left sidebar)
3. Toggle **Admin user** to **Enabled**
4. Note the **Username** and **Passwords** for later use

**Create Container Apps Environment:**
1. Search for **Container Apps environments** in the portal
2. Click **Create**
3. Fill in:
   - **Subscription**: Select your subscription
   - **Resource group**: Select `documust-rg`
   - **Name**: `documust-env`
   - **Region**: `East US`
4. Click **Review + Create**, then **Create** (this takes a few minutes)

## 6. Create Persistent Storage

This app stores working files under `/data/docuseal`. If you skip persistence, data can disappear when revisions restart or move.

### Option 1: Using PowerShell Script

Create the storage account and file share:

```powershell
az storage account create `
  --name $StorageAccount `
  --resource-group $ResourceGroup `
  --location $Location `
  --sku Standard_LRS

$StorageKey = az storage account keys list `
  --resource-group $ResourceGroup `
  --account-name $StorageAccount `
  --query "[0].value" -o tsv

az storage share create `
  --name $FileShare `
  --account-name $StorageAccount `
  --account-key $StorageKey
```

Register the Azure Files share in the Container Apps environment:

```powershell
az containerapp env storage set `
  --name $EnvironmentName `
  --resource-group $ResourceGroup `
  --storage-name $StorageMountName `
  --storage-type AzureFile `
  --azure-file-account-name $StorageAccount `
  --azure-file-account-key $StorageKey `
  --azure-file-share-name $FileShare `
  --access-mode ReadWrite
```

### Option 2: Using Azure Portal UI

**Create Storage Account:**
1. Search for **Storage accounts** in the portal
2. Click **Create**
3. Fill in:
   - **Subscription**: Select your subscription
   - **Resource group**: Select `documust-rg`
   - **Storage account name**: `documuststorage12345` (must be globally unique, lowercase alphanumeric)
   - **Region**: `East US`
   - **Performance**: `Standard`
   - **Redundancy**: `Locally-redundant storage (LRS)`
4. Click **Review + Create**, then **Create**

**Create File Share:**
1. After creation, open the storage account
2. Go to **File shares** (left sidebar under Data storage)
3. Click **+ File share**
4. Enter:
   - **Name**: `docuseal-data`
5. Click **Create**

**Get Storage Account Key:**
1. In the storage account, go to **Access keys** (left sidebar)
2. Copy the **Storage account name** and one of the **Keys** (under key1)
3. Save these values for the next portal step

**Register Storage in Container Apps Environment:**
1. Go to your Container Apps environment (`documust-env`)
2. Go to **Storage** (left sidebar under Settings)
3. Click **Add**
4. Fill in:
   - **Storage name**: `docusealfiles`
   - **Storage type**: `Azure File`
   - **Storage account name**: `documuststorage12345`
   - **Storage account key**: Paste the key from step 2
   - **File share name**: `docuseal-data`
5. Click **Add**

## 7. Build the First Image Into ACR

Before GitHub Actions updates the app, create an initial image so the Container App can be created cleanly.

### Option 1: Using PowerShell Script

```powershell
$AcrLoginServer = az acr show --name $AcrName --query loginServer -o tsv

az acr build `
  --registry $AcrName `
  --image "$AppName:initial" `
  .
```

Get ACR bootstrap credentials:

```powershell
$AcrUsername = az acr credential show --name $AcrName --query username -o tsv
$AcrPassword = az acr credential show --name $AcrName --query "passwords[0].value" -o tsv
```

### Option 2: Using Azure Portal UI

**Get ACR Credentials from Portal:**
1. Go to your ACR resource
2. Go to **Access keys** (left sidebar)
3. Copy and save:
   - **Login server** URL (e.g., `documustacr12345.azurecr.io`)
   - **Username**
   - **Password** (password1 or password2)

**Build Image Using Cloud Shell (Easiest Portal-Only Method):**
1. In the Azure portal, click the **Cloud Shell** icon at the top (looks like `>_`)
2. Select **PowerShell** if prompted
3. Run:
   ```powershell
   git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git
   cd YOUR_REPO
   az acr build --registry documustacr12345 --image documust-app:initial .
   ```
   Replace `YOUR_USERNAME` and `YOUR_REPO` with your GitHub repo details
4. Wait for the build to complete (you'll see "Run ID: xxxxx" when done)

**Alternative: Build Locally and Push to ACR**

If you prefer to build on your machine:
1. On your local computer, open PowerShell/Terminal in the repo directory
2. Login to ACR:
   ```powershell
   docker login documustacr12345.azurecr.io -u USERNAME -p PASSWORD
   ```
   Use the username and password from step 1 (Get ACR Credentials)
3. Build and push:
   ```powershell
   docker build -t documustacr12345.azurecr.io/documust-app:initial .
   docker push documustacr12345.azurecr.io/documust-app:initial
   ```
4. Wait for the push to complete

## 8. Create the Container App

Create the app with external ingress and target port `3000`.

### Option 1: Using PowerShell Script

```powershell
az containerapp create `
  --name $AppName `
  --resource-group $ResourceGroup `
  --environment $EnvironmentName `
  --image "$AcrLoginServer/$AppName:initial" `
  --target-port 3000 `
  --ingress external `
  --min-replicas 1 `
  --registry-server $AcrLoginServer `
  --registry-username $AcrUsername `
  --registry-password $AcrPassword
```

Important:

- The target port must be `3000` for this repo.
- If the target port is wrong, GitHub builds can still succeed while Azure shows the placeholder container or failed revisions.

### Option 2: Using Azure Portal UI

**Create Container App:**
1. Search for **Container Apps** in the portal
2. Click **Create Container App**
3. Fill in Basics tab:
   - **Subscription**: Select your subscription
   - **Resource group**: `documust-rg`
   - **Container app name**: `documust-app` (or your chosen name)
   - **Container Apps environment**: Select `documust-env` (created in step 5)
   - **Deployment source**: `Docker container`
4. Click **Next: Container**

**Configure Container:**
1. In the Container tab:
   - **Image source**: `Azure Container Registry`
   - **Authentication**: `Use admin credentials`
   - **Registry**: Select your ACR (`documustacr12345`)
   - **Image**: `documust-app`
   - **Image tag**: `initial`
   - **CPU and memory**: `0.5 CPU, 1Gi Memory` (adjust as needed)
2. Click **Next: Ingress**

**Configure Ingress:**
1. In the Ingress tab:
   - **Ingress**: **Enabled** 
   - **Ingress traffic**: `External`
   - **Target port**: `3000` ← **CRITICAL: Must be 3000**
   - **Transport**: `HTTP`
2. Click **Review + Create**, then **Create**

The container app will now be created and should start automatically.

## 9. Give the Container App Permission to Pull From ACR

### Option 1: Using PowerShell Script

Assign a managed identity to the Container App:

```powershell
az containerapp identity assign `
  --name $AppName `
  --resource-group $ResourceGroup `
  --system-assigned
```

Get the Container App principal ID and the ACR resource ID:

```powershell
$PrincipalId = az containerapp identity show `
  --name $AppName `
  --resource-group $ResourceGroup `
  --query principalId -o tsv

$AcrId = az acr show --name $AcrName --resource-group $ResourceGroup --query id -o tsv
```

Grant `AcrPull`:

```powershell
az role assignment create `
  --assignee $PrincipalId `
  --role AcrPull `
  --scope $AcrId
```

Tell Container Apps to use its managed identity for ACR pulls:

```powershell
az containerapp registry set `
  --name $AppName `
  --resource-group $ResourceGroup `
  --server $AcrLoginServer `
  --identity system
```

### Option 2: Using Azure Portal UI

**Enable Managed Identity:**
1. Go to your Container App (`documust-app`)
2. Go to **Identity** (left sidebar under Settings)
3. In the **System assigned** tab, toggle **Status** to **On**
4. Click **Save**
5. Note the **Object (principal) ID** for the next step

**Assign AcrPull Role:**
1. Go to your Container Registry (`documustacr12345`)
2. Go to **Access Control (IAM)** (left sidebar)
3. Click **Add** → **Add role assignment**
4. In the Role tab:
   - **Role**: Search for and select `AcrPull`
5. Click **Next**
6. In the Members tab:
   - **Assign access to**: `Managed identity`
   - Click **+ Select members**
   - **Subscription**: Select your subscription
   - **Managed identity**: Select `Container App`
   - **Select**: Check your container app (`documust-app`)
7. Click **Select** → **Review + assign** → **Assign**

**Update Container App Registry Settings:**
1. Go back to your Container App (`documust-app`)
2. Go to **Registries** (left sidebar under Settings)
3. Click **Update**
4. Change:
   - **Use admin credentials**: Toggle to **OFF**
   - **Identity**: Select `System assigned`
5. Click **Save**

This switches from using hardcoded ACR credentials to using the managed identity.

## 10. Add the Storage Mount to the Container App

### Option 1: Using PowerShell Script

Azure Container Apps uses YAML to add Azure Files volume mounts.

Export the current app definition:

```powershell
az containerapp show `
  --name $AppName `
  --resource-group $ResourceGroup `
  --output yaml > app.yaml
```

Edit `app.yaml` and add the following under `properties.template`.

```yaml
containers:
  - image: <keep existing image value>
    name: <keep existing container name>
    volumeMounts:
      - volumeName: docuseal-data-volume
        mountPath: /data/docuseal
volumes:
  - name: docuseal-data-volume
    storageType: AzureFile
    storageName: docusealfiles
```

Then apply the YAML:

```powershell
az containerapp update `
  --name $AppName `
  --resource-group $ResourceGroup `
  --yaml app.yaml
```

### Option 2: Using Azure Portal UI

**Add Volume Mount via Portal:**
1. Go to your Container App (`documust-app`)
2. Go to **Revisions and replicas** (left sidebar under Application)
3. Click your active revision (usually the one with a recent creation time)
4. Click **Edit and deploy** at the top
5. In the Container tab:
   - Scroll down to **Volume mounts**
   - Click **Add**
   - Fill in:
     - **Volume mount name**: `docuseal-data-volume`
     - **Mount path**: `/data/docuseal`
     - **Storage name**: Select `docusealfiles` (registered in step 6)
   - Click **Add volume mount**
6. Click **Create** at the bottom to deploy the updated revision

This creates a new revision with the storage mount attached. The app will use this mount path for all persistent data.

## 11. Set Application Secrets and Environment Values

### Option 1: Using PowerShell Script

At minimum, set a strong `SECRET_KEY_BASE`.

Generate one locally:

```powershell
[guid]::NewGuid().ToString("N") + [guid]::NewGuid().ToString("N") + [guid]::NewGuid().ToString("N")
```

Store it in the Container App:

```powershell
$SecretKeyBase = "<paste-a-long-random-secret-here>"

az containerapp secret set `
  --name $AppName `
  --resource-group $ResourceGroup `
  --secrets secret-key-base=$SecretKeyBase
```

Attach environment variables to the app:

```powershell
az containerapp update `
  --name $AppName `
  --resource-group $ResourceGroup `
  --set-env-vars SECRET_KEY_BASE=secretref:secret-key-base PORT=3000
```

### Option 2: Using Azure Portal UI

**Generate a Secret Key:**

Get a random strong secret at the command line:

```powershell
[guid]::NewGuid().ToString("N") + [guid]::NewGuid().ToString("N") + [guid]::NewGuid().ToString("N")
```

Copy the output.

**Add Secrets and Environment Variables via Portal:**
1. Go to your Container App (`documust-app`)
2. Go to **Revisions and replicas** (left sidebar under Application)
3. Click your active revision
4. Click **Edit and deploy**
5. In the Container tab, scroll down to **Environment variables**:
   - Click **Add**
   - Enter:
     - **Name**: `PORT`
     - **Value**: `3000`
   - Click **Add environment variable**
6. In the Secrets section (above Environment variables):
   - Click **Add**
   - Enter:
     - **Name**: `secret-key-base`
     - **Value**: Paste your generated secret from above
   - Click **Add secret**
7. Add another environment variable:
   - Click **Add**
   - Enter:
     - **Name**: `SECRET_KEY_BASE`
     - **Value**: Click **Reference a secret** icon and select `secret-key-base`
   - Click **Add environment variable**
8. Click **Create** at the bottom to deploy with the new secrets and variables

Optional environment variables depending on your setup:

- `DATABASE_URL` if you want PostgreSQL or MySQL instead of SQLite
- `ENCRYPTION_SECRET` if you want a value separate from `SECRET_KEY_BASE`
- `FORCE_SSL=false` only if you have a specific proxy or redirect issue

For a minimal first deployment, this repo can start with:

- `PORT=3000`
- `SECRET_KEY_BASE` as a secret reference

## 12. Create GitHub Credentials for Azure Login

This is what the workflow uses in the `azure/login@v2` step.

### Option 1: Using PowerShell Script

Create a service principal scoped to the resource group:

```powershell
az ad sp create-for-rbac `
  --name "documust-github-actions" `
  --role contributor `
  --scopes "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$ResourceGroup" `
  --sdk-auth
```

Copy the JSON output.

In GitHub, go to:

- `Settings`
- `Secrets and variables`
- `Actions`

Create a repository secret named:

- `AZURE_CREDENTIALS`

Paste the full JSON into that secret.

Example structure:

```json
{
  "clientId": "<appId>",
  "clientSecret": "<password>",
  "subscriptionId": "<subscriptionId>",
  "tenantId": "<tenantId>"
}
```

### Option 2: Using Azure Portal + GitHub UI

**Create Service Principal via Portal:**
1. Search for **App registrations** in the Azure portal
2. Click **New registration**
3. Fill in:
   - **Name**: `documust-github-actions`
   - **Supported account types**: `Accounts in this organizational directory only`
4. Click **Register**

**Create Client Secret:**
1. In your app, go to **Certificates & secrets** (left sidebar)
2. Click **New client secret** (under Client secrets)
3. Fill in:
   - **Description**: `GitHub Actions secret`
   - **Expires**: `24 months`
4. Click **Add**
5. **Copy the Value** immediately (you won't see it again)

**Get Required IDs:**
1. In the app's **Overview** tab, copy:
   - **Application (client) ID**
   - **Directory (tenant) ID**
2. Go back to your subscription, copy the **Subscription ID**:
   - Search for **Subscriptions** in portal
   - Copy your subscription's ID

**Add Permissions to Service Principal:**

You have two options:

**Option A: Assign the built-in Azure RBAC `Contributor` role at the resource-group scope (Simpler, broader permissions)**
1. Go to your resource group (`documust-rg`)
2. Go to **Access Control (IAM)**
3. Click **Add** → **Add role assignment**
4. Select role: `Contributor`
5. Click **Next**
6. Click **+ Select members**
7. Search for your app by name (`documust-github-actions`)
8. Click **Select** → **Review + assign** → **Assign**

This means the GitHub service principal gets the Azure role named `Contributor` **for resources inside `documust-rg` only**. It can create, update, and delete Azure resources in that resource group, but it does **not** become subscription owner and it does **not** get permission to manage RBAC role assignments unless you separately grant that.

**Option B: Minimal Permissions (More secure, granular)**

If you prefer tighter security, grant only what's needed:

1. Go to your resource group (`documust-rg`)
2. Go to **Access Control (IAM)**
3. For each role below, click **Add** → **Add role assignment**:
  - Role: `Container Apps Contributor` (for Container App updates)
  - Role: `AcrPush` (for pushing images to ACR, scoped to your ACR resource)
  - Role: `Storage File Data SMB Share Contributor` (for storage mount access, scoped to storage account)
4. Each time, select your service principal (`documust-github-actions`) as the member

**Recommendation:**
- **First deployment**: Use Option A (Contributor) for simplicity — easier to debug if permissions are wrong
- **Production/Long-term**: Use Option B (minimal permissions) after everything works

**Add to GitHub:**
1. In GitHub, go to your repo: **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. **Name**: `AZURE_CREDENTIALS`
4. **Value**: Paste this JSON (fill in the IDs from above):
   ```json
   {
     "clientId": "<Application (client) ID>",
     "clientSecret": "<client secret value>",
     "subscriptionId": "<Subscription ID>",
     "tenantId": "<Directory (tenant) ID>"
   }
   ```
5. Click **Add secret**

## 13. Add GitHub Secrets for ACR

### Option 1: Using PowerShell Script

Create these GitHub Actions secrets:

- `REGISTRY_USERNAME`
- `REGISTRY_PASSWORD`

Values:

```powershell
az acr credential show --name $AcrName
```

Also create plain repository variables or hardcode safe non-secret values inside the workflow for:

- ACR login server
- Resource group
- Container app name

### Option 2: Using Azure Portal + GitHub UI

**Get ACR Credentials from Portal:**
1. Go to your Container Registry in Azure portal
2. Go to **Access keys** (left sidebar)
3. Copy and save:
   - **Username** (under Admin user credentials)
   - **Password** (one of the two passwords listed)

**Add Secrets to GitHub:**
1. In GitHub, go to your repo: **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Add first secret:
   - **Name**: `REGISTRY_USERNAME`
   - **Value**: Paste the username from ACR
   - Click **Add secret**
4. Click **New repository secret** again
5. Add second secret:
   - **Name**: `REGISTRY_PASSWORD`
   - **Value**: Paste the password from ACR
   - Click **Add secret**

**Add Repository Variables (Optional but Recommended):**

Instead of hardcoding values in the workflow, use repository variables:

1. In GitHub: **Settings** → **Secrets and variables** → **Variables**
2. Click **New repository variable** for each:
   - **Variable name**: `REGISTRY_LOGIN_SERVER`
     **Value**: `documustacr12345.azurecr.io` (your ACR login server)
   - **Variable name**: `RESOURCE_GROUP`
     **Value**: `documust-rg`
   - **Variable name**: `CONTAINER_APP_NAME`
     **Value**: `documust-app`

These can then be referenced in your workflow as `${{ vars.REGISTRY_LOGIN_SERVER }}` instead of hardcoding.

## 14. Create the GitHub Actions Workflow

Create `.github/workflows/deploy-aca.yml` with one deployment path only.

Recommended workflow:

```yaml
name: Deploy to Azure Container Apps

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Build and deploy to Azure Container Apps
        uses: azure/container-apps-deploy-action@v2
        with:
          appSourcePath: ${{ github.workspace }}
          dockerfilePath: ./Dockerfile
          registryUrl: <YOUR_ACR_NAME>.azurecr.io
          registryUsername: ${{ secrets.REGISTRY_USERNAME }}
          registryPassword: ${{ secrets.REGISTRY_PASSWORD }}
          containerAppName: <YOUR_CONTAINER_APP_NAME>
          resourceGroup: <YOUR_RESOURCE_GROUP>
          imageToBuild: <YOUR_ACR_NAME>.azurecr.io/<YOUR_IMAGE_NAME>:${{ github.sha }}
```

Replace:

- `<YOUR_ACR_NAME>`
- `<YOUR_CONTAINER_APP_NAME>`
- `<YOUR_RESOURCE_GROUP>`
- `<YOUR_IMAGE_NAME>`

If your branch is `master`, change the workflow branch trigger to `master`.

## 15. Push the Workflow and Trigger Deployment

Commit the workflow and push to the deployment branch.

```powershell
git add .github/workflows/deploy-aca.yml
git commit -m "Add Azure Container Apps deployment workflow"
git push origin main
```

Then check GitHub:

- Open the `Actions` tab.
- Open the `Deploy to Azure Container Apps` run.
- Wait for the build-and-deploy job to finish.

## 16. Verify the App in Azure

### Option 1: Using PowerShell Script

Get the app URL:

```powershell
az containerapp show `
  --name $AppName `
  --resource-group $ResourceGroup `
  --query properties.configuration.ingress.fqdn -o tsv
```

List revisions:

```powershell
az containerapp revision list `
  --name $AppName `
  --resource-group $ResourceGroup `
  -o table
```

Stream logs:

```powershell
az containerapp logs show `
  --name $AppName `
  --resource-group $ResourceGroup `
  --follow
```

If needed, inspect a specific revision:

```powershell
az containerapp revision show `
  --name $AppName `
  --resource-group $ResourceGroup `
  --revision <REVISION_NAME>
```

### Option 2: Using Azure Portal UI

**View Application URL and Status:**
1. Go to your Container App (`documust-app`)
2. In the Overview tab, find the **Application URL** under Essentials
3. Click it to open your app (or copy and paste into browser)
4. Check if the app loads correctly

**View App Revisions and Logs:**
1. In your Container App, go to **Revisions and replicas** (left sidebar)
2. You'll see a list of revisions with their:
   - **FQDN** (full URL)
   - **Traffic** percentage (active revisions)
   - **Created** time
   - **Status** (Active, Inactive)
3. Click on a revision to see:
   - Container logs
   - System logs
   - Replica status
   - Configuration details

**Stream Logs for Debugging:**
1. Go to **Revisions and replicas**
2. Click on your active revision
3. Click on a replica
4. Scroll down to see **Container logs** and **System logs**
5. You can refresh to see new log entries

**If Your App Is Still Showing Placeholder:**
1. Check the active revision status (should show "Active" and green checkmark)
2. Look at "System logs" to find why the revision is not healthy
3. Check if all environment variables were properly set:
   - Go to your revision → Click **Edit and deploy**
   - Scroll to **Environment variables** section
4. Verify the target port in Ingress settings is `3000`

## 17. Common Problems and Fixes

### GitHub build is green, but Azure still shows the placeholder container

Check these first:

- The workflow ran on the correct branch.
- The deploy workflow ran, not just CI.
- The Container App ingress target port is `3000`.
- The app revision became healthy after deployment.

### Azure says the cloud build is still in progress

Usually this means one of these is true:

- The deployment workflow never started.
- The deployment workflow failed before updating the Container App.
- The new revision was created but failed health checks or startup.

### Revision fails after deployment

Check:

- `SECRET_KEY_BASE` exists.
- Container App can pull from ACR.
- Persistent storage is mounted to `/data/docuseal`.
- Target port is `3000`.

### Data disappears after restart

You are likely running without the Azure Files volume mounted to `/data/docuseal`.

## 18. Security Cleanup You Should Do Immediately

If you ever committed registry passwords directly into a workflow file:

1. Rotate the ACR password immediately.
2. Remove the password from the workflow.
3. Move it into GitHub Actions secrets.
4. Redeploy after the workflow is cleaned up.

## 19. Recommended Final State

Your final setup should look like this:

- One deployment workflow only
- One deployment branch only
- ACR for image storage
- Azure Container App using target port `3000`
- Azure Files mounted to `/data/docuseal`
- `SECRET_KEY_BASE` stored as a Container App secret
- Azure login JSON stored in GitHub as `AZURE_CREDENTIALS`
- No hardcoded passwords in the repository

## 19.5. Final Verification Checklist

Use this checklist to verify your complete setup before declaring success:

### Azure Resources
- [ ] Resource group exists (`documust-rg` or chosen name)
- [ ] Container Registry exists with admin credentials enabled
- [ ] Container Apps environment exists
- [ ] Storage account exists with a file share named `docuseal-data`
- [ ] Storage is registered in Container Apps environment

### Container App Configuration
- [ ] Container app exists (`documust-app` or chosen name)
- [ ] Target ingress port is `3000`
- [ ] External ingress is enabled
- [ ] Environment variable `PORT=3000`
- [ ] Secret `secret-key-base` exists
- [ ] Environment variable `SECRET_KEY_BASE` references the secret
- [ ] Volume mount exists: `/data/docuseal` → `docusealfiles` storage
- [ ] Managed identity is assigned (System assigned)
- [ ] Managed identity has `AcrPull` role on the ACR
- [ ] Container App registry is configured to use managed identity (not admin credentials)

### GitHub Setup
- [ ] Repository secret `AZURE_CREDENTIALS` exists (JSON format with clientId, clientSecret, subscriptionId, tenantId)
- [ ] Repository secret `REGISTRY_USERNAME` exists
- [ ] Repository secret `REGISTRY_PASSWORD` exists
- [ ] Workflow file `.github/workflows/deploy-aca.yml` exists
- [ ] Workflow triggers on the correct branch (`main` or `master`, should match repo default branch)
- [ ] No hardcoded passwords in any workflow files

### Deployment Status
- [ ] First GitHub Actions build completed successfully
- [ ] Container App revision shows "Active" status
- [ ] Container App URL is accessible in a browser
- [ ] App loads without showing placeholder message
- [ ] No "unhealthy" revisions in revision list

### Data Persistence
- [ ] App can create files in `/data/docuseal` (upload a document if your app has that feature)
- [ ] Files persist after container restart/redeployment
- [ ] Azure Files share shows files created by the app

## 20. Official References

This guide is based on Microsoft Learn guidance for:

- Azure Container Apps GitHub Actions deployment
- Azure Container Apps managed identity image pull
- Azure Container Apps revisions and log streaming
- Azure Container Apps Azure Files storage mounts
