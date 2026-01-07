# DID Method: finger

## 1. Introduction

The **finger** DID method is a Decentralized Identifier (DID) method implemented in the F-DID project. It is a DID server and client module written in Go, providing functionality for creating, registering, resolving, and revoking DID documents.

## 2. DID Method Name

The registered method name is `finger`.
A valid DID starts with: `did:finger:`

## 3. DID Syntax

A `did:finger` DID follows this grammar:

**Human-readable:**
```
did:finger:<method-specific-identifier>
```

**Regex:**
```
^did:finger:[A-Za-z0-9+/=]+$
```

**ABNF:**
```
did-finger    = "did:finger:" method-specific-id
method-specific-id = 1*( ALPHA / DIGIT / "+" / "/" / "=" )
ALPHA         = %x41-5A / %x61-7A   ; A-Z, a-z
DIGIT         = %x30-39              ; 0-9
```

### Method-Specific Identifier

The method-specific identifier is a Base64-encoded UUID string.

**Generation Process:**

1. Generate a UUID v4 (e.g., `3d3e4bdd-9adac-04a83-ba37-0f67f69462fd2`)
2. Convert the UUID string to a byte array
3. Encode using Base64 standard encoding (StdEncoding)
4. The resulting string becomes the method-specific identifier

**Rules:**
- **Character Set:** Base64 standard character set (A-Z, a-z, 0-9, +, /, =)
- **Length:** Approximately 44-48 characters (result of Base64 encoding a UUID)
- **Collision Management:** Uniqueness guaranteed by UUID v4
- **Case Sensitivity:** Case-sensitive as it is a Base64 encoding result

**Note:** This specification uses standard Base64 encoding (RFC 4648) which includes `+`, `/`, and `=` padding characters. Implementers should ensure proper URL encoding when transmitting DIDs in URL paths to avoid routing issues.

**Example:**
```
did:finger:ZDNlNGJkZDktYWRhYy00YTgzLWJhMzctMGY2N2Y2OTQ2ZmQy
```

## 4. Method Operations

The finger DID method supports the following operations:

### Create

This operation generates and registers a new DID and DID document.

**DID Generation Process:**

1. Select a cryptographic algorithm (e.g., P256, P384, P521, Secp256k1, RSA2048, RSA4096)
2. Generate a key pair (private key/public key) using the selected algorithm
3. Generate a UUID v4 and encode it in Base64 to create the method-specific identifier
4. Create a DID in the format `did:finger:<method-specific-identifier>`
5. Construct a DID object with the generated key pair

**DID Document Creation and Registration:**

1. Create a DID document using the generated DID:
   - Include `@context`, `id`, `verificationMethod`, and `authentication` fields
   - Optionally include `name`, `desc`, `service`, and `proof` fields
2. Serialize the DID document to JSON
3. Sign the DID document with the private key of the DID that created it:
   - The signature algorithm must match the type specified in verificationMethod
   - The signature includes a nonce and timestamp
4. Encode the signed DID document in Base64
5. Store it in the database through the registration API:
   - **Endpoint:** `POST /api/v2/did/registDIDDocument`
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
didBase64, err := did.CreateDid(keytype)  // keytype: "P256", "P384", etc.

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
   - Search the database for a DID document with that identifier in Active status
   - If found, return it in W3C DID Resolution Result format
   - If not found or revoked, return a `404 notFound` error
4. Response Format:
   ```json
   {
     "didDocument": { /* DID Document */ },
     "didResolutionMetadata": {
       "contentType": "application/did+ld+json",
       "driver": "f-did-go",
       "driverVersion": "1.0.0"
     },
     "didDocumentMetadata": {}
   }
   ```

**Authenticated Query Endpoints:**

1. Query by User ID: `GET /api/v2/did/getDIDDocumentByID?userID=<user-id>`
2. Query by DID ID: `GET /api/v2/did/getDIDDocumentByDIDID?userDIDID=<base64-encoded-did>`
3. API Key authentication required (`X-API-Key` header)
4. Only queries DID documents for a specific consumer

### Update

This operation updates an existing DID document.

**Update Process:**

1. Prepare the DID document to update:
   - Must use the same `id` field as the existing DID
   - Include updated content (verificationMethod, service, etc.)
2. Sign the DID document with the private key of the DID that created it
3. Encode the signed DID document in Base64
4. Update using the registration API:
   - **Endpoint:** `POST /api/v2/did/registDIDDocument` (same endpoint as Create)
   - **Authentication:** API Key header required
   - The system automatically finds the existing DID document, changes it to `Revoked` status, and stores the new document with `Active` status

**Important Notes:**

- When updating, all previous versions of the DID document are set to `Revoked` status
- The `id` field of the DID cannot be changed
- **Authentication Requirements:**
  - Both API Key authentication and DID signature verification are required
  - The signature in the DID Document's `proof` field must verify against a key in the current Active DID Document's `authentication` array
  - For signature verification, `authentication[0]` (first authentication method) is used as the default verification key

### Deactivate

This operation revokes a DID document so it can no longer be used.

**Revocation Process:**

1. Identify the DID ID to revoke
2. Call the revocation API:
   - **Endpoint:** `POST /api/v2/did/revokeDIDDocument`
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

## 5. DID Document Structure

The finger DID method conforms to the W3C DID Core specification and has the following structure:

```json
{
  "@context": ["https://www.w3.org/ns/did/v1"],
  "id": "did:finger:<method-specific-identifier>",
  "name": "<optional-name>",
  "desc": "<optional-description>",
  "verificationMethod": [
    {
      "id": "did:finger:<method-specific-identifier>",
      "type": "<verification-key-type>",
      "publicKeyBase58": "<base58-encoded-public-key>"
    }
  ],
  "authentication": [
    {
      "id": "did:finger:<method-specific-identifier>",
      "type": "<verification-key-type>",
      "publicKeyBase58": "<base58-encoded-public-key>"
    }
  ],
  "service": {
    "id": "did:finger:<method-specific-identifier>#service",
    "type": "SmartContractService",
    "serviceEndpoint": {
      "chain": "<blockchain-name>",
      "contractAddress": "<contract-address>",
      "rpcUrl": "<rpc-url>",
      "abi": "<contract-abi>"
    }
  },
  "proof": {
    "type": "<verification-key-type>",
    "creator": "did:finger:<method-specific-identifier>",
    "created": "<iso8601-timestamp-rfc3339nano>",
    "nonce": "<hex-encoded-32-char-nonce>",
    "signatureValue": "<base64-url-safe-encoded-signature>"
  }
}
```

## 6. Supported Cryptographic Algorithms

The finger DID method supports the following cryptographic algorithms:

- **ECDSA P-256** (`EcdsaP256VerificationKey2019`)
- **ECDSA P-384** (`EcdsaP384VerificationKey2019`)
- **ECDSA P-521** (`EcdsaP521VerificationKey2019`)
- **ECDSA secp256k1** (`EcdsaSecp256k1VerificationKey2019`)
- **RSA 2048** (`RsaVerificationKey2018`)
- **RSA 4096** (`RsaVerificationKey2018`)

## 7. DID Resolution

The finger DID method conforms to the W3C DID Resolution standard.

### Resolution Endpoint

```
GET /1.0/identifiers/{did}
```

**Example:**
```
GET /1.0/identifiers/did:finger:ZDNlNGJkZDktYWRhYy00YTgzLWJhMzctMGY2N2Y2OTQ2ZmQy
```

### Response Format

On success, responds in W3C DID Resolution Result format:

```json
{
  "didDocument": {
    "@context": ["https://www.w3.org/ns/did/v1"],
    "id": "did:finger:...",
    ...
  },
  "didResolutionMetadata": {
    "contentType": "application/did+ld+json",
    "driver": "f-did-go",
    "driverVersion": "1.0.0"
  },
  "didDocumentMetadata": {}
}
```

### Resolution Errors

Errors are returned in the following cases:

- **400 Bad Request:** Invalid DID format
- **404 notFound:** DID document not found (revoked or does not exist)
- **500 Internal Server Error:** Server internal error

## 8. DID Document

The returned DID Document conforms to DID Core:

- **@context:** Always the first element (`https://www.w3.org/ns/did/v1`)
- **id:** The resolved DID
- **verificationMethod:** Supported public keys (e.g., `EcdsaP256VerificationKey2019`)
- **authentication:** References to keys used for authentication
- **service:** Service endpoints (optional)
- **proof:** Signature of the DID document (optional)

## 9. DID Document Management

The finger DID method provides functionality for registering, querying, and revoking DID documents. All management operations are protected through API Key-based authentication, and DID documents are stored and managed in a database.

## 10. Verifiable Data Registry (VDR) and Trust Model

### VDR Architecture

The finger DID method uses a centralized Verifiable Data Registry (VDR) managed by the F-DID service operator. The VDR stores DID documents and manages their lifecycle (Active/Revoked states).

**Resolution Model:**
- DID resolution is performed through a public HTTP endpoint (`GET /1.0/identifiers/{did}`)
- The resolver queries the VDR database maintained by the F-DID service operator
- All Active DID documents are publicly resolvable without authentication

**Trust Model:**
- **Availability Dependency:** DID resolution depends on the availability and operational integrity of the F-DID service operator's infrastructure
- **Integrity Protection:** While the VDR is centralized, DID document integrity is protected through cryptographic signatures in the `proof` field
- **Trust Assumptions:** 
  - Clients must trust the F-DID service operator to:
    - Maintain database availability
    - Return accurate DID documents for Active DIDs
    - Not arbitrarily revoke or modify DID documents without proper authorization
  - DID document authenticity is independently verifiable through cryptographic signature verification, regardless of VDR operator actions

**Mitigation Strategies:**
- **Cryptographic Verification:** All DID documents include cryptographic signatures that can be verified independently of the VDR
- **Audit Logging:** State changes (Create/Update/Deactivate) are logged for audit purposes
- **Signature-Based Authorization:** Updates and deactivations require valid cryptographic signatures from DID controllers, preventing arbitrary modifications by the VDR operator

**Residual Risks:**
- **Service Availability:** DID resolution may be unavailable if the F-DID service is offline
- **Operator Compromise:** If the VDR operator is compromised, they could:
  - Deny resolution (DoS)
  - Return incorrect or stale data (detectable through signature verification)
  - However, they cannot forge valid signatures without controller private keys
- **Database Breach:** A database breach could expose DID document contents, but would not allow unauthorized updates without controller private keys

**Operational Considerations:**
- The F-DID service operator maintains high availability infrastructure
- Database backups and disaster recovery procedures are maintained
- API Key management follows security best practices
- Service status and maintenance windows are communicated to users

## 11. Blockchain Integration

The finger DID method can optionally integrate with blockchains for additional verification and registry capabilities.

## 12. Governance

- **Authority:** F-DID Project Team
- **DID Acquisition Process:** DID creation and registration available through API Key authentication
- **Maintenance:** Continuously managed and improved by the F-DID project

## 13. Privacy & Security Considerations

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

- Public keys are stored in the `verificationMethod` field of DID Documents in Base58-encoded format
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
  - The DID document (excluding the `proof` field) is serialized to JSON using Go's standard `json.Marshal` function
  - The resulting JSON bytes are used as the input for cryptographic signing
  - Note: Go's `json.Marshal` produces deterministic output for the same input structure, but implementers should be aware that field ordering may vary between implementations
- **Proof Structure:**
  - Signatures include the following information:
    - `creator`: The DID that created the signature
    - `created`: Signature creation time (ISO8601 format, RFC3339Nano)
    - `type`: The verification method type (matches verificationMethod)
    - `nonce`: Random hexadecimal string (32 hex characters, 16 bytes) to prevent replay attacks
    - `signatureValue`: Base64 URL-safe encoded signature value (base64.URLEncoding)
- **Signature Process:**
  1. Create DID document structure without `proof` field
  2. Serialize to JSON bytes using `json.Marshal`
  3. Sign the JSON bytes using the private key
  4. Create proof object with signature and metadata
  5. Add proof to the DID document
- Signature algorithms must match the type specified in verificationMethod:
  - ECDSA P-256: Uses SHA-256 hash
  - ECDSA P-384: Uses SHA-384 hash
  - ECDSA P-521: Uses SHA-512 hash
  - Secp256k1: Uses SHA-256 hash
  - RSA: Uses SHA-256 hash and PKCS1v15 padding

**Signature Verification:**

- Signature verification must be performed when receiving DID documents
- **Verification Process:**
  1. Extract the `proof` field from the DID document
  2. Create a copy of the DID document without the `proof` field
  3. Serialize the proof-free document to JSON bytes using the same method as signing (Go's `json.Marshal`)
  4. Extract the public key from the `authentication[0].PublicKeyBase58` field (first authentication method)
  5. Decode the `signatureValue` from Base64 URL-safe encoding
  6. Verify the signature using the public key and the serialized JSON bytes
- DID documents are considered invalid if signature verification fails
- The `nonce` field in proof can be used by implementers for replay attack prevention (storage and validation of nonces is implementation-specific)
- Signature verification is performed using the DID's public key from the `authentication` field

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
  - For signature verification, the first key in the `authentication` array (`authentication[0]`) is used
  - When multiple authentication keys exist, the controller is responsible for ensuring the signing key matches one of the keys in the `authentication` array
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
  - ECDSA: Uses P-256, P-384, P-521 curves
  - RSA: Uses key lengths of 2048 bits or more
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

## 14. Implementation Information

The finger DID method is implemented in the Go programming language. This section provides information for developers who wish to implement DID Resolvers or other software that interacts with the `did:finger` method.

### Implementing a DID Resolver

A DID Resolver for the `did:finger` method can be implemented in any programming language by following these steps:

**1. DID Format Validation:**

Validate that the DID follows the format `did:finger:<method-specific-identifier>` where:
- The method-specific identifier is a Base64-encoded UUID string
- Character set: A-Z, a-z, 0-9, +, /, =
- Length: approximately 44-48 characters

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
     "didDocumentMetadata": {}
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
  curl "https://your-server.com/1.0/identifiers/did:finger:ZDNlNGJkZDktYWRhYy00YTgzLWJhMzctMGY2N2Y2OTQ2ZmQy"
  ```

**Management Endpoints (API Key Authentication Required):**

These endpoints require API Key authentication and are used for DID document management:

- **Register/Update:** `POST /api/v2/did/registDIDDocument`
  - Header: `X-API-Key: <your-api-key>`
  - Body: JSON with `userId`, `userDIDID`, `didDocument`
  
- **Get by User ID:** `GET /api/v2/did/getDIDDocumentByID?userID=<user-id>`
  - Header: `X-API-Key: <your-api-key>`
  
- **Get by DID ID:** `GET /api/v2/did/getDIDDocumentByDIDID?userDIDID=<base64-encoded-did>`
  - Header: `X-API-Key: <your-api-key>`
  
- **Revoke:** `POST /api/v2/did/revokeDIDDocument`
  - Header: `X-API-Key: <your-api-key>`
  - Body: JSON with `userDIDID`

**Note:** API Keys are issued by the F-DID service provider. Contact the service provider to obtain API Keys for accessing management endpoints.

### Creating DIDs Programmatically

**DID Generation Algorithm:**

1. Generate a UUID v4 (e.g., `3d3e4bdd-9adac-04a83-ba37-0f67f69462fd2`)
2. Convert the UUID string to a byte array
3. Encode using Base64 standard encoding (StdEncoding)
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
    uuidObj, err := uuid.NewUUID()
    if err != nil {
        return "", err
    }
    uuidStr := uuidObj.String()
    
    // Convert to bytes and Base64 encode
    uuidBytes := []byte(uuidStr)
    methodSpecificId := base64.StdEncoding.EncodeToString(uuidBytes)
    
    // Create DID
    did := fmt.Sprintf("did:finger:%s", methodSpecificId)
    return did, nil
}
```

### DID Document Structure

When creating or updating DID documents, ensure they conform to the following structure:

```json
{
  "@context": ["https://www.w3.org/ns/did/v1"],
  "id": "did:finger:<method-specific-identifier>",
  "verificationMethod": [
    {
      "id": "did:finger:<method-specific-identifier>",
      "type": "EcdsaP256VerificationKey2019",
      "publicKeyBase58": "<base58-encoded-public-key>"
    }
  ],
  "authentication": [
    "did:finger:<method-specific-identifier>"
  ],
  "proof": {
    "type": "EcdsaP256VerificationKey2019",
    "creator": "did:finger:<method-specific-identifier>",
    "created": "<iso8601-timestamp>",
    "nonce": "<random-nonce>",
    "signatureValue": "<base64-encoded-signature>"
  }
}
```

### Supported Cryptographic Algorithms

When implementing signature verification, support the following algorithms:

- **ECDSA P-256** with SHA-256 (`EcdsaP256VerificationKey2019`)
- **ECDSA P-384** with SHA-384 (`EcdsaP384VerificationKey2019`)
- **ECDSA P-521** with SHA-512 (`EcdsaP521VerificationKey2019`)
- **ECDSA secp256k1** with SHA-256 (`EcdsaSecp256k1VerificationKey2019`)
- **RSA 2048** with SHA-256 (`RsaVerificationKey2018`)
- **RSA 4096** with SHA-256 (`RsaVerificationKey2018`)

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
- **Organization:** Finger, Technology Research Institute

The specification document and API endpoints described herein provide all information necessary for independent implementation of DID Resolvers and related software.

## 15. References

### Standards

- [W3C DID Core Specification](https://www.w3.org/TR/did-core/)
- [W3C DID Resolution](https://www.w3.org/TR/did-resolution/)
- [W3C DID Implementation Guide](https://www.w3.org/TR/did-imp-guide/)

## 16. Contact Information

- **Email:** youngseoka@finger.co.kr
- **Organization:** Finger, Technology Research Institute

## 17. Lifecycle

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

## 18. Test Vectors

The following are examples of valid `did:finger` DIDs:

**Example 1:**
```
did:finger:ZDNlNGJkZDktYWRhYy00YTgzLWJhMzctMGY2N2Y2OTQ2ZmQy
```

**Example 2:**
```
did:finger:YmMzMWNjMTEtYWYwYS00ZGY2LWJiNzUtOGNkMTk2MTBjMTA0
```

**Example 3:**
```
did:finger:OTliNWZjNmItYjcxOC00YzNkLWE4NGUtNTA4MDlmNzljM2Y1
```

Each DID should resolve to a valid DID Document.

## 19. W3C Registration

This method is being prepared for registration in the W3C DID Method Registry.

**Status:** Registration Pending

## 20. Status

This DID method is currently **under development** and being prepared for registration in the W3C DID Extensions registry.
