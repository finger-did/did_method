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

**Example:**
```
did:finger:ZDNlNGJkZDktYWRhYy00YTgzLWJhMzctMGY2N2Y2OTQ2ZmQy
```

## 4. Method Operations

The finger DID method supports the following operations:

- **Create:** Generate and register a new DID and DID document
- **Read/Resolve:** Resolve a DID to retrieve its DID document
- **Update:** Update an existing DID document
- **Deactivate:** Revoke (deactivate) a DID document

All operations are protected through API Key-based authentication.

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
    "id": "did:finger:<method-specific-identifier>",
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
    "created": "<iso8601-timestamp>",
    "nonce": "<nonce>",
    "signatureValue": "<signature>"
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

## 10. Blockchain Integration

The finger DID method can optionally integrate with blockchains.

## 11. Governance

- **Authority:** F-DID Project Team
- **DID Acquisition Process:** DID creation and registration available through API Key authentication
- **Maintenance:** Continuously managed and improved by the F-DID project

## 12. Privacy & Security Considerations

### Key Management

- Private keys are stored encrypted with memory protection features
- Support for various cryptographic algorithms during key pair generation (ECDSA, RSA, etc.)
- Private keys are managed on the client side or stored in encrypted form

### Signing and Verification

- All DID documents guarantee integrity through signature verification
- Signature algorithms must match the type specified in verificationMethod
- DID documents are considered invalid if signature verification fails

### State Management

- DID documents are managed in Active/Revoked states
- Revoked DIDs return a 404 error upon resolution
- State changes are protected through API Key authentication

### Authentication and Access Control

- Management APIs require API Key-based authentication
- Resolution APIs are publicly accessible, but DID documents themselves are public information
- Rate limiting and DoS protection features are provided

### Data Minimization

- DID Documents contain only minimal information (identifier, public keys, service endpoints)
- Personal information or sensitive data is not stored in DID Documents
- Private keys are never included in DID Documents

### Transport Security

- All API communications are encrypted through TLS
- Secure communication guaranteed through HTTPS

## 13. Implementation Information

- **Project Name:** F-DID
- **Language:** Go
- **Framework:** Fiber (Go web framework)
- **Database:** PostgreSQL

## 14. References

- [W3C DID Core Specification](https://www.w3.org/TR/did-core/)
- [W3C DID Resolution](https://www.w3.org/TR/did-resolution/)
- [W3C DID Implementation Guide](https://www.w3.org/TR/did-imp-guide/)
- [F-DID Project](https://github.com/finger/f-did_go)

## 15. Contact Information

- **Project Repository:** https://github.com/finger/f-did_go (or GitLab)
- **Inquiries:** Available through the project issue tracker

## 16. Lifecycle

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

## 17. Test Vectors

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

## 18. W3C Registration

This method is being prepared for registration in the W3C DID Method Registry.

**Status:** Registration Pending

## 19. Status

This DID method is currently **under development** and being prepared for registration in the W3C DID Extensions registry.

