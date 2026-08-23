# Azure Storage Redundancy Practical: LRS, ZRS, GRS

This is a good hands-on lab for understanding how Microsoft Azure Storage protects data.

## 1. Basic Concept

When you create an Azure Storage Account, Azure maintains multiple copies of your data. The redundancy option determines **where those copies are stored**.

| Type    | Full Form                 | Copies | Location of copies                     | Protects against          |
| ------- | ------------------------- | -----: | -------------------------------------- | ------------------------- |
| **LRS** | Locally Redundant Storage |      3 | One datacenter / primary region        | Disk/server/rack failures |
| **ZRS** | Zone-Redundant Storage    |      3 | Three availability zones in one region | Availability-zone failure |
| **GRS** | Geo-Redundant Storage     |      6 | Primary + secondary region             | Regional disaster         |

Conceptually:

```text
LRS
Azure Region
└── Datacenter
    ├── Copy 1
    ├── Copy 2
    └── Copy 3
```

```text
ZRS
Azure Region
├── Availability Zone 1 → Copy
├── Availability Zone 2 → Copy
└── Availability Zone 3 → Copy
```

```text
GRS
Primary Region
├── Copy 1
├── Copy 2
└── Copy 3
        │
        │ Geo-replication
        ▼
Secondary Region
├── Copy 4
├── Copy 5
└── Copy 6
```

---

# 2. Practical Lab

### Objective

Create three storage accounts:

```text
storage-lrs
storage-zrs
storage-grs
```

Upload files and compare their redundancy configurations.

> Storage account names must be globally unique, so add random numbers to the names.

## Step 1 — Create LRS Storage

Azure Portal:

```text
Azure Portal
   ↓
Storage accounts
   ↓
Create
```

Configure:

```text
Resource Group: storage-rg
Storage account: atullrs12345
Region: Central India
Performance: Standard
Redundancy: Locally-redundant storage (LRS)
```

Click:

```text
Review + create
        ↓
Create
```

Architecture:

```text
Central India
      │
      └── LRS
           ├── Copy 1
           ├── Copy 2
           └── Copy 3
```

---

## Step 2 — Create ZRS Storage

Create another storage account:

```text
Resource Group: storage-rg
Storage account: atulzrs12345
Region: choose a ZRS-supported region
Performance: Standard
Redundancy: Zone-redundant storage (ZRS)
```

Architecture:

```text
Azure Region
      │
 ┌────┼────┐
 ▼    ▼    ▼
AZ-1 AZ-2 AZ-3
Copy Copy Copy
```

If **AZ-1 fails**, data remains available through the other zones.

---

## Step 3 — Create GRS Storage

Create the third account:

```text
Resource Group: storage-rg
Storage account: atulgrs12345
Performance: Standard
Redundancy: Geo-redundant storage (GRS)
```

Architecture:

```text
PRIMARY REGION
     │
     │ LRS
 ┌───┼───┐
 C1  C2  C3
     │
     │ asynchronous
     │ geo-replication
     ▼
SECONDARY REGION
     │
 ┌───┼───┐
 C4  C5  C6
```

GRS therefore maintains **six copies** overall.

---

# 3. Upload Test Data

Open each storage account:

```text
Storage Account
      ↓
Data storage
      ↓
Containers
      ↓
+ Container
```

Create:

```text
Name: demo
```

Create a local file:

```bash
echo "Azure Storage Redundancy Practical" > test.txt
```

Upload `test.txt` into the `demo` container of each storage account.

You now conceptually have:

```text
test.txt
   │
   ├── LRS → 3 local copies
   │
   ├── ZRS → 3 zone copies
   │
   └── GRS → 3 primary + 3 secondary copies
```

Azure manages these replicas internally; you won't see six separate `test.txt` objects in the portal.

---

# 4. Check Redundancy Using Azure CLI

List your storage accounts:

```bash
az storage account list \
  --resource-group storage-rg \
  --query "[].{Name:name,SKU:sku.name,Location:location}" \
  -o table
```

Example:

```text
Name             SKU             Location
---------------  --------------  ------------
atullrs12345     Standard_LRS    centralindia
atulzrs12345     Standard_ZRS    centralindia
atulgrs12345     Standard_GRS    centralindia
```

Check one account:

```bash
az storage account show \
  --name atulgrs12345 \
  --resource-group storage-rg \
  --query "{Name:name,SKU:sku.name,Primary:primaryLocation,Secondary:secondaryLocation}" \
  -o table
```

For GRS, you can see the **primary and paired secondary locations**.

---

# 5. Change LRS → GRS Practical

Check current SKU:

```bash
az storage account show \
  --name atullrs12345 \
  --resource-group storage-rg \
  --query "sku.name" \
  -o tsv
```

Expected:

```text
Standard_LRS
```

Change redundancy:

```bash
az storage account update \
  --name atullrs12345 \
  --resource-group storage-rg \
  --sku Standard_GRS
```

Check again:

```bash
az storage account show \
  --name atullrs12345 \
  --resource-group storage-rg \
  --query "sku.name" \
  -o tsv
```

Expected:

```text
Standard_GRS
```

Not every redundancy conversion is available for every account/configuration, so for production migrations check the supported conversion path first.

---

# 6. Failure Scenarios

The easiest way to remember them is:

```text
Disk / Server / Rack Failure
          ↓
         LRS ✓

Availability Zone Failure
          ↓
         ZRS ✓

Entire Azure Region Disaster
          ↓
         GRS ✓
```

GRS is aimed at regional disaster recovery, but replication to the secondary region is asynchronous, so it should not be interpreted as guaranteed zero-data-loss replication.

## Final Comparison

| Feature                      | LRS      | ZRS           | GRS                     |
| ---------------------------- | -------- | ------------- | ----------------------- |
| Copies                       | 3        | 3             | 6                       |
| Multiple racks               | ✅        | ✅             | ✅                       |
| Multiple AZs                 | ❌        | ✅             | ❌*                      |
| Multiple regions             | ❌        | ❌             | ✅                       |
| Server failure               | ✅        | ✅             | ✅                       |
| Zone failure protection      | ❌        | ✅             | Depends on architecture |
| Regional disaster protection | ❌        | ❌             | ✅                       |
| Relative cost                | $        | $$            | $$$                     |
| Typical use                  | Dev/Test | Production HA | Disaster Recovery       |

* Standard GRS uses LRS within each region. **GZRS** combines zone redundancy in the primary region with geo-replication.

### Interview shortcut

**LRS = Local**, **ZRS = Zones**, **GRS = Geography/Regions**.

For a fuller Azure Storage lab, the natural next practical is **RA-GRS and GZRS**, followed by a simulated **storage-account failover** exercise.
