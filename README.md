# azure-policy-foundations

# Azure Policy Foundations

## Project Overview

This project introduces the fundamentals of **Azure Policy** by creating and assigning a custom policy that requires Azure resources to contain an `Environment` tag.

The project demonstrates how Azure Policy can be used to:

- Create governance rules
- Assign policies to Azure resources
- Audit non-compliant resources
- Test policies before enforcement
- Deny non-compliant deployments
- Monitor policy compliance

---

## Project Objective

The goal of this project is to create a custom Azure Policy that requires resources to contain the following tag:

```text
Environment
```

The policy is first deployed using `DoNotEnforce` mode so that resources can be evaluated without being blocked.

After testing compliance, enforcement is enabled so that Azure blocks resources that do not contain the required tag.

---

## Architecture

```text
Azure Subscription
        |
        v
Resource Group
        |
        v
Azure Policy Assignment
        |
        v
Require Environment Tag
        |
   +----+----+
   |         |
Tag Exists   Tag Missing
   |         |
   v         v
Allowed     Denied
```

---

## Technologies Used

- Microsoft Azure
- Azure Policy
- Azure CLI
- Azure Resource Groups
- Azure Storage Accounts
- Git
- GitHub

---

## Project Structure

```text
azure-policy-foundations/
|
└── policy/
    ├── policy-rule.json
    └── parameters.json
```

---

## Step 1 — Configure Azure CLI

Login to Azure:

```bash
az login
```

Check the active subscription:

```bash
az account show \
  --query "{Subscription:name, SubscriptionId:id}" \
  --output table
```

Set the required subscription if necessary:

```bash
az account set \
  --subscription "<SUBSCRIPTION-ID>"
```

---

## Step 2 — Create Project Variables

```bash
RG="rg-policy-foundations"
LOCATION="canadacentral"

POLICY_NAME="require-environment-tag"
ASSIGNMENT="require-environment-tag-foundations"

SUB_ID=$(az account show \
  --query id \
  --output tsv)

SCOPE="/subscriptions/$SUB_ID/resourceGroups/$RG"
```

---

## Step 3 — Create the Resource Group

```bash
az group create \
  --name "$RG" \
  --location "$LOCATION"
```

---

## Step 4 — Create the Policy Rule

Create:

```text
policy/policy-rule.json
```

```json
{
  "if": {
    "field": "[concat('tags[', parameters('tagName'), ']')]",
    "exists": "false"
  },
  "then": {
    "effect": "deny"
  }
}
```

The policy checks whether the required tag exists.

If the tag is missing, the deployment is denied when enforcement is enabled.

---

## Step 5 — Create the Policy Parameters

Create:

```text
policy/parameters.json
```

```json
{
  "tagName": {
    "type": "String",
    "metadata": {
      "displayName": "Required tag",
      "description": "The tag that resources must contain"
    }
  }
}
```

Using a parameter makes the policy reusable for different tags.

---

## Step 6 — Create the Policy Definition

```bash
az policy definition create \
  --name "$POLICY_NAME" \
  --display-name "Require Environment Tag" \
  --description "Requires resources to contain a specified governance tag." \
  --mode Indexed \
  --rules ./policy/policy-rule.json \
  --params ./policy/parameters.json
```

Verify:

```bash
az policy definition show \
  --name "$POLICY_NAME" \
  --output table
```

---

## Step 7 — Assign the Policy Without Enforcement

```bash
az policy assignment create \
  --name "$ASSIGNMENT" \
  --display-name "Require Environment Tag - Foundations" \
  --policy "$POLICY_NAME" \
  --scope "$SCOPE" \
  --params '{"tagName":{"value":"Environment"}}' \
  --enforcement-mode DoNotEnforce
```

`DoNotEnforce` allows Azure to evaluate resources without blocking deployments.

---

## Step 8 — Create a Non-Compliant Resource

Create a storage account without the `Environment` tag:

```bash
STG_BAD="stbad$RANDOM$RANDOM"

az storage account create \
  --name "$STG_BAD" \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --sku Standard_LRS
```

The deployment should succeed because enforcement is currently disabled.

---

## Step 9 — Check Policy Compliance

Trigger an Azure Policy scan:

```bash
az policy state trigger-scan \
  --resource-group "$RG"
```

View compliance:

```bash
az policy state list \
  --resource-group "$RG" \
  --query "[].{Resource:resourceId,Compliance:complianceState}" \
  --output table
```

The storage account should be identified as non-compliant.

---

## Step 10 — Enable Enforcement

```bash
az policy assignment update \
  --name "$ASSIGNMENT" \
  --scope "$SCOPE" \
  --enforcement-mode Default
```

The policy can now block non-compliant deployments.

---

## Step 11 — Test Policy Enforcement

Attempt to create another storage account without the required tag:

```bash
STG_DENIED="stdeny$RANDOM$RANDOM"

az storage account create \
  --name "$STG_DENIED" \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --sku Standard_LRS
```

The deployment should fail with an error similar to:

```text
RequestDisallowedByPolicy
```

This confirms that the policy is working correctly.

---

## Step 12 — Create a Compliant Resource

Create a storage account with the required tag:

```bash
STG_GOOD="stgood$RANDOM$RANDOM"

az storage account create \
  --name "$STG_GOOD" \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --sku Standard_LRS \
  --tags Environment=dev
```

Verify the tags:

```bash
az storage account show \
  --name "$STG_GOOD" \
  --resource-group "$RG" \
  --query tags \
  --output json
```

Expected result:

```json
{
  "Environment": "dev"
}
```

---

## Troubleshooting

### Policy is not blocking deployments

Check the enforcement mode:

```bash
az policy assignment show \
  --name "$ASSIGNMENT" \
  --scope "$SCOPE" \
  --query enforcementMode
```

If the result is:

```text
DoNotEnforce
```

enable enforcement:

```bash
az policy assignment update \
  --name "$ASSIGNMENT" \
  --scope "$SCOPE" \
  --enforcement-mode Default
```

### Compliance data is not showing

Trigger another scan:

```bash
az policy state trigger-scan \
  --resource-group "$RG"
```

Then check:

```bash
az policy state summarize \
  --resource-group "$RG"
```

### MissingSubscription Error

Verify the active subscription:

```bash
az account show
```

Set the correct subscription:

```bash
az account set \
  --subscription "<SUBSCRIPTION-ID>"
```

---

## Cleanup

Delete the policy assignment:

```bash
az policy assignment delete \
  --name "$ASSIGNMENT" \
  --scope "$SCOPE"
```

Delete the custom policy definition:

```bash
az policy definition delete \
  --name "$POLICY_NAME"
```

Delete the resource group:

```bash
az group delete \
  --name "$RG" \
  --yes \
  --no-wait
```

---

## What I Learned

This project provided hands-on experience with:

- Azure Policy definitions
- Azure Policy assignments
- Policy parameters
- Policy effects
- `Deny` enforcement
- `DoNotEnforce` testing
- Azure Policy compliance
- Manual compliance scans
- Azure governance
- Resource tagging standards
- Azure CLI

---

## Key Takeaway

Azure Policy provides centralized governance controls that can automatically evaluate and enforce organizational standards across Azure resources.

The project demonstrated a safe policy deployment workflow:

```text
Create Policy
      ↓
Assign with DoNotEnforce
      ↓
Evaluate Compliance
      ↓
Test Resources
      ↓
Enable Enforcement
      ↓
Block Non-Compliant Deployments
```

This approach allows governance controls to be tested before they begin affecting production deployments.