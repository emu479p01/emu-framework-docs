# Manage Apps, Models, deployments, and licenses

## Purpose

Inspect installed Apps and Models, deploy selected Models between environments, and administer optional offline ISV licenses.

## Audience

Framework administrators. Everything under **Apps & Models** requires `FW_SystemAdminRole`. Building a deployment package and importing one also need Designer permission for the App (see [Manage users and application access](user-security.md)).

## Prerequisites

- A Full backup, including the Designer component, before any deployment or license change. See [Back up databases](backup.md).
- For licensing: the customer ID agreed with the vendor, the vendor's Ed25519 public key delivered through a channel you trust, and the license file the vendor issues.

## Review Apps and Models

Open **Settings → Apps & Models**. The page has three parts.

**Installation & ISV licenses** shows this installation's **Installation ID**, the registered **Customer ID**, the trusted vendors, and the controls described below.

**One card per App** shows the App and its dependencies. A warning reading **Read-only** and the reasons appears when a required license is missing, invalid, not yet valid, or expired. The table lists each Model:

| Column | Meaning |
|---|---|
| Model | The Model name |
| Layer / Source | The Layer (SYS, ISV, LOC, DEV, or CUS) and whether the Model comes from the Designer (`designer`) or from files (`file`) |
| Revision | The first 12 characters of the Model's metadata hash. This is a change fingerprint, not the vendor's product version. File-based Models show **File deployment**. |
| License / Vendor | The license state and the vendor ID, or `not-required` |
| Expires (UTC) | The expiry date of the installed license |

Each App card also has a **Deployment history** with the time, user, and description of recent change sets for the App.

**License audit** lists trust changes, license import successes and rejections, and observed license state changes.

## Deploy selected Models

Use deployment packages to move whole Models between environments. The package format and the HTTP API are described in [Deploy Models and issue ISV licenses](../developer/model-deployment.md).

### Choose a mode

| Mode | Layers | What the destination does |
|---|---|---|
| **Vendor update** | SYS, ISV, LOC | Replaces only the selected Models. The destination's DEV and CUS customizations and every unselected Model are kept. |
| **UAT → Prod** | All layers | Replaces each selected Model with its complete source snapshot, including removing metadata that no longer exists in the source. |

Packages come from Designer metadata only. File-based Apps keep their existing file deployment process, and the framework's `system` App cannot be deployed.

### Build the package (source environment)

1. In the Designer, open the App's menu and choose **Build deployment package**.
2. Pick the mode and select the complete Models to include.
3. Choose **Download package** and keep the JSON file.

Dependencies are never added automatically. Install the required Apps and Models at the destination first, or rebuild the package with them.

### Import and confirm (destination environment)

1. Take a Full backup of the destination.
2. In the Designer, import the package. The **Review Metadata Import** dialog opens with a preview.
3. Read the preview. It starts **collapsed** and shows counts per Model: create, update, and delete totals plus the number of high-risk items. Use **Expand all**, or open a Model, to list each artifact, including where moved or deleted artifacts came from. A banner warns when the package contains executable code or metadata deletions.
4. Resolve anything that blocks the import. Ownership conflicts, missing dependencies, and customizations that are incompatible with the new metadata block the whole deployment.
5. Confirm. The confirmation uses a short-lived preview tied to your user and checks the destination's metadata revision again, so a stale preview is rejected. Run the preview again if it was.

What happens to data: business tables and columns are not dropped when their metadata disappears, and a deployment does not transfer business data, users, the installation identity, trusted vendor keys, or installed licenses. App labels, the default language, and other App-level settings of an existing App are kept; required dependencies are merged.

Scripts are trusted server code. Test vendor updates against representative customer customizations in UAT before production. Do not put passwords or other credentials in metadata or script source. If the machine fails between the writes to `data.db` and `designer.db`, use the Full backup to recover.

## Administer ISV licenses

Licensing is optional. It applies only to ISV-layer Models whose definition names a **License vendor ID** in the Designer's Model dialog. Models without one are never restricted. A normal Model edit or deployment cannot remove or change an installed license requirement.

### Set up this installation

1. In **Apps & Models**, copy the **Installation ID**. It is generated automatically in `designer.db`.
2. Enter the **Customer ID** that you agreed with the vendor and choose **Register customer**. Registration happens once and cannot be changed through the UI.
3. Send the vendor the customer ID, the installation ID, and the App and Model names. Production and UAT are different installations and need separately issued licenses.
4. Confirm the vendor's public key with the vendor through a trusted channel, then enter the **Vendor ID** and the **Vendor public key** (PEM, Ed25519) and choose **Trust vendor**. A registered vendor's key cannot be replaced, and key rotation is not supported.
5. When the vendor sends the license file, choose **Import / renew license** and select the JSON file.

Private keys stay with the vendor and must never be copied to the customer's deployment. The vendor builds license files with the seller CLI described in [Deploy Models and issue ISV licenses](../developer/model-deployment.md).

### Renew a license

Import the new file with **Import / renew license**. A renewal uses the same identifiers, a later expiry, and a higher sequence number than the installed license. It must already be valid when you import it, and an expired file is rejected. A first license dated in the future is accepted but does not allow writes before its start date. The next request after a successful renewal can write; no redeployment or restart is needed. Pages that were already open update their license warning on refresh.

### License states and expiry behavior

| State | Writes allowed |
|---|---|
| `not-required` | Yes |
| `active` | Yes |
| `expiring` (30 days or fewer remain) | Yes; the page warns |
| `not-yet-valid`, `expired`, `missing`, `invalid` | No |

When a required license is not active, the owning App and every App that depends on it become **read-only**, with no grace period. Metadata stays loaded. Users can still read and export data, and administrators can still run backups and manage licenses. Business writes, actions, Functions, imports, attachment changes, and archive processing are refused, and a transactional write is checked again before it commits. An asynchronous action cannot undo external effects it already performed before the license expired. Users see `App '<name>' is read-only: <app>/<model>: <state>`.

### Installation identity

Each installation has its own identity, stored in `designer.db`. **Never clone a Designer database to create another environment:** the copy would carry the same installation identity. Use deployment packages to move metadata between installations. A Full backup is for recovering the same installation. License data, trusted vendors, and the license audit are part of the Designer backup component, so restoring a Designer backup returns the license state from that moment.

### Limits

Offline licensing detects tampering with signed license files. It cannot stop someone who controls the server, source code, database, or clock, and it has no online revocation. It does not sandbox trusted scripts or native extensions.

## Common errors

- **Missing dependencies on import:** install the required Apps or Models, or build a package that includes them, then preview again.
- **"Installation customer is already registered":** the customer ID is fixed once registered.
- **"Vendor key is already registered; key replacement is not supported":** contact the vendor; the key cannot be changed from the UI.
- **A license import is rejected:** read **License audit** for the reason. Check the installation ID, customer, App and Model names, dates, and sequence number against what the vendor issued.

## Related topics

[Deploy Models and issue ISV licenses](../developer/model-deployment.md) · [Understand Apps, Models, and Layers](../developer/app-model-layer.md) · [Back up databases](backup.md) · [Troubleshooting](troubleshooting.md)
