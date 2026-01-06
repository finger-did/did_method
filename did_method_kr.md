# DID Method: finger

## 개요

**finger** DID 메서드는 F-DID 프로젝트에서 구현된 분산 식별자(DID) 메서드입니다. Go 언어로 작성된 DID 서버 및 클라이언트 모듈로, DID 문서 생성, 등록, 해석, 폐기 기능을 제공합니다.

## DID Method Name

등록된 메서드 이름은 `finger`입니다.
유효한 DID는 `did:finger:`로 시작합니다.

## DID Syntax

`did:finger` DID는 다음 문법을 따릅니다:

**Human-readable:**
```
did:finger:<method-specific-identifier>
```

**정규식 (Regex):**
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

Method-specific identifier는 Base64로 인코딩된 UUID 문자열입니다.

**생성 과정:**

1. UUID v4를 생성합니다 (예: `3d3e4bdd-9adac-04a83-ba37-0f67f69462fd2`)
2. UUID 문자열을 바이트 배열로 변환합니다
3. Base64 표준 인코딩(StdEncoding)을 사용하여 인코딩합니다
4. 결과 문자열이 method-specific identifier가 됩니다

**규칙:**
- **문자셋:** Base64 표준 문자셋 (A-Z, a-z, 0-9, +, /, =)
- **길이:** 약 44-48자 (UUID를 Base64 인코딩한 결과)
- **충돌 관리:** UUID v4의 고유성에 의해 보장됨
- **대소문자 구분:** Base64 인코딩 결과이므로 대소문자를 구분함

**예시:**
```
did:finger:ZDNlNGJkZDktYWRhYy00YTgzLWJhMzctMGY2N2Y2OTQ2ZmQy
```

## Method Operations

finger DID 메서드는 다음 작업을 지원합니다:

- **Create:** 새로운 DID와 DID 문서를 생성하고 등록합니다
- **Read/Resolve:** DID를 해석하여 DID 문서를 조회합니다
- **Update:** 기존 DID 문서를 업데이트합니다
- **Deactivate:** DID 문서를 폐기(비활성화)합니다

모든 작업은 API Key 기반 인증을 통해 보호됩니다.

## DID 문서 구조

finger DID 메서드는 W3C DID Core 사양을 준수하며, 다음과 같은 구조를 가집니다:

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

## 지원 암호화 알고리즘

finger DID 메서드는 다음 암호화 알고리즘을 지원합니다:

- **ECDSA P-256** (`EcdsaP256VerificationKey2019`)
- **ECDSA P-384** (`EcdsaP384VerificationKey2019`)
- **ECDSA P-521** (`EcdsaP521VerificationKey2019`)
- **ECDSA secp256k1** (`EcdsaSecp256k1VerificationKey2019`)
- **RSA 2048** (`RsaVerificationKey2018`)
- **RSA 4096** (`RsaVerificationKey2018`)

## DID Resolution

finger DID 메서드는 W3C DID Resolution 표준을 준수합니다.

### 해석 엔드포인트

```
GET /1.0/identifiers/{did}
```

**예시:**
```
GET /1.0/identifiers/did:finger:ZDNlNGJkZDktYWRhYy00YTgzLWJhMzctMGY2N2Y2OTQ2ZmQy
```

### 응답 형식

성공 시 W3C DID Resolution Result 형식으로 응답합니다:

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

### 해석 오류

다음과 같은 경우 오류가 반환됩니다:

- **400 Bad Request:** 잘못된 DID 형식
- **404 notFound:** DID 문서를 찾을 수 없음 (폐기되었거나 존재하지 않음)
- **500 Internal Server Error:** 서버 내부 오류

## DID Document

반환되는 DID Document는 DID Core를 준수합니다:

- **@context:** 항상 첫 번째 요소 (`https://www.w3.org/ns/did/v1`)
- **id:** 해석된 DID
- **verificationMethod:** 지원되는 공개 키 (예: `EcdsaP256VerificationKey2019`)
- **authentication:** 인증에 사용되는 키 참조
- **service:** 서비스 엔드포인트 (선택적)
- **proof:** DID 문서의 서명 (선택적)

## DID 문서 관리

finger DID 메서드는 DID 문서의 등록, 조회, 폐기 기능을 제공합니다. 모든 관리 작업은 API Key 기반 인증을 통해 보호되며, DID 문서는 데이터베이스에 저장되어 관리됩니다.

## 블록체인 통합

finger DID 메서드는 선택적으로 블록체인과 통합할 수 있습니다.

## Governance

- **관리 주체:** F-DID 프로젝트 팀
- **DID 획득 프로세스:** API Key 인증을 통해 DID 생성 및 등록 가능
- **유지보수:** F-DID 프로젝트에서 지속적으로 관리 및 개선

## Privacy & Security Considerations

### 키 관리

- 개인 키는 암호화되어 저장되며, 메모리 보호 기능 제공
- 키 쌍 생성 시 다양한 암호화 알고리즘 지원 (ECDSA, RSA 등)
- 개인 키는 클라이언트 측에서 관리되거나 암호화된 형태로 저장됨

### 서명 및 검증

- 모든 DID 문서는 서명 검증을 통해 무결성 보장
- 서명 알고리즘은 verificationMethod에 명시된 타입과 일치해야 함
- 서명 검증 실패 시 DID 문서는 유효하지 않은 것으로 간주됨

### 상태 관리

- DID 문서는 Active/Revoked 상태로 관리됨
- 폐기된 DID는 해석 시 404 오류를 반환함
- 상태 변경은 API Key 인증을 통해 보호됨

### 인증 및 접근 제어

- 관리 API는 API Key 기반 인증 필요
- 해석 API는 공개적으로 접근 가능하지만, DID 문서 자체는 공개 정보
- Rate limiting 및 DoS 보호 기능 제공

### 데이터 최소화

- DID Document에는 최소한의 정보만 포함 (식별자, 공개 키, 서비스 엔드포인트)
- 개인 정보나 민감한 데이터는 DID Document에 저장되지 않음
- 개인 키는 절대 DID Document에 포함되지 않음

### 전송 보안

- 모든 API 통신은 TLS를 통해 암호화됨
- HTTPS를 통한 안전한 통신 보장

## 구현 정보

- **프로젝트 이름:** F-DID
- **언어:** Go
- **프레임워크:** Fiber (Go web framework)
- **데이터베이스:** PostgreSQL

## References

- [W3C DID Core Specification](https://www.w3.org/TR/did-core/)
- [W3C DID Resolution](https://www.w3.org/TR/did-resolution/)
- [W3C DID Implementation Guide](https://www.w3.org/TR/did-imp-guide/)
- [F-DID 프로젝트](https://github.com/finger/f-did_go)

## 연락처 정보

- **프로젝트 리포지토리:** https://github.com/finger/f-did_go (또는 GitLab)
- **문의:** 프로젝트 이슈 트래커를 통해 문의 가능

## Lifecycle

### 버전 관리

- 이 스펙은 시맨틱 버전 관리(Semantic Versioning)를 따릅니다
- 주요 변경사항은 새 버전과 사전 공지를 요구합니다

### 폐기 및 비활성화

- DID가 폐기되면 해석 시 `notFound` 오류를 반환합니다
- 폐기된 DID는 더 이상 사용할 수 없습니다
- 폐기된 DID의 상태는 데이터베이스에서 `Revoked`로 표시됩니다

### 변경 관리

- 이슈 및 Pull Request는 프로젝트 리포지토리에서 추적됩니다
- 주요 변경사항은 커뮤니티에 공지됩니다

## Test Vectors

다음은 유효한 `did:finger` DID 예시입니다:

**예시 1:**
```
did:finger:ZDNlNGJkZDktYWRhYy00YTgzLWJhMzctMGY2N2Y2OTQ2ZmQy
```

**예시 2:**
```
did:finger:YmMzMWNjMTEtYWYwYS00ZGY2LWJiNzUtOGNkMTk2MTBjMTA0
```

**예시 3:**
```
did:finger:OTliNWZjNmItYjcxOC00YzNkLWE4NGUtNTA4MDlmNzljM2Y1
```

각 DID는 유효한 DID Document로 해석되어야 합니다.

## W3C Registration

이 메서드는 W3C DID Method Registry에 등록을 준비 중입니다.

**상태:** 등록 준비 중

## 상태

이 DID 메서드는 현재 **개발 중** 상태이며, W3C DID Extensions 레지스트리에 등록을 준비 중입니다.

