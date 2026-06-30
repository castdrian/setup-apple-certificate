# Setup Apple Certificate

GitHub Action that manages Apple certificates and provisioning profiles via the App Store Connect API.

Supports both **App Store Distribution** and **Developer ID** certificates.

Uses a persistent private key and reuses existing valid certificates - only creates new ones when no valid matching certificate exists or when the current one is expiring soon.

## Setup

Add the following secrets to your repository (Settings > Secrets and variables > Actions):

| Secret | Description |
|---|---|
| `APP_STORE_CONNECT_KEY_ID` | API Key ID from App Store Connect (Keys section) |
| `APP_STORE_CONNECT_ISSUER_ID` | Issuer ID from App Store Connect (Keys section) |
| `APP_STORE_CONNECT_PRIVATE_KEY_BASE64` | The `.p8` API key file content, base64-encoded |
| `DISTRIBUTION_PRIVATE_KEY_BASE64` | A persistent RSA private key used for certificate signing, base64-encoded |

The App Store Connect API key must have the **Admin** role. See [Apple's role permissions reference](https://developer.apple.com/help/app-store-connect/reference/role-permissions/).

To generate the distribution private key:

```bash
openssl genrsa -out distribution_key.pem 2048
base64 -w 0 distribution_key.pem
```

Store the base64 output as the `DISTRIBUTION_PRIVATE_KEY_BASE64` secret.

## Usage

```yaml
- uses: kebechet/setup-apple-certificate@v1
  id: cert
  with:
    app-store-connect-key-id: ${{ secrets.APP_STORE_CONNECT_KEY_ID }}
    app-store-connect-issuer-id: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
    app-store-connect-private-key: ${{ secrets.APP_STORE_CONNECT_PRIVATE_KEY_BASE64 }}
    distribution-private-key: ${{ secrets.DISTRIBUTION_PRIVATE_KEY_BASE64 }}
    bundle-identifier: com.example.app
```

This action is designed to be used together with [maui-actions/apple-provisioning](https://github.com/maui-actions/apple-provisioning), which installs the certificate and provisioning profile on the macOS runner for signing:

```yaml
- uses: maui-actions/apple-provisioning@v4
  with:
    certificate: ${{ steps.cert.outputs.p12-certificate-base64 }}
    certificate-passphrase: ${{ steps.cert.outputs.p12-password }}
    bundle-identifiers: com.example.app
    profile-types: IOS_APP_STORE
    app-store-connect-key-id: ${{ secrets.APP_STORE_CONNECT_KEY_ID }}
    app-store-connect-issuer-id: ${{ secrets.APP_STORE_CONNECT_ISSUER_ID }}
    app-store-connect-private-key: ${{ secrets.APP_STORE_CONNECT_PRIVATE_KEY_BASE64 }}
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `app-store-connect-key-id` | yes | | App Store Connect API Key ID |
| `app-store-connect-issuer-id` | yes | | App Store Connect API Issuer ID |
| `app-store-connect-private-key` | yes | | API key (.p8) content, base64-encoded |
| `distribution-private-key` | yes | | Persistent RSA private key, base64-encoded |
| `bundle-identifier` | yes | | App bundle ID (e.g. `com.example.app`) |
| `cert-renewal-buffer-days` | no | `14` | Days before expiry to trigger renewal |
| `certificate-type` | no | `APP_STORE` | Certificate type: `APP_STORE` or `DEVELOPER_ID` |

## Outputs

| Output | Description |
|---|---|
| `p12-certificate-base64` | P12 certificate, base64-encoded |
| `p12-password` | Random password for the P12 |
| `keychain-password` | Random keychain password |
| `provisioning-profile-name` | Name of the provisioning profile |
| `codesign-identity` | Certificate Common Name (e.g. `Apple Distribution: Name (TEAMID)`) |

## How it works

1. Generates a JWT for the App Store Connect API
2. Lists existing certificates (App Store or Developer ID, depending on configuration) and checks if any match the provided private key using RSA public key modulus comparison
3. If a valid matching certificate exists (not expiring within the buffer period), reuses it
4. If no valid match exists, creates a new CSR and submits it to get a new certificate
5. Looks up or creates a provisioning profile linked to the certificate
6. Packages everything into a P12 and writes outputs for downstream steps

## How RSA Key Matching Works

This action uses a **persistent private key** to enable certificate reuse across workflow runs without requiring server-side state or certificate storage.

### Why a Persistent Private Key?

In public key cryptography, a certificate's public key is mathematically derived from its corresponding private key. By persisting the same private key across runs, the action can:

- **Identify existing certificates** that belong to this key
- **Reuse valid certificates** instead of creating duplicates
- **Automatically renew** when the current certificate approaches expiry

### The Matching Algorithm

When the action runs, it:

1. **Extracts the RSA public key parameters** from your persistent private key:
   - **Modulus (n)**: A large integer that forms part of the RSA key pair
   - **Exponent (e)**: Typically 65537 for RSA keys

2. **Downloads all certificates** of the requested type from App Store Connect

3. **For each certificate**, extracts its public key modulus and exponent

4. **Compares bit-for-bit**: If both the modulus and exponent match your private key's parameters, the certificate and private key are a cryptographic pair

5. **Checks expiry**: If the match is valid and not expiring soon (within `cert-renewal-buffer-days`), reuses it; otherwise creates a new one

### Example

```
Your persistent private key → RSA public key (n=123...789, e=65537)
                                     ↓
        ┌──────────────────────────────────────────────┐
        │  App Store Connect Certificates              │
        ├──────────────────────────────────────────────┤
        │  Cert A: (n=999...111, e=65537)  ❌ No match │
        │  Cert B: (n=123...789, e=65537)  ✅ Match!   │
        │  Cert C: (n=456...012, e=65537)  ❌ No match │
        └──────────────────────────────────────────────┘
                                     ↓
                    Cert B is cryptographically paired
                    with your private key → Reuse it!
```

### Benefits

- ✅ **No storage required**: No need to store certificates in secrets
- ✅ **Automatic renewal**: Creates new certificates only when needed
- ✅ **Idempotent**: Multiple runs won't create duplicate certificates
- ✅ **Secure**: The private key remains secret; only public key parameters are compared

## Certificate Types

This action supports two certificate types:

| Type | Description | Use Case | Profile Type | Platform |
|---|---|---|---|---|
| **APP_STORE** (default) | Apple Distribution certificate | Submitting iOS apps to the App Store | `IOS_APP_STORE` | iOS |
| **DEVELOPER_ID** | Developer ID Application certificate | Distributing macOS apps outside the App Store (notarization) | `MAC_APP_DIRECT` | macOS |

To use Developer ID certificates:

```yaml
- uses: kebechet/setup-apple-certificate@v1
  with:
    # ... other inputs ...
    certificate-type: DEVELOPER_ID
```

## Requirements

The runner must have `python3`, `openssl`, and `curl` available. The action installs `PyJWT[crypto]` automatically.

## License

[MIT](LICENSE)
