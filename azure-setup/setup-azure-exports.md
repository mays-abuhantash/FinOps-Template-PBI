# Configure Azure Cost Management exports

A reproducible, click-by-click guide to setting up the daily blob exports that feed this accelerator.

You can configure exports either via the **Azure portal** or via **Azure CLI / PowerShell / Bicep**. Both are below.

---

### Step 1 — Create the Resource Group

A Resource Group is a logical container that holds all related Azure resources together.

1. Sign in to the **Azure Portal** at `portal.azure.com` with your organisational credentials.
2. In the top search bar, type **Resource groups** and click it.
3. Click **+ Create**.
4. Fill in the details:
   - **Subscription:** your target subscription
   - **Resource Group Name:** `rg-{org}-{projectName}-{region}` (e.g. `rg-contoso-finops-eastus`)
   - **Region:** choose the region closest to your team (e.g. East US, West Europe)
5. Click **Review + Create**, verify the details, then click **Create**.

> **Best Practice:** Use a consistent naming convention — `rg-{org}-{projectName}-{region}` clearly identifies the subscription owner, project, and region at a glance.

---

### Step 2 — Create the storage account & container

#### Storage Account

1. In the top search bar, type **Storage accounts** and click it.
2. Click **+ Create**.
3. Fill in the **Basics** tab:
   - **Subscription / Resource group:** use the Resource Group created in Step 1
   - **Storage account name:** `{org}{project}{custom}` — lowercase letters and numbers only, no special characters, 3–24 chars, globally unique (e.g. `contosfinopsprd`)
   - **Region:** same region as your Resource Group
   - **Performance:** Standard
   - **Redundancy:** LRS is the minimum for cost exports. LRS is not recommended per the Cloud Adoption Framework (CAF); if the customer is unsure, choose **ZRS**. Use GRS or GZRS if disaster recovery is required.
4. Under the **Advanced** tab:
   - **Access Tier:** Hot
   - Leave all other options at their defaults.
5. Under the **Networking** tab:
   - **Network access:** select **Enable public access from all networks**
   - Leave all other settings as default.

   > **Note:** Public access is the only supported approach for Cost Management to deliver export files to a storage account. Private Endpoint is not yet supported for this scenario. Access will be scoped to Microsoft trusted services at the firewall level after creation. See [Tutorial — Create and manage Cost Management exports](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-improved-exports) for details.

6. Click **Review + Create** → **Create**. Wait for deployment to finish (usually 1–2 minutes), then click **Go to resource**.
7. After the storage account is created, navigate to **Settings → Configuration** and set **Permitted scope for copy operations (preview)** to **From any storage account**.

> **Azure Policy Note:** If the storage account is deployed under a Corp management group, or if an Azure Policy exists that prevents PaaS resources from having public access, a **Policy Exemption** will need to be created for this storage account before exports can be configured.

#### Blob Container

The single container holds all three export types, each separated by a directory prefix (`amortized/`, `actual/`, `focus/`).

1. Inside the storage account, navigate to **Data Storage → Containers**.
2. Click **+ Container**:
   - **Name:** `{org}-{project}-cost-exports` (e.g. `contoso-finops-cost-exports`)
   - **Anonymous access level:** Private (leave as default)
3. Click **Create**.

---

### Step 3 — Data residency, access requirements & compatibility

#### Data Residency

- All export data is stored in the Azure region you selected for the storage account. Confirm this aligns with your organisation's data residency and compliance requirements before creating the resource.
- If regulations require data to remain within a specific geography, choose **ZRS** or **GZRS** within that region rather than GRS, which replicates to a paired region that may be in a different country.

#### Required Azure Access

The account configuring Cost Management exports requires the following roles:

| Role | Scope | Purpose |
|---|---|---|
| Cost Management Contributor (or higher) | Subscription | Create and manage exports |
| Owner or User Access Administrator | Storage account | Allow Cost Management to assign the managed identity |
| Storage Blob Data Contributor | Storage account | Write export files to the container |

> A system-assigned managed identity is created automatically for a new export if the configuring user has `Microsoft.Authorization/roleAssignments/write` permissions on the storage account. After the export is created, Owner-level access on the storage account is no longer required for routine operations. If exports are not appearing after 24 hours, verify the managed identity assignment under **Storage Account → Access Control (IAM)**.

#### Application Compatibility

- **Power BI Desktop:** Always use the latest version when opening the `.pbit` template. Older versions may not support the visual types or M query functions used in this accelerator, resulting in compatibility warnings or broken visuals on open.
- **Power BI Service:** Ensure the workspace capacity (Shared, Premium, or Fabric) supports the data volume of your cost exports. Very large exports (multi-GB) may require Fabric or Premium capacity for reliable scheduled refresh.
- **File format:** This accelerator expects `.csv.gz` (GZIP-compressed CSV). Do not change the export compression format after initial setup — doing so will break the Power Query source step in the `.pbix`/`.pbit` file.

---

### Step 4 — Create the Cost Management exports

> **Managed Identity Note:** A system-assigned managed identity is created for a new export if the user has `Microsoft.Authorization/roleAssignments/write` permissions on the storage account. This ensures exports continue to work if you later enable a storage firewall. After the export is created, Owner-level access on the storage account is no longer required for routine operations.

#### 4a — Amortized Cost (Daily)

Amortized cost spreads reservation and savings plan fees evenly across the usage period — ideal for budget tracking and chargeback.

1. In the top search bar, type **Cost Management** and click it.
2. At the top of the Cost Management page, confirm the **scope** is set to your target subscription. If not, click the scope selector and choose it.
3. In the left-hand menu, click **Exports** → **+ Create**.
4. Fill in the export details:
   - **Export name:** `{org}-{project}-daily-amortized`
   - **Metric / Dataset:** Amortized cost
   - **Export type:** Daily export of month-to-date costs
   - **Start date:** today's date
5. Set the storage destination:
   - **Subscription:** the subscription holding your storage account
   - **Storage account:** the account created in Step 2
   - **Container:** `{org}-{project}-cost-exports`
   - **Directory / Prefix:** `amortized`
   - **Format:** CSV
   - **Compression:** GZIP
6. Click **Next** → **Review + Create** → **Create**.
7. Click your new export → **Run now** to seed the first file immediately (otherwise the first file may take up to 24 hours to appear).

> **Best Practice:** Use Amortized cost when you want to see the true daily cost including reserved instances spread evenly — ideal for chargeback and budget tracking.

---

#### 4b — Actual Cost (Daily)

Actual cost shows the exact amount billed each day without spreading reservations — ideal for invoice reconciliation and auditing.

1. In the top search bar, type **Cost Management** and click it.
2. Confirm the **scope** is set to your target subscription.
3. In the left-hand menu, click **Exports** → **+ Create**.
4. Fill in the export details:
   - **Export name:** `{org}-{project}-daily-actual`
   - **Metric / Dataset:** Actual cost
   - **Export type:** Daily export of month-to-date costs
   - **Start date:** today's date
5. Set the storage destination:
   - **Subscription:** the subscription holding your storage account
   - **Storage account:** the account created in Step 2
   - **Container:** `{org}-{project}-cost-exports`
   - **Directory / Prefix:** `actual`
   - **Format:** CSV
   - **Compression:** GZIP
6. Click **Next** → **Review + Create** → **Create**.
7. Click your new export → **Run now** to seed the first file immediately.

> **Best Practice:** Use Actual cost for invoice reconciliation and to see exactly what was charged on a given day — ideal for finance and auditing teams.

---

#### 4c — FOCUS Cost (Daily)

FOCUS (FinOps Open Cost and Usage Specification) combines actual and amortized costs in one file — the modern standard for FinOps analysis.

> **What is FOCUS?** An open-source industry standard format developed by the FinOps Foundation. It combines actual and amortized costs in a single file, reduces data processing time, and is designed to work across multiple cloud providers. Use this if your team works with Power BI, Azure Synapse, Microsoft Fabric, or any FinOps tooling.

1. In the top search bar, type **Cost Management** and click it.
2. Confirm the **scope** is set to your target subscription.
3. In the left-hand menu, click **Exports** → **+ Create**.
4. Fill in the export details:
   - **Export name:** `{org}-{project}-daily-focus`
   - **Metric / Dataset:** Cost and usage details (FOCUS)
   - **Export type:** Daily export of month-to-date costs
   - **Start date:** today's date
5. Set the storage destination:
   - **Subscription:** the subscription holding your storage account
   - **Storage account:** the account created in Step 2
   - **Container:** `{org}-{project}-cost-exports`
   - **Directory / Prefix:** `focus`
   - **Format:** CSV
   - **Compression:** GZIP
6. Click **Next** → **Review + Create** → **Create**.
7. Click your new export → **Run now** to seed the first file immediately.

> **Best Practice:** FOCUS is the recommended format if you plan to use Power BI, Microsoft Fabric, or any FinOps tool. It combines actual and amortized costs so you only need one file instead of two.

---

### Step 5 — Grant Power BI the right to read

For the account that will run the dataset refresh in Power BI Service:

1. Storage account → **Access Control (IAM)** → **+ Add → Add role assignment**
2. Role: **Storage Blob Data Reader**
3. Assign access to: **User, group, or service principal**
4. Select the user or service principal that will own the dataset refresh
5. **Review + assign**

Wait ~5 minutes for the role to propagate before testing in Power BI.

---

## Verification checklist

Before opening the `.pbit`, confirm:

- [ ] At least one `.csv.gz` file (not a manifest) exists in the container under each prefix (`amortized/`, `actual/`, `focus/`)
- [ ] Your Power BI account has **Storage Blob Data Reader** on the storage account
- [ ] Storage account firewall is either open or has Power BI Service IPs allow-listed
- [ ] The storage account **Permitted scope for copy operations** is set to **From any storage account**
- [ ] You can browse the container in Storage Explorer / portal with your account
- [ ] You are using the latest version of Power BI Desktop to open the `.pbit`

If all are ✅, the `.pbit` will load on first try.

---

## Official documentation

- [Create a Storage Account](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-create)
- [Cost Management Exports](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-improved-exports)
- [FOCUS Format Documentation](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/export-cost-data-storage-account)
