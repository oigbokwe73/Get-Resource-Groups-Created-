Excellent — you’re asking how to get the **creation timestamp (or last modified time)** of Azure Resource Groups using the **Azure Resource Manager (ARM) REST API** via PowerShell.

Let’s go step-by-step 👇

---

## 🧠 Background

Azure Resource Groups themselves **do not store a `createdTime` field** in the standard `az group list` or `Get-AzResourceGroup` output.
However, you can fetch that metadata from the **Azure Management REST API** using:

```
GET https://management.azure.com/subscriptions/{subscriptionId}/resourcegroups/{resourceGroupName}?api-version=2021-04-01
```

The response contains a property under:

```
"properties": {
  "provisioningState": "Succeeded",
  "createdTime": "2025-11-04T14:32:19.4047683Z",
  "changedTime": "2025-11-06T18:15:12.8757461Z",
  "tags": {},
  "resourceGroupType": "Default"
}
```

Those `createdTime` and `changedTime` are what you’re after.

---

## ⚙️ PowerShell Script

Here’s a full working script that authenticates, loops through all resource groups in your subscription, and displays timestamps:

```powershell
# Variables
$subscriptionId = "<your-subscription-id>"
$apiVersion = "2021-04-01"

# Login if needed
Connect-AzAccount
Set-AzContext -SubscriptionId $subscriptionId

# Get access token for the REST call
$token = (Get-AzAccessToken -ResourceUrl "https://management.azure.com").Token

# Get all resource groups in the subscription
$resourceGroups = Get-AzResourceGroup

# Create an array to hold results
$result = @()

foreach ($rg in $resourceGroups) {
    $url = "https://management.azure.com/subscriptions/$subscriptionId/resourcegroups/$($rg.ResourceGroupName)?api-version=$apiVersion"
    $response = Invoke-RestMethod -Uri $url -Headers @{ Authorization = "Bearer $token" } -Method GET
    $result += [PSCustomObject]@{
        ResourceGroupName = $rg.ResourceGroupName
        Location          = $rg.Location
        CreatedTime       = $response.properties.createdTime
        ChangedTime       = $response.properties.changedTime
        ProvisioningState = $response.properties.provisioningState
    }
}

# Display result
$result | Format-Table -AutoSize
```

---

## 📊 Optional: Export to CSV

If you want to store the output:

```powershell
$result | Export-Csv -Path "C:\Temp\ResourceGroupTimestamps.csv" -NoTypeInformation
```

---

## 🧩 Sample Output

| ResourceGroupName | Location | CreatedTime             | ChangedTime             | ProvisioningState |
| ----------------- | -------- | ----------------------- | ----------------------- | ----------------- |
| rg-network-prod   | eastus   | 2025-10-30T13:42:51.74Z | 2025-11-05T17:01:10.13Z | Succeeded         |
| rg-storage-dev    | westus2  | 2025-09-25T11:20:19.31Z | 2025-11-03T09:44:07.44Z | Succeeded         |

---

Would you like me to modify this script to **also include the user (principal) who created each resource group** (via Activity Log lookup)? That’s possible but requires an additional API call.


Excellent question — and a common point of confusion!
Azure **does not directly store “createdDate” or “createdTime”** as a visible property of Resource Groups.
However, you can **derive or retrieve it** in a few reliable ways using **Activity Logs**, **Azure CLI**, **PowerShell**, or **Resource Graph**.

Let’s go through each option 👇

---

## 🧭 **1️⃣ Using Azure Activity Logs (Best for Historical Accuracy)**

When a Resource Group is created, Azure logs an **"Create Resource Group"** event in the **Activity Log**.

### **Azure CLI**

```bash
# Replace with your subscription ID
SUBSCRIPTION_ID="<your-subscription-id>"

# Get resource group creation timestamps
az monitor activity-log list \
  --subscription $SUBSCRIPTION_ID \
  --status Succeeded \
  --offset 90d \
  --query "[?operationName.value=='Microsoft.Resources/subscriptions/resourcegroups/write'].[resourceGroupName, eventTimestamp]" \
  -o table
```

✅ **Output Example:**

| ResourceGroupName | EventTimestamp       |
| ----------------- | -------------------- |
| rg-app-prod       | 2025-10-28T15:24:31Z |
| rg-data-dev       | 2025-10-29T08:14:02Z |

> The `--offset 90d` flag limits the search to the last 90 days.
> You can adjust it to `--offset 365d` for a year, or remove it to query all logs (slower).

---

## 🧩 **2️⃣ Using Azure PowerShell**

```powershell
# Login
Connect-AzAccount

# Retrieve Resource Group creation events
Get-AzActivityLog -StartTime (Get-Date).AddDays(-90) `
  | Where-Object {$_.OperationNameValue -eq "Microsoft.Resources/subscriptions/resourcegroups/write"} `
  | Select-Object ResourceGroupName, EventTimestamp
```

✅ This gives you the same creation timestamps for each Resource Group.

---

## 🧠 **3️⃣ Using Azure Resource Graph (Fastest for Current State)**

While Resource Graph doesn’t directly show “createdTime”, you can use metadata or tags (if you enforced tagging on creation).

```bash
az graph query -q "ResourceContainers 
| where type=='microsoft.resources/subscriptions/resourcegroups'
| project name, id, tags, properties.provisioningState, subscriptionId"
```

👉 If your team uses a “CreatedOn” or “CreatedDate” tag, it will show here.
Otherwise, combine this with Activity Logs (Method 1) for actual creation timestamps.

---

## 🔄 **4️⃣ Optional — Combine with Automation**

You can store this data in Azure Table Storage or Log Analytics using a Function App or Logic App that:

* Periodically queries `Activity Logs`
* Extracts all “write” events for `resourceGroups`
* Writes `{ ResourceGroupName, EventTimestamp }` to a central location

---

## 🧾 **5️⃣ REST API Method**

You can call the **Activity Logs REST API** directly:

```bash
curl -X GET "https://management.azure.com/subscriptions/<subscriptionId>/providers/microsoft.insights/eventtypes/management/values?api-version=2017-03-01-preview&$filter=eventTimestamp ge '2025-01-01T00:00:00Z' and operationName/value eq 'Microsoft.Resources/subscriptions/resourcegroups/write'" \
  -H "Authorization: Bearer $(az account get-access-token --query accessToken -o tsv)"
```

Then parse `.value[].eventTimestamp` for creation times.

---

### ✅ **Summary Table**

| Method         | Accuracy    | Pros                       | Cons                                        |
| -------------- | ----------- | -------------------------- | ------------------------------------------- |
| Activity Log   | ✅ Exact     | Shows actual creation time | Retention (90 days default unless exported) |
| PowerShell     | ✅ Exact     | Easy for scripting         | Slow for large subscriptions                |
| Resource Graph | ⚙️ Indirect | Fast and scalable          | No creation timestamp by default            |
| REST API       | ✅ Exact     | Integrates with automation | Requires token handling                     |

---

