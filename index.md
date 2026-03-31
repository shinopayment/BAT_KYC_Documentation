# KYC API Documentation

## Overview

The KYC (Know Your Customer) API provides endpoints for submitting and querying KYC Name Reconciliation requests.

**Base URL (Production):** `https://mapi.oppay88.com/api/KYC`

**Base URL (Staging):** `https://mapi.battt.io/api/KYC`

---

## Authentication & Encryption

All requests must be wrapped in an encrypted `APIRequestModel` envelope. The platform uses **AES-256-CBC** for payload encryption and **RSA-SHA1** for request signing.

### Request Envelope

Every API call must be sent as the following JSON structure:

```json
{
  "merchantCode": "YOUR_MERCHANT_CODE",
  "data": "<AES encrypted + base64 encoded payload>",
  "signature": "<RSA-SHA1 signature + base64 encoded>"
}
```

---

### Step 1 — Serialize Your Payload

Serialize your request object to a JSON string. This plain-text JSON is what gets encrypted and signed.

```json
{
  "merchantReferenceId": "REF001",
  "merchantCode": "MCH001",
  "currencyCode": "THB",
  "bankCode": "KBANK",
  "bankAccountNumber": "1234567890",
  "notifyUrl": "https://yoursite.com/notify",
  "createdBy": "merchant-system"
}
```

---

### Step 2 — Encrypt the Payload (AES-256-CBC)

- **Algorithm:** AES-256-CBC, PKCS7 padding
- **Key:** Derived from your `AESKey` using PBKDF1 (`Rfc2898DeriveBytes`) with your `AESKey` as both the password and the salt, extracting 32 bytes (256 bits)
- **IV:** Randomly generated 16 bytes per request
- **Output format:** Base64(`IV bytes` + `Ciphertext bytes`) — the IV is **prepended** to the ciphertext before encoding

**C# Example**
```csharp
public static string EncryptRandomIV(string plainText, string passwordHash, string saltKey)
{
    using var aes = new AesCryptoServiceProvider
    {
        Key = new Rfc2898DeriveBytes(passwordHash, Encoding.UTF8.GetBytes(saltKey)).GetBytes(256 / 8),
        Mode = CipherMode.CBC,
        Padding = PaddingMode.PKCS7
    };
    var input = Encoding.UTF8.GetBytes(plainText);
    aes.GenerateIV();
    var iv = aes.IV;

    using var cipherStream = new MemoryStream();
    cipherStream.Write(iv); // IV is written directly to the plain stream (not encrypted)
    using (var cryptoStream = new CryptoStream(cipherStream, aes.CreateEncryptor(aes.Key, iv), CryptoStreamMode.Write))
    using (var binaryWriter = new BinaryWriter(cryptoStream))
    {
        binaryWriter.Write(input);
        cryptoStream.FlushFinalBlock();
    }
    return Convert.ToBase64String(cipherStream.ToArray());
}
```

---

### Step 3 — Sign the Payload (RSA-SHA1)

Sign the **plain-text JSON string** (before encryption) using your **RSA private key** with SHA1 digest.

- **Algorithm:** RSA with SHA1 digest (PKCS#1 v1.5)
- **Input:** The plain-text JSON string from Step 1 (UTF-8 encoded bytes)
- **Key:** Your merchant RSA private key (Base64-encoded DER format)
- **Output:** Base64-encoded signature

**C# Example**
```csharp
public static string GenerateSignature_SHA1(string original, string privateKey)
{
    RsaDigestSigner signer = new RsaDigestSigner(new Sha1Digest());
    byte[] keyBytes = Convert.FromBase64String(privateKey);
    AsymmetricKeyParameter keyParameter = PrivateKeyFactory.CreateKey(keyBytes);
    signer.Init(true, keyParameter);

    var buf = Encoding.UTF8.GetBytes(original);
    signer.BlockUpdate(buf, 0, buf.Length);
    return Convert.ToBase64String(signer.GenerateSignature());
}
```

**Python Example**
```python
import base64
from Crypto.Signature import pkcs1_15
from Crypto.Hash import SHA1
from Crypto.PublicKey import RSA

def sign(plain_text: str, private_key_base64: str) -> str:
    key = RSA.import_key(base64.b64decode(private_key_base64))
    h = SHA1.new(plain_text.encode())
    return base64.b64encode(pkcs1_15.new(key).sign(h)).decode()
```

---

### Step 4 — Assemble and Send

```csharp
var payload = JsonConvert.SerializeObject(requestDto);
var encryptedData = EncryptRandomIV(payload, merchantAESKey, merchantAESKey); // AESKey used as both password and salt
var signature = GenerateSignature_SHA1(payload, merchantRSAPrivateKey);

var envelope = new
{
    merchantCode = "MCH001",
    data = encryptedData,
    signature = signature
};
// POST envelope as JSON to the endpoint
```

---

### Credentials Required

Contact the platform to obtain the following credentials for your merchant account:

| Credential | Description |
|---|---|
| `MerchantCode` | Your unique merchant identifier |
| `AESKey` | Symmetric key for payload encryption/decryption |
| `MerchantPrivateKey` | RSA private key for signing requests (kept secret by merchant) |
| `MerchantPublicKey` | RSA public key registered with the platform for signature verification |

---

## Endpoints

### 1. Create KYC Name Recon

Submit a new KYC Name Reconciliation request.

**Endpoint**
```
POST /api/KYC/Create/NameRecon
```

**Request Body**

The request body is an encrypted `APIRequestModel`. Once decrypted, the payload maps to the following fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `MerchantReferenceId` | `string` | Yes | Merchant's own reference ID for this request |
| `MerchantCode` | `string` | Yes | Unique merchant identifier |
| `CurrencyCode` | `string` | Yes | ISO currency code (e.g. `THB`, `VND`) |
| `BankCode` | `string` | Yes | Target bank code (e.g. `KBANK`) |
| `BankAccountNumber` | `string` | Yes | Bank account number to verify |
| `NotifyUrl` | `string` | Yes | Callback URL for result notification |
| `CreatedBy` | `string` | Yes | Identifier of the requester |

**Response**

| HTTP Status | Description |
|---|---|
| `200 OK` | Request accepted. Returns `KYCNameReconCreateResponseDTO`. |
| `400 Bad Request` | Invalid request model or validation failure. |
| `500 Internal Server Error` | Unexpected server error. |

**Success Response Body (`200 OK`)**

After request has been created successfully, we will directly return the following response
```json
{
  "isSuccess": true,
  "successCode": 200,
  "data": {
    "orderId": "KYC20260317000001",
    "merchantReferenceId": "REF001",
    "status": "Pending",
    "createdAt": "2026-03-17T10:00:00.000Z",
    "updatedAt": "2026-03-17T10:01:00.000Z",
    "errorCode": null,
    "errorDescription": null
  }
}
```

After name scraped successfully, we will callback to the request's notify url with the following response
```json
{
  "isSuccess": true,
  "successCode": 200,
  "data": {
    "orderId": "KYC20260317000001",
    "merchantReferenceId": "REF001",
    "accountHolderName": "John Doe",
    "status": "Success",
    "createdAt": "2026-03-17T10:00:00.000Z",
    "updatedAt": "2026-03-17T10:01:00.000Z",
    "errorCode": null,
    "errorDescription": null
  }
}
```

**Response Fields**

| Field | Type | Nullable | Description |
|---|---|---|---|
| `orderId` | `string` | No | System-generated order ID (e.g. `KYC20260317000001`) |
| `merchantReferenceId` | `string` | No | Merchant's reference ID echoed back |
| `accountHolderName` | `string` | No | The verified account holder name returned from the bank |
| `status` | `string` | No | Current status of the KYC request |
| `createdAt` | `datetime` | No | Timestamp when the request was created |
| `updatedAt` | `datetime` | Yes | Timestamp of last update, null if not yet processed |
| `errorCode` | `string` | Yes | Error code if the request failed, otherwise null |
| `errorDescription` | `string` | Yes | Human-readable error description, otherwise null |

**Error Response Body (`400 Bad Request`)**
```json
{
  "isSuccess": false,
  "errorMessage": "..."
}
```

---

### 2. Requery KYC Name Recon

Query the result of a previously submitted KYC Name Reconciliation request.

**Endpoint**
```
POST /api/KYC/Requery/NameRecon
```

**Request**

The request is an encrypted `APIRequestModel` passed as a query parameter. Once decrypted, the payload maps to the following fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `MerchantReferenceId` | `string` | Yes | The reference ID used in the original Create request |
| `MerchantCode` | `string` | Yes | Unique merchant identifier |

**Response**

| HTTP Status | Description |
|---|---|
| `200 OK` | Query successful. Returns `KYCNameReconRequeryResponseDTO`. |
| `400 Bad Request` | Invalid request model or validation failure. |
| `500 Internal Server Error` | Unexpected server error. |

**Success Response Body (`200 OK`)**
```json
{
  "isSuccess": true,
  "successCode": 200,
  "data": {
    "orderId": "KYC20260317000001",
    "merchantReferenceId": "REF001",
    "scrapedBankAccountName": "John Doe",
    "status": "Completed",
    "createdAt": "2026-03-17T10:00:00.000Z",
    "updatedAt": "2026-03-17T10:01:00.000Z",
    "errorCode": null,
    "errorDescription": null
  }
}
```

**Response Fields**

| Field | Type | Nullable | Description |
|---|---|---|---|
| `orderId` | `string` | No | System-generated order ID (e.g. `KYC20260317000001`) |
| `merchantReferenceId` | `string` | No | Merchant's reference ID echoed back |
| `scrapedBankAccountName` | `string` | No | The bank account name retrieved during reconciliation |
| `status` | `string` | No | Current status of the KYC request |
| `createdAt` | `datetime` | No | Timestamp when the request was originally created |
| `updatedAt` | `datetime` | Yes | Timestamp of last update, null if not yet processed |
| `errorCode` | `string` | Yes | Error code if the request failed, otherwise null |
| `errorDescription` | `string` | Yes | Human-readable error description, otherwise null |

---

## Error Codes

### HTTP Errors

| HTTP Status | Description |
|---|---|
| `400` | Bad request — invalid or missing parameters |
| `500` | Internal server error — contact support if persistent |

### Business Error Codes

These are returned in the `errorCode` and `errorDescription` fields of the response body when a business-level failure occurs.

| Error Code | Description |
|---|---|
| `BAT10020` | No KYC Name Recon Order Able To Be Found |
| `ERR10001` | Bank Account Info Not Found |
| `ERR10002` | Robot Exception |
| `ERR10003` | Database Exception |
| `ERR10004` | Failed To Login |
| `ERR10005` | Failed To Retrieve Account Holder Name |
| `ERR99999` | Robot General System Error |

---

## Notes

- All request payloads must be encrypted before submission. Plain-text payloads will be rejected.
- The `NotifyUrl` provided during Create will receive a server-to-server callback once the name reconciliation result is available.
- Requery can be used to poll for the result if the callback was not received.
