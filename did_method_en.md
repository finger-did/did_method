# DID Method: finger

## 1. Introduction

The **finger** DID method is a Decentralized Identifier (DID) method implemented in the F-DID project. It is a DID server and client module written in Go, providing functionality for creating, registering, resolving, and revoking DID documents.

## 2. Purpose and Use Cases

### Purpose

The finger DID method is defined for the following purposes:

- **Standards-Compliant DID Provision:** Provides a practical DID method that conforms to W3C DID Core and DID Resolution standards.
- **Hybrid VDR (Blockchain + IPFS + DB):** The reference implementation of this specification uses a hybrid Verifiable Data Registry (VDR): DID documents are stored on IPFS (content-addressed storage), the resulting content identifier (CID) is anchored on an Ethereum-compatible smart contract for tamper-evidence, and an operator-managed database supports issuance and status tracking. It supports predictable costs and simple deployment in enterprise and financial environments while adding cryptographic tamper-evidence through the on-chain anchor. The design may be extended to other registries in the future; this is non-normative.
- **Cryptographic Signature-Based Integrity:** DID document authenticity can be independently verified through cryptographic signatures in the `proof` field, regardless of the VDR operator.
- **F-DID Ecosystem Integration:** Builds digital identity infrastructure integrated with Finger Co., Ltd.'s F-DID solution and financial/blockchain platforms.
- **B2B DID Infrastructure Provision:** The goal is to provide managed DID services (DID-as-a-Service) to enterprises, financial institutions, and public agencies that have difficulty building and operating DIDs on their own. F-DID is a commercial solution developed by Finger Co., Ltd. based on this DID method and is offered to the B2B market.

### Use Cases

- **B2B Managed DID Service:** Provides DID infrastructure as a SaaS offering to enterprises, financial institutions, and public agencies. Clients can use the DID lifecycle (create, register, resolve, revoke) through F-DID without building or operating DIDs themselves.
- **Enterprise and Financial Identity Management:** Issuance and verification of digital identifiers for users, systems, and services of financial institutions and fintech companies.
- **Blockchain Integration:** Integration with blockchain-based services such as smart contracts, tokenized securities (STO), and NFTs through the `service` endpoint in DID documents.
- **Self-Sovereign Identity (SSI):** Scenarios where individuals and organizations safely present and verify their identity information together with Verifiable Credentials.
- **API-Based Identity Authentication:** Integration of DID-based authentication into existing financial and public systems through REST API for DID registration, lookup, and revocation.

### Target Users

- Financial institutions and fintech companies
- Blockchain-based service providers
- Organizations building enterprise digital identity and authentication systems
- Public and private SSI and Verifiable Credential services

## 3. DID Method Name

The registered method name is `finger`.
A valid DID starts with: `did:finger:`

## 4. DID Syntax

A `did:finger` DID follows this grammar:

**Human-readable:**
```
did:finger:<method-specific-identifier>
```

**Regex:**
```
^did:finger:[A-Za-z0-9_-]+$
```

**ABNF:**
```
did-finger    = "did:finger:" method-specific-id
method-specific-id = 1*( ALPHA / DIGIT / "-" / "_" )
ALPHA         = %x41-5A / %x61-7A   ; A-Z, a-z
DIGIT         = %x30-39              ; 0-9
```

### Method-Specific Identifier

The method-specific identifier is a Base64url-encoded (no padding) UUID string.

**Generation Process:**

1. Generate a UUID v4 (e.g., `5554e07d-6128-4804-a15f-99a4d5579479`)
2. Convert the UUID string to a byte array
3. Encode using Base64url encoding without padding (`base64.RawURLEncoding`, RFC 4648 §5)
4. The resulting string becomes the method-specific identifier

**Rules:**
- **Character Set:** Base64url character set (RFC 4648 §5: A-Z, a-z, 0-9, -, _), no padding
- **Length:** Approximately 48 characters (result of Base64url-encoding, without padding, a 36-character UUID string)
- **Collision Management:** Uniqueness guaranteed by UUID v4
- **Case Sensitivity:** Case-sensitive as it is a Base64url encoding result

**Note:** This specification uses Base64url encoding (RFC 4648 §5, `base64.RawURLEncoding` in Go) without `=` padding. Because the character set (`A-Z`, `a-z`, `0-9`, `-`, `_`) is already URL-safe, no additional percent-encoding is required when transmitting DIDs in URL paths.

**Example:**
```
did:finger:NTU1NGUwN2QtNjEyOC00ODA0LWExNWYtOTlhNGQ1NTc5NDc5
```

## 5. Method Operations

The finger DID method supports the following operations:

### Create

This operation generates and registers a new DID and DID document.

**DID Generation Process:**

1. Select a cryptographic algorithm (currently EC P-256; see Section 7)
2. Generate a key pair (private key/public key) using the selected algorithm
3. Generate a UUID v4 and encode it in Base64url (no padding) to create the method-specific identifier
4. Create a DID in the format `did:finger:<method-specific-identifier>`
5. Construct a DID object with the generated key pair

**DID Document Creation and Registration:**

1. Create a DID document using the generated DID:
   - Include `@context`, `id`, `verificationMethod`, `authentication`, and `assertionMethod` fields
   - Optionally include a `service` field
2. Canonicalize the DID document (excluding the `proof` field) using JCS (JSON Canonicalization Scheme, RFC 8785)
3. Sign the canonicalized bytes with the private key of the DID that created it, using the `ecdsa-jcs-2019` cryptosuite (ECDSA P-256), and construct a `DataIntegrityProof` object that references the signing key via `verificationMethod`
4. Add the `proof` object to the DID document and encode the resulting JSON document in Base64 for API transport
5. Register it via the registration API (the operator stores the document on IPFS, anchors its CID on-chain, and records status in its database — see Section 11):
   - **Endpoint:** `POST /api/v3/did/registDIDDocument`
   - **Authentication:** API Key header (`X-API-Key`) required
   - **Request Body:**
     ```json
     {
       "userId": "<user-id>",
       "userDIDID": "<base64-encoded-did>",
       "didDocument": "<base64-encoded-did-document>"
     }
     ```
6. If an existing DID document exists, it is automatically changed to `Revoked` status, and the new document is stored with `Active` status

**Client Library Usage:**

The Go client library can be used to create DIDs:

```go
// Create DID
didBase64, err := did.CreateDid(keytype)  // keytype: "P256" (currently supported curve)

// Create DID Document
docBase64, err := did.CreateDidDocument(didBase64, nameBase64, descBase64)
```

### Read/Resolve

This operation resolves a DID to retrieve its DID document.

**Public Resolution Endpoint (W3C Standard):**

1. Resolve a DID according to the W3C DID Resolution standard
2. **Endpoint:** `GET /1.0/identifiers/{did}`
   - API Key authentication is not required
   - Searches for DIDs across all consumers
3. Processing:
   - Remove the `did:finger:` prefix from the DID to extract the method-specific identifier
   - Look up the CID anchored on-chain for that identifier and fetch the corresponding DID document from IPFS (see Section 11)
   - Cross-check the DID's status (Active/Revoked) against the operator database
   - If found and Active, return it in W3C DID Resolution Result format
   - If not found, revoked, or no CID is anchored, return a `404 notFound` error
4. Response Format:
   ```json
   {
     "didDocument": { /* DID Document */ },
     "didResolutionMetadata": {
       "contentType": "application/did+ld+json",
       "driver": "f-did-go",
       "driverVersion": "1.0.0"
     },
     "didDocumentMetadata": {
       "versionId": "<IPFS CID>",
       "cid": "<IPFS CID>"
     }
   }
   ```

**Authenticated Query Endpoints:**

1. Query by User ID: `GET /api/v3/did/getDIDDocumentByID?userID=<user-id>`
2. Query by DID ID: `GET /api/v3/did/getDIDDocumentByDIDID?userDIDID=<base64-encoded-did>`
3. API Key authentication required (`X-API-Key` header)
4. Only queries DID documents for a specific consumer

### Update

This operation updates an existing DID document.

**Update Process:**

1. Prepare the DID document to update:
   - Must use the same `id` field as the existing DID
   - Include updated content (verificationMethod, service, etc.)
2. Canonicalize the document (excluding `proof`) using JCS and sign it with the private key of the DID that created it, producing a new `DataIntegrityProof` (`ecdsa-jcs-2019`)
3. Encode the signed DID document in Base64
4. Update using the registration API:
   - **Endpoint:** `POST /api/v3/did/registDIDDocument` (same endpoint as Create)
   - **Authentication:** API Key header required
   - The system automatically finds the existing DID document, changes it to `Revoked` status, and stores the new document with `Active` status

**Important Notes:**

- When updating, all previous versions of the DID document are set to `Revoked` status
- The `id` field of the DID cannot be changed
- **Authentication Requirements:**
  - Both API Key authentication and DID signature verification are required
  - The `proof.verificationMethod` in the DID Document must reference a key present in the current Active DID Document's `verificationMethod` array (and referenced by `authentication`/`assertionMethod`)
  - The `publicKeyJwk` of the referenced `verificationMethod` entry is used to verify the `DataIntegrityProof`

### Deactivate

This operation revokes a DID document so it can no longer be used.

**Revocation Process:**

1. Identify the DID ID to revoke
2. Call the revocation API:
   - **Endpoint:** `POST /api/v3/did/revokeDIDDocument`
   - **Authentication:** API Key header required
   - **Request Body:**
     ```json
     {
       "userDIDID": "<base64-encoded-did>"
     }
     ```
3. Processing:
   - Find the DID in the database
   - Change the DID document status from `Active` to `Revoked`
4. After Revocation:
   - Revoked DIDs return a `404 notFound` error upon resolution
   - Revoked DIDs cannot be updated
   - Revoked DIDs cannot be reactivated

**All operations are protected through API Key-based authentication.**

## 6. DID Document Structure

The finger DID method conforms to the W3C DID Core specification and has the following structure:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1.1",
    "https://www.w3.org/ns/cid/v1",
    "https://w3id.org/security/data-integrity/v2"
  ],
  "id": "did:finger:<method-specific-identifier>",
  "verificationMethod": [
    {
      "id": "did:finger:<method-specific-identifier>#key-1",
      "type": "JsonWebKey",
      "controller": "did:finger:<method-specific-identifier>",
      "publicKeyJwk": {
        "kty": "EC",
        "crv": "P-256",
        "x": "<base64url-encoded-x-coordinate>",
        "y": "<base64url-encoded-y-coordinate>"
      }
    }
  ],
  "authentication": [
    "did:finger:<method-specific-identifier>#key-1"
  ],
  "assertionMethod": [
    "did:finger:<method-specific-identifier>#key-1"
  ],
  "proof": {
    "type": "DataIntegrityProof",
    "cryptosuite": "ecdsa-jcs-2019",
    "created": "<rfc3339-utc-timestamp>",
    "verificationMethod": "did:finger:<method-specific-identifier>#key-1",
    "proofPurpose": "assertionMethod",
    "proofValue": "z<multibase-base58btc-encoded-signature>"
  }
}
```

**Note:** A `service` property (optional, e.g., for blockchain smart-contract endpoints, see Section 12) MAY additionally be included; this is non-normative.

## 7. Supported Cryptographic Algorithms

The finger DID method's DID Documents currently use:

- **Verification method type:** `JsonWebKey`, with key material carried in `publicKeyJwk`
- **Currently supported curve:** EC P-256 (JWK `kty`: `EC`, `crv`: `P-256`)
- **Proof:** `DataIntegrityProof` with cryptosuite `ecdsa-jcs-2019` (ECDSA over P-256, JCS/RFC 8785 canonicalization)

Support for additional curves or key types may be added in the future; this is non-normative.

## 8. DID Resolution

The finger DID method conforms to the W3C DID Resolution standard.

### Resolution Endpoint

```
GET /1.0/identifiers/{did}
```

**Example:**
```
GET /1.0/identifiers/did:finger:NTU1NGUwN2QtNjEyOC00ODA0LWExNWYtOTlhNGQ1NTc5NDc5
```

### Response Format

On success, responds in W3C DID Resolution Result format:

```json
{
  "didDocument": {
    "@context": [
      "https://www.w3.org/ns/did/v1.1",
      "https://www.w3.org/ns/cid/v1",
      "https://w3id.org/security/data-integrity/v2"
    ],
    "id": "did:finger:...",
    ...
  },
  "didResolutionMetadata": {
    "contentType": "application/did+ld+json",
    "driver": "f-did-go",
    "driverVersion": "1.0.0"
  },
  "didDocumentMetadata": {
    "versionId": "<IPFS CID>",
    "cid": "<IPFS CID>"
  }
}
```

### Resolution Errors

Errors are returned in the following cases:

- **400 Bad Request:** Invalid DID format
- **404 notFound:** DID document not found (revoked or does not exist)
- **500 Internal Server Error:** Server internal error

## 9. DID Document

The returned DID Document conforms to DID Core:

- **@context:** A three-element array (`https://www.w3.org/ns/did/v1.1`, `https://www.w3.org/ns/cid/v1`, `https://w3id.org/security/data-integrity/v2`)
- **id:** The resolved DID
- **verificationMethod:** Supported public keys, using the `JsonWebKey` type with key material in `publicKeyJwk` (EC P-256)
- **authentication:** String references (e.g., `#key-1`) to keys used for authentication
- **assertionMethod:** String references to keys used for issuing assertions (currently the same key as `authentication`)
- **service:** Service endpoints (optional)
- **proof:** `DataIntegrityProof` (cryptosuite `ecdsa-jcs-2019`) signature of the DID document

## 10. DID Document Management

The finger DID method provides functionality for registering, querying, and revoking DID documents. All management operations are protected through API Key-based authentication. DID documents are stored on IPFS with their content identifiers (CIDs) anchored on-chain, alongside operator-managed database records used for issuance and status tracking (see Section 11).

## 11. Verifiable Data Registry (VDR) and Trust Model

### VDR Architecture

The finger DID method uses a hybrid Verifiable Data Registry (VDR) operated by the F-DID service operator, combining three layers:

- **IPFS (document storage):** The full DID document is stored on IPFS, which produces a content identifier (CID), a cryptographic hash of the document's content.
- **On-chain anchor (integrity/registry):** The CID is anchored on an Ethereum-compatible smart contract (`didRegistry.setCid(keccak256(did), cid)`), providing a tamper-evident, publicly auditable record of which CID is currently associated with a given DID.
- **Operator database:** The F-DID service operator additionally maintains a database for the issuance workflow, consumer/API-Key management, and Active/Revoked status tracking.

This specification is defined on the premise of this chain-anchored, IPFS-backed hybrid VDR. The design may be extended to other registries (e.g., other distributed ledgers) in the future; this is non-normative.

**Resolution Model:**
- DID resolution is performed through a public HTTP endpoint (`GET /1.0/identifiers/{did}`)
- The resolver reads the CID anchored on-chain for the DID (`didRegistry.getCid(keccak256(did))`), fetches the corresponding DID document from IPFS, and cross-checks Active/Revoked status against the operator database
- All Active DID documents are publicly resolvable without authentication

**Trust Model:**
- **Availability Dependency:** DID resolution depends on the availability and operational integrity of the F-DID service operator's infrastructure (blockchain node, IPFS node/pinning service, and database)
- **Integrity Protection:** Even though the operator's database is centralized, DID document integrity is doubly protected: the on-chain CID anchor detects any substitution of the IPFS-stored document, and the cryptographic signature in the `proof` field independently verifies the document's authenticity
- **Trust Assumptions:** 
  - Clients must trust the F-DID service operator to:
    - Maintain blockchain, IPFS, and database availability
    - Return accurate DID documents for Active DIDs
    - Not arbitrarily revoke or modify DID documents without proper authorization
  - DID document authenticity is independently verifiable through cryptographic signature verification and CID/content matching, regardless of VDR operator actions

**Mitigation Strategies:**
- **Cryptographic Verification:** All DID documents include cryptographic signatures that can be verified independently of the VDR
- **Content-Addressed Integrity:** Because IPFS content is addressed by its CID, and the CID is anchored on-chain, any tampering with the stored document content is detectable by recomputing and comparing the CID
- **Audit Logging:** State changes (Create/Update/Deactivate) are logged for audit purposes
- **Signature-Based Authorization:** Updates and deactivations require valid cryptographic signatures from DID controllers, preventing arbitrary modifications by the VDR operator

**Residual Risks:**
- **Service Availability:** DID resolution may be unavailable if the blockchain node, IPFS node, or F-DID service is offline
- **Operator Compromise:** If the VDR operator is compromised, they could:
  - Deny resolution (DoS)
  - Return incorrect or stale data (detectable through CID mismatch or signature verification)
  - However, they cannot forge valid signatures without controller private keys, nor rewrite the on-chain CID anchor without the registry contract's authorized signer key
- **Database Breach:** A database breach could expose status/metadata, but would not allow unauthorized updates without controller private keys and authorized on-chain write access

**Operational Considerations:**
- The F-DID service operator maintains high availability infrastructure for the blockchain node, IPFS node/pinning, and database
- Database backups and disaster recovery procedures are maintained
- API Key management follows security best practices
- Service status and maintenance windows are communicated to users

## 12. Blockchain Integration

Blockchain integration is part of the finger DID method's core VDR architecture (see Section 11): the CID of each DID document stored on IPFS is anchored on an Ethereum-compatible smart contract (`didRegistry.setCid(keccak256(did), cid)`), providing a tamper-evident, publicly verifiable record of the document's current state. This is distinct from the optional `service`-endpoint based smart-contract integration described in Section 2/6, which lets a DID subject reference its own external smart contracts (e.g., for STO/NFT services).

## 13. Governance

- **Authority:** F-DID Project Team
- **DID Acquisition Process:** DID creation and registration available through API Key authentication
- **Maintenance:** Continuously managed and improved by the F-DID project

## 14. Privacy & Security Considerations

### Public Key and Private Key Handling

**Private Key Management:**

- Private keys are used for DID document creation and signing, and are never included in DID Documents
- Private keys are generated and managed on the client side
- Private keys can be encrypted when stored (e.g., using AES/CBC/PKCS7 encryption)
- Key files can be encrypted using password-based encryption:
  - Encryption keys are generated from passwords using SHA-256 hashing
  - Private keys are encrypted using AES-256-CBC
- Protection features are provided to clear keys from memory immediately after use

**Public Key Exposure:**

- Public keys are stored in the `verificationMethod` field of DID Documents as a JWK (`publicKeyJwk`, EC P-256)
- Public keys are public information and can be queried by anyone
- Public keys are used for signature verification and cannot be used to generate signatures without the private key

**Key Rotation:**

- New verificationMethods can be added by updating the DID document
- Existing keys can be maintained while adding new keys
- If a key is compromised, the DID document can be updated with a new key, and the old key can be removed

### Signing and Verification Security

**Signature Generation:**

- All DID documents are signed with the private key of the DID that created them
- **Canonicalization and Serialization:**
  - The DID document (excluding the `proof` field) is canonicalized using JCS (JSON Canonicalization Scheme, RFC 8785)
  - The resulting canonical bytes are used as the input for cryptographic signing
  - Note: JCS produces a single deterministic byte-serialization for a given JSON value (fixed key ordering, fixed number formatting), which removes the field-ordering ambiguity of ad-hoc JSON serialization
- **Proof Structure:**
  - The `proof` (`DataIntegrityProof`) includes the following information:
    - `type`: Always `DataIntegrityProof`
    - `cryptosuite`: `ecdsa-jcs-2019`
    - `verificationMethod`: The DID URL (with fragment, e.g., `#key-1`) of the key that created the signature
    - `created`: Signature creation time (RFC3339, UTC)
    - `proofPurpose`: `assertionMethod`
    - `proofValue`: Multibase (base58btc, `z`-prefixed) encoded signature value
- **Signature Process:**
  1. Create DID document structure without `proof` field
  2. Canonicalize the document using JCS (RFC 8785)
  3. Sign the canonical bytes using the private key (ECDSA P-256)
  4. Create proof object (`DataIntegrityProof`) with `proofValue` and metadata
  5. Add proof to the DID document
- The signature algorithm is determined by the proof's `cryptosuite`; the currently supported cryptosuite is `ecdsa-jcs-2019` (ECDSA over the P-256 curve, SHA-256 hash, JCS-canonicalized input). Support for additional cryptosuites/curves may be added in the future; this is non-normative.

**Signature Verification:**

- Signature verification must be performed when receiving DID documents
- **Verification Process:**
  1. Extract the `proof` field from the DID document
  2. Create a copy of the DID document without the `proof` field
  3. Canonicalize the proof-free document using the same method as signing (JCS, RFC 8785)
  4. Resolve the key referenced by `proof.verificationMethod` (e.g., `#key-1`) in the DID Document's `verificationMethod` array, and extract its `publicKeyJwk` (EC P-256)
  5. Decode the `proofValue` from multibase (base58btc, `z`-prefixed) encoding
  6. Verify the signature using the public key and the canonicalized bytes
- DID documents are considered invalid if signature verification fails
- Signature verification is performed using the public key (`publicKeyJwk`) of the `verificationMethod` entry referenced by `proof.verificationMethod`

**Forgery Prevention:**

- Only the DID owner who possesses the private key can create or update DID documents
- DID document integrity is guaranteed through signature verification
- DID documents stored in the database are only stored after signature verification

### State Management Security

**Active/Revoked State Management:**

- DID documents are managed in `Active` or `Revoked` states
- State changes are protected through API Key authentication
- Revoked DIDs return a `404 notFound` error upon resolution, clearly indicating they can no longer be used
- Revoked DIDs cannot be reactivated

**Existing DID Document Handling:**

- When updating a DID document, existing documents are automatically changed to `Revoked` status
- This enables version management and history tracking of DID documents
- Revoked versions of DID documents are retained in the database but are not returned upon resolution

### Authentication and Access Control

**API Key Authentication:**

- All DID management operations (Create, Update, Deactivate) require API Key-based authentication
- API Keys are transmitted through the HTTP header `X-API-Key`
- API Keys are issued per consumer (organization/user) and can only manage DIDs for that consumer
- **Authentication Model:**
  - **Create/Update/Deactivate operations:** Require both:
    1. Valid API Key authentication (via `X-API-Key` header) - establishes consumer identity and access rights
    2. Valid cryptographic signature in the DID Document's `proof` field - establishes DID controller authorization
  - The API Key controls **which consumer** can submit operations, while the DID signature proves **who controls the DID**
  - Both must be valid for the operation to succeed
- **Verification Key Selection:**
  - For signature verification, the key referenced by `proof.verificationMethod` (e.g., `#key-1`) is used
  - When multiple keys exist, the controller is responsible for ensuring the signing key is present in the `verificationMethod` array and referenced by `authentication`/`assertionMethod` as appropriate
- If an API Key is compromised, it must be immediately revoked and a new key issued

**Public Resolution API:**

- The DID resolution API (`GET /1.0/identifiers/{did}`) is publicly accessible
- This is to comply with the W3C DID Resolution standard
- DID documents themselves are public information and can be queried without authentication

**Rate Limiting and DoS Protection:**

- Rate limiting is applied to API requests
- DoS protection features are provided to protect the service from excessive requests
- Requests are logged in audit logs for tracking

### Data Minimization and Privacy

**DID Document Content:**

- DID Documents contain only minimal information:
  - DID identifier (`id`)
  - Public keys (`verificationMethod`)
  - Authentication information (`authentication`)
  - Service endpoints (`service`, optional)
- Personal information (PII) or sensitive data is not stored in DID Documents
- Private keys are never included in DID Documents

**Privacy Considerations:**

- DID identifiers are UUID-based, so personal information cannot be inferred from the identifier itself
- DID resolution is publicly available, so the existence of a DID can be confirmed
- Sensitive information should be stored in Verifiable Credentials, not in DID Documents
- Service endpoints should not contain sensitive information

**GDPR and Right to be Forgotten:**

- The finger DID method stores minimal data: only DID identifiers and public keys
- No Personally Identifiable Information (PII) is stored in DID Documents
- When a DID is revoked, it becomes unresolvable (returns `404 notFound`), effectively removing it from active use
- Revoked DID documents are retained in the database for audit and integrity purposes, but are not accessible through public resolution
- Controllers who wish to completely remove their DID data should contact the service provider for data deletion requests
- The method supports data minimization principles by storing only essential cryptographic and identifier information

### Transport Security

**TLS/HTTPS Encryption:**

- All API communications are encrypted through TLS
- Secure communication is guaranteed through HTTPS
- TLS version 1.2 or higher must be used

**Encryption Strength:**

- Supported cryptographic algorithms follow industry standards:
  - ECDSA: Uses the P-256 curve (see Section 7 for the currently supported cryptosuite)
- Weak cryptographic algorithms are not used

### Additional Security Recommendations

**Implementer Recommendations:**

- Private keys must be kept secure (use of Hardware Security Modules (HSM) recommended)
- API Keys should be stored in environment variables or secure key stores
- Regular key rotation is recommended
- Changes should be carefully reviewed when updating DID documents
- Logging and appropriate handling should be in place when signature verification fails

**Privacy Recommendations:**

- Do not include unnecessary information in DID Documents
- Do not include personal information in service endpoint URLs
- Use Verifiable Credentials to manage sensitive information
- Clearly inform users about the public nature of DIDs

## 15. Implementation Information

The finger DID method is implemented in the Go programming language. This section provides information for developers who wish to implement DID Resolvers or other software that interacts with the `did:finger` method.

### Implementing a DID Resolver

A DID Resolver for the `did:finger` method can be implemented in any programming language by following these steps:

**1. DID Format Validation:**

Validate that the DID follows the format `did:finger:<method-specific-identifier>` where:
- The method-specific identifier is a Base64url-encoded (RFC 4648 §5, no padding) UUID string
- Character set: A-Z, a-z, 0-9, -, _
- Length: approximately 48 characters

**2. Resolution Process:**

Implement the resolution process:

1. Extract the method-specific identifier by removing the `did:finger:` prefix
2. Query the resolution endpoint: `GET /1.0/identifiers/{did}`
3. The endpoint returns a W3C DID Resolution Result:
   ```json
   {
     "didDocument": { /* DID Document */ },
     "didResolutionMetadata": {
       "contentType": "application/did+ld+json",
       "driver": "f-did-go",
       "driverVersion": "1.0.0"
     },
     "didDocumentMetadata": {
       "versionId": "<IPFS CID>",
       "cid": "<IPFS CID>"
     }
   }
   ```
4. Handle error responses:
   - `400 Bad Request`: Invalid DID format
   - `404 notFound`: DID document not found (revoked or does not exist)
   - `500 Internal Server Error`: Server internal error

**3. Implementation Guide:**

- Implement DID format validation logic (check for `did:finger:` prefix)
- Implement method-specific identifier extraction logic (remove `did:finger:` prefix)
- Use HTTP client to call the resolution endpoint
- Parse the response in W3C DID Resolution Result format
- Handle error responses (404, 500, etc.)

### API Endpoints for Integration

**Public Resolution Endpoint (No Authentication Required):**

- **Endpoint:** `GET /1.0/identifiers/{did}`
- **Purpose:** Resolve any `did:finger` DID to retrieve its DID document
- **Authentication:** None required (public endpoint)
- **Example:**
  ```bash
  curl "https://your-server.com/1.0/identifiers/did:finger:NTU1NGUwN2QtNjEyOC00ODA0LWExNWYtOTlhNGQ1NTc5NDc5"
  ```

**Management Endpoints (API Key Authentication Required):**

These endpoints require API Key authentication and are used for DID document management:

- **Register/Update:** `POST /api/v3/did/registDIDDocument`
  - Header: `X-API-Key: <your-api-key>`
  - Body: JSON with `userId`, `userDIDID`, `didDocument`
  
- **Get by User ID:** `GET /api/v3/did/getDIDDocumentByID?userID=<user-id>`
  - Header: `X-API-Key: <your-api-key>`
  
- **Get by DID ID:** `GET /api/v3/did/getDIDDocumentByDIDID?userDIDID=<base64-encoded-did>`
  - Header: `X-API-Key: <your-api-key>`
  
- **Revoke:** `POST /api/v3/did/revokeDIDDocument`
  - Header: `X-API-Key: <your-api-key>`
  - Body: JSON with `userDIDID`

**Note:** API Keys are issued by the F-DID service provider. Contact the service provider to obtain API Keys for accessing management endpoints.

### Creating DIDs Programmatically

**DID Generation Algorithm:**

1. Generate a UUID v4 (e.g., `5554e07d-6128-4804-a15f-99a4d5579479`)
2. Convert the UUID string to a byte array
3. Encode using Base64url encoding without padding (`base64.RawURLEncoding`, RFC 4648 §5)
4. Prepend `did:finger:` to create the full DID

**Example Implementation (Go):**

```go
package main

import (
    "encoding/base64"
    "fmt"
    "github.com/google/uuid"
)

func GenerateFingerDID() (string, error) {
    // Generate UUID v4
    uuidObj, err := uuid.NewRandom()
    if err != nil {
        return "", err
    }
    uuidStr := uuidObj.String()
    
    // Convert to bytes and Base64url encode (no padding)
    uuidBytes := []byte(uuidStr)
    methodSpecificId := base64.RawURLEncoding.EncodeToString(uuidBytes)
    
    // Create DID
    did := fmt.Sprintf("did:finger:%s", methodSpecificId)
    return did, nil
}
```

### DID Document Structure

When creating or updating DID documents, ensure they conform to the following structure:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1.1",
    "https://www.w3.org/ns/cid/v1",
    "https://w3id.org/security/data-integrity/v2"
  ],
  "id": "did:finger:<method-specific-identifier>",
  "verificationMethod": [
    {
      "id": "did:finger:<method-specific-identifier>#key-1",
      "type": "JsonWebKey",
      "controller": "did:finger:<method-specific-identifier>",
      "publicKeyJwk": {
        "kty": "EC",
        "crv": "P-256",
        "x": "<base64url-encoded-x-coordinate>",
        "y": "<base64url-encoded-y-coordinate>"
      }
    }
  ],
  "authentication": [
    "did:finger:<method-specific-identifier>#key-1"
  ],
  "assertionMethod": [
    "did:finger:<method-specific-identifier>#key-1"
  ],
  "proof": {
    "type": "DataIntegrityProof",
    "cryptosuite": "ecdsa-jcs-2019",
    "created": "<rfc3339-utc-timestamp>",
    "verificationMethod": "did:finger:<method-specific-identifier>#key-1",
    "proofPurpose": "assertionMethod",
    "proofValue": "z<multibase-base58btc-encoded-signature>"
  }
}
```

### Supported Cryptographic Algorithms

When implementing signature verification, support:

- **Verification method type:** `JsonWebKey` with `publicKeyJwk`
- **Currently supported curve:** EC P-256
- **Proof cryptosuite:** `ecdsa-jcs-2019` (ECDSA over P-256, JCS/RFC 8785 canonicalized input, SHA-256 hash)

Support for additional cryptosuites/curves may be added in the future; this is non-normative.

### Universal Resolver Integration

To integrate with Universal Resolver:

1. Implement a driver that follows the Universal Resolver driver specification
2. The driver should handle resolution requests for `did:finger` DIDs
3. Return results in W3C DID Resolution Result format
4. Handle errors according to DID Resolution specification

**Driver Metadata:**

- Driver name: `f-did-go`
- Version: `1.0.0`
- Resolution endpoint format: `GET /1.0/identifiers/{did}`

### Implementation Libraries

**Official Implementation:**

The finger DID method is implemented in Go, providing:

- Server implementation with REST API endpoints
- Client library for DID creation and management
- DID Resolver implementation compatible with W3C standards

**For Third-Party Developers:**

Third-party developers can implement support for the `did:finger` method by:

1. Implementing DID format validation according to this specification
2. Implementing resolution by calling the public resolution endpoint
3. Supporting the cryptographic algorithms listed above
4. Following W3C DID Core and DID Resolution specifications

**Note:** While the reference implementation is available, third-party implementations in other languages are encouraged and supported as long as they conform to this specification.

### Contact for Implementation Support

For questions about implementing support for the `did:finger` method, please contact:

- **Email:** youngseoka@finger.co.kr
- **Organization:** Finger Co., Ltd., Technology Research Institute
- **Website:** https://www.finger.co.kr/homepage/html/main/main-01-01.html?cate=main&sub=01&page=01

**Organization Overview:** Finger Co., Ltd. is a B2C fintech company in Korea providing smart financial platforms and blockchain-based digital asset and identity services. This DID method specification is authored and maintained by the Technology Research Institute of Finger Co., Ltd. F-DID is a DID solution developed and operated by Finger.

The specification document and API endpoints described herein provide all information necessary for independent implementation of DID Resolvers and related software.

## 16. References

### Standards

- [W3C DID Core Specification](https://www.w3.org/TR/did-core/)
- [W3C DID Resolution](https://www.w3.org/TR/did-resolution/)
- [W3C DID Implementation Guide](https://www.w3.org/TR/did-imp-guide/)

## 17. Contact Information

- **Email:** youngseoka@finger.co.kr
- **Organization:** Finger Co., Ltd., Technology Research Institute
- **Website:** https://www.finger.co.kr/homepage/html/main/main-01-01.html?cate=main&sub=01&page=01

Finger Co., Ltd. is a B2C fintech company in Korea. This DID method specification is authored and maintained by the Technology Research Institute of Finger. F-DID is a DID solution developed and operated by Finger.

## 18. Lifecycle

### Versioning

- This specification follows Semantic Versioning
- Breaking changes require a new version and prior notice

### Revocation and Deactivation

- When a DID is revoked, resolution returns a `notFound` error
- Revoked DIDs can no longer be used
- The state of revoked DIDs is marked as `Revoked` in the database

### Change Management

- Issues and Pull Requests are tracked in the project repository
- Major changes are announced to the community

## 19. Test Vectors

The following are examples of valid `did:finger` DIDs, currently resolvable on the reference resolver (e.g., `https://did-dev.fingerservice.co.kr:5070/1.0/identifiers/<did>`):

**Example 1:**
```
did:finger:NTU1NGUwN2QtNjEyOC00ODA0LWExNWYtOTlhNGQ1NTc5NDc5
```

**Example 2:**
```
did:finger:MWY0YzgyMjQtMzNkYy00ZjY5LTg3NDgtZWQ3NWM3Mzc4ZjJm
```

Each DID should resolve to a valid DID Document.

## 20. W3C Registration

This method is registered in the W3C DID Method Registry (method name `finger`).

**Status:** Registered

## 21. Status

This DID method is currently **operational** and registered in the W3C DID Extensions registry.
