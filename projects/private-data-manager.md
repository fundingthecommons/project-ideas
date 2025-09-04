# Private Data Manager (PDM)

**Nillion docs and resources to reference**

- [Nillion Private Storage Docs](https://docs.nillion.com/build/private-storage/overview)
    - Collection Type: [User Owned Collections](https://docs.nillion.com/build/private-storage/overview#collection-types)
    - [User Owned Collections Node.js Quickstart](https://docs.nillion.com/build/private-storage/quickstart)
- [secretvaults-ts GitHub](https://github.com/NillionNetwork/secretvaults-ts)
    - [TypeScript SecretVaultUserClient TypeDoc](https://nillion.pub/secretvaults-ts/classes/SecretVaultUserClient.html)

## The Problem

Users need a friendly interface for managing their private data and data permissions in Nillion's User Owned Collections.

The Nillion TypeScript [SecretVaultUserClient](https://docs.nillion.com/build/private-storage/secretvaults) has methods like `createData()`, `grantAccess()`, and `revokeAccess()`, but users need to be developers to use them. The [User Owned Collection Quickstart](https://docs.nillion.com/build/private-storage/quickstart) shows a backend only flow where a user has to trust their app with their keypair, which is not ideal.

## What Should Happen

Here's how User Owned Collections should work:

1. App creates a User Owned Collection
2. User logs in, app learns their DID ← missing UI that holds the user's keypair and DID
3. App grants the user's DID permission to add data to the collection ← missing UI 
4. **User adds their data** ← missing UI
5. **User grants app permissions to access their data** (read-only, or full CRUD) ← missing UI
6. **User can change permissions later** ← missing UI

## Solution: Browser Extension

Build a browser extension that wraps the SecretVaultUserClient methods with a simple UI.

## What It Does

**Identity Management**

- Generate and show user's DID
    - Optionally have some sort of social wallet like Privy wrapped around this
    - Privy provides [Chrome Extension Docs + a Starter Repo](https://docs.privy.io/recipes/react/chrome-extension#content-security-policy-guidelines)
- Let apps discover the user's DID
- Store keypair securely in browser extension

**Data Management**

- `listDataReferences()` → "My Data" dashboard
- `createData()` → forms to add data
- `readData()` → view your documents
- `deleteData()` → remove data

**Permission Control**

- `grantAccess()` → give apps specific permissions
- `revokeAccess()` → take permissions away
- Dashboard showing which apps access what data

## Main Screens

- **My DID**: Shows identity, connected apps
- **My Data**: All collections and documents
- **App Permissions**: What each app can access
- **Add Data**: Forms for new documents

## Tech Stack

- Browser extension (Chrome/Firefox)
- [secretvaults-ts library](https://github.com/NillionNetwork/secretvaults-ts)
- Extension secure storage
- postMessage for app communication

## Demo Flow

1. Install extension, create DID
2. Health app requests access to user's DID
3. User adds health data through extension
4. User grants health app read-only permission
5. Health app reads data (can't modify)
6. User revokes permission later through extension

## What Gets Built

**MVP**

- **Browser Extension Setup**
  - Chrome/Firefox extension with secure keypair storage
  - DID generation and display on first install
  - Extension popup showing user's DID and connected status

- **Identity & App Connection**
  - postMessage API for websites to discover user's DID
  - Permission prompt when apps request DID access
  - Connected apps list showing which sites know your DID

- **Data Management Interface**
  - "My Data" dashboard listing all collections and documents via `listDataReferences()`
  - Add data forms that call `createData()` with user-friendly input fields
  - Data viewer to display stored documents using `readData()`
  - Delete button for each document using `deleteData()`

- **Basic Permission Controls**
  - Permission granting UI that wraps `grantAccess()`
  - Read-only vs full access permission levels
  - Visual indicators showing which apps have access to which data

**Full Demo**

- Complete permission management dashboard with revoke capabilities
- Live demo app (e.g., health tracker) showing the full flow
- Permission request handling with approve/deny UI
- Activity log showing all permission changes
- Data access audit trail

## Why This Matters

Creating a Private Data Manager makes privacy preserving User Owned Collections usable by regular people. Users get real control over their private data, and apps can still request the access they need. It fills the missing UX gap that's blocking User Owned Collections adoption.