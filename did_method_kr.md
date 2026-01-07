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

**참고:** 이 스펙은 표준 Base64 인코딩(RFC 4648)을 사용하며, `+`, `/`, `=` 패딩 문자를 포함합니다. 구현자는 URL 경로에서 DID를 전송할 때 라우팅 문제를 방지하기 위해 적절한 URL 인코딩을 보장해야 합니다.

**예시:**
```
did:finger:ZDNlNGJkZDktYWRhYy00YTgzLWJhMzctMGY2N2Y2OTQ2ZmQy
```

## Method Operations

finger DID 메서드는 다음 작업을 지원합니다:

### Create (생성)

새로운 DID와 DID 문서를 생성하고 등록하는 작업입니다.

**DID 생성 과정:**

1. 암호화 알고리즘을 선택합니다 (예: P256, P384, P521, Secp256k1, RSA2048, RSA4096)
2. 선택한 알고리즘으로 키 쌍(개인 키/공개 키)을 생성합니다
3. UUID v4를 생성하고 Base64로 인코딩하여 method-specific identifier를 만듭니다
4. `did:finger:<method-specific-identifier>` 형식의 DID를 생성합니다
5. 생성된 키 쌍과 함께 DID 객체를 구성합니다

**DID 문서 생성 및 등록:**

1. 생성된 DID를 사용하여 DID 문서를 생성합니다
   - `@context`, `id`, `verificationMethod`, `authentication` 필드를 포함
   - 선택적으로 `name`, `desc`, `service`, `proof` 필드를 포함할 수 있습니다
2. DID 문서를 JSON으로 직렬화합니다
3. DID 문서를 생성한 DID의 개인 키로 서명합니다
   - 서명 알고리즘은 verificationMethod에 명시된 타입과 일치해야 합니다
   - 서명에는 nonce와 타임스탬프가 포함됩니다
4. 서명된 DID 문서를 Base64로 인코딩합니다
5. 등록 API를 통해 데이터베이스에 저장합니다:
   - **엔드포인트:** `POST /api/v2/did/registDIDDocument`
   - **인증:** API Key 헤더 (`X-API-Key`) 필요
   - **요청 본문:**
     ```json
     {
       "userId": "<user-id>",
       "userDIDID": "<base64-encoded-did>",
       "didDocument": "<base64-encoded-did-document>"
     }
     ```
6. 기존 DID 문서가 있는 경우, 자동으로 `Revoked` 상태로 변경한 후 새 문서를 `Active` 상태로 저장합니다

**클라이언트 라이브러리 사용:**

Go 클라이언트 라이브러리를 사용하여 DID를 생성할 수 있습니다:

```go
// DID 생성
didBase64, err := did.CreateDid(keytype)  // keytype: "P256", "P384", etc.

// DID 문서 생성
docBase64, err := did.CreateDidDocument(didBase64, nameBase64, descBase64)
```

### Read/Resolve (읽기/해석)

DID를 해석하여 DID 문서를 조회하는 작업입니다.

**공개 해석 엔드포인트 (W3C 표준):**

1. W3C DID Resolution 표준에 따라 DID를 해석합니다
2. **엔드포인트:** `GET /1.0/identifiers/{did}`
   - API Key 인증이 필요하지 않습니다
   - 모든 consumer의 DID를 검색합니다
3. 처리 과정:
   - DID에서 `did:finger:` 접두사를 제거하여 method-specific identifier를 추출합니다
   - 데이터베이스에서 해당 identifier를 가진 Active 상태의 DID 문서를 검색합니다
   - DID 문서가 존재하면 W3C DID Resolution Result 형식으로 반환합니다
   - DID 문서가 없거나 폐기된 경우 `404 notFound` 오류를 반환합니다
4. 응답 형식:
   ```json
   {
     "didDocument": { /* DID 문서 */ },
     "didResolutionMetadata": {
       "contentType": "application/did+ld+json",
       "driver": "f-did-go",
       "driverVersion": "1.0.0"
     },
     "didDocumentMetadata": {}
   }
   ```

**인증된 조회 엔드포인트:**

1. 사용자 ID로 조회: `GET /api/v2/did/getDIDDocumentByID?userID=<user-id>`
2. DID ID로 조회: `GET /api/v2/did/getDIDDocumentByDIDID?userDIDID=<base64-encoded-did>`
3. API Key 인증이 필요합니다 (`X-API-Key` 헤더)
4. 특정 consumer의 DID 문서만 조회됩니다

### Update (업데이트)

기존 DID 문서를 업데이트하는 작업입니다.

**업데이트 과정:**

1. 업데이트할 DID 문서를 준비합니다
   - 기존 DID와 동일한 `id` 필드를 사용해야 합니다
   - 업데이트된 내용(verificationMethod, service 등)을 포함합니다
2. DID 문서를 생성한 DID의 개인 키로 서명합니다
3. 서명된 DID 문서를 Base64로 인코딩합니다
4. 등록 API를 사용하여 업데이트합니다:
   - **엔드포인트:** `POST /api/v2/did/registDIDDocument` (Create와 동일한 엔드포인트)
   - **인증:** API Key 헤더 필요
   - 시스템이 자동으로 기존 DID 문서를 찾아 `Revoked` 상태로 변경한 후, 새 문서를 `Active` 상태로 저장합니다

**주의사항:**

- 업데이트 시 기존 DID 문서의 모든 이전 버전은 `Revoked` 상태가 됩니다
- DID의 `id` 필드는 변경할 수 없습니다
- **인증 요구사항:**
  - API Key 인증과 DID 서명 검증 모두 필요합니다
  - DID Document의 `proof` 필드에 있는 서명은 현재 Active DID Document의 `authentication` 배열에 있는 키에 대해 검증되어야 합니다
  - 서명 검증의 경우, `authentication[0]` (첫 번째 인증 방법)이 기본 검증 키로 사용됩니다

### Deactivate (비활성화/폐기)

DID 문서를 폐기하여 더 이상 사용할 수 없도록 하는 작업입니다.

**폐기 과정:**

1. 폐기할 DID ID를 확인합니다
2. 폐기 API를 호출합니다:
   - **엔드포인트:** `POST /api/v2/did/revokeDIDDocument`
   - **인증:** API Key 헤더 필요
   - **요청 본문:**
     ```json
     {
       "userDIDID": "<base64-encoded-did>"
     }
     ```
3. 처리 과정:
   - 데이터베이스에서 해당 DID를 찾습니다
   - DID 문서의 상태를 `Active`에서 `Revoked`로 변경합니다
4. 폐기 후:
   - 폐기된 DID는 해석 시 `404 notFound` 오류를 반환합니다
   - 폐기된 DID는 더 이상 업데이트할 수 없습니다
   - 폐기된 DID는 재활성화할 수 없습니다

**모든 작업은 API Key 기반 인증을 통해 보호됩니다.**

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

## Verifiable Data Registry (VDR) 및 신뢰 모델

### VDR 아키텍처

finger DID 메서드는 F-DID 서비스 운영자가 관리하는 중앙화된 Verifiable Data Registry (VDR)를 사용합니다. VDR은 DID 문서를 저장하고 라이프사이클(Active/Revoked 상태)을 관리합니다.

**해석 모델:**
- DID 해석은 공개 HTTP 엔드포인트(`GET /1.0/identifiers/{did}`)를 통해 수행됩니다
- Resolver는 F-DID 서비스 운영자가 유지 관리하는 VDR 데이터베이스를 쿼리합니다
- 모든 Active DID 문서는 인증 없이 공개적으로 해석 가능합니다

**신뢰 모델:**
- **가용성 의존성:** DID 해석은 F-DID 서비스 운영자 인프라의 가용성 및 운영 무결성에 의존합니다
- **무결성 보호:** VDR이 중앙화되어 있지만, DID 문서 무결성은 `proof` 필드의 암호화 서명을 통해 보호됩니다
- **신뢰 가정:**
  - 클라이언트는 F-DID 서비스 운영자가 다음을 수행할 것을 신뢰해야 합니다:
    - 데이터베이스 가용성 유지
    - Active DID에 대해 정확한 DID 문서 반환
    - 적절한 권한 없이 DID 문서를 임의로 폐기하거나 수정하지 않음
  - DID 문서 인증은 VDR 운영자의 조치와 무관하게 암호화 서명 검증을 통해 독립적으로 검증 가능합니다

**완화 전략:**
- **암호화 검증:** 모든 DID 문서는 VDR과 독립적으로 검증 가능한 암호화 서명을 포함합니다
- **감사 로깅:** 상태 변경(Create/Update/Deactivate)은 감사 목적으로 기록됩니다
- **서명 기반 권한 부여:** 업데이트 및 비활성화는 DID 컨트롤러의 유효한 암호화 서명을 요구하여, VDR 운영자의 임의 수정을 방지합니다

**잔여 위험:**
- **서비스 가용성:** F-DID 서비스가 오프라인일 때 DID 해석을 사용할 수 없을 수 있습니다
- **운영자 침해:** VDR 운영자가 침해당할 경우:
  - 해석 거부 (DoS)
  - 잘못되었거나 오래된 데이터 반환 (서명 검증을 통해 감지 가능)
  - 그러나 컨트롤러 개인 키 없이는 유효한 서명을 위조할 수 없습니다
- **데이터베이스 침해:** 데이터베이스 침해는 DID 문서 내용을 노출할 수 있지만, 컨트롤러 개인 키 없이는 무단 업데이트를 허용하지 않습니다

**운영 고려사항:**
- F-DID 서비스 운영자는 고가용성 인프라를 유지합니다
- 데이터베이스 백업 및 재해 복구 절차가 유지됩니다
- API Key 관리는 보안 모범 사례를 따릅니다
- 서비스 상태 및 유지보수 기간이 사용자에게 통지됩니다

## 블록체인 통합

finger DID 메서드는 선택적으로 블록체인과 통합하여 추가 검증 및 레지스트리 기능을 제공할 수 있습니다.

## Governance

- **관리 주체:** F-DID 프로젝트 팀
- **DID 획득 프로세스:** API Key 인증을 통해 DID 생성 및 등록 가능
- **유지보수:** F-DID 프로젝트에서 지속적으로 관리 및 개선

## Privacy & Security Considerations

### 공개 키 및 개인 키 처리

**개인 키 관리:**

- 개인 키는 DID 문서 생성 및 서명에 사용되며, 절대 DID Document에 포함되지 않습니다
- 개인 키는 클라이언트 측에서 생성되고 관리됩니다
- 개인 키 저장 시 암호화를 사용할 수 있습니다 (예: AES/CBC/PKCS7 암호화)
- 키 파일 저장 시 비밀번호 기반 암호화를 적용할 수 있습니다:
  - SHA-256 해시를 사용하여 비밀번호에서 암호화 키를 생성합니다
  - AES-256-CBC를 사용하여 개인 키를 암호화합니다
- 메모리에서 사용 후 즉시 지워지는 보호 기능을 제공합니다

**공개 키 노출:**

- 공개 키는 DID Document의 `verificationMethod`에 Base58 인코딩된 형태로 저장됩니다
- 공개 키는 공개 정보이므로 누구나 조회할 수 있습니다
- 공개 키는 서명 검증에 사용되며, 개인 키 없이는 서명을 생성할 수 없습니다

**키 로테이션:**

- DID 문서 업데이트를 통해 새로운 verificationMethod를 추가할 수 있습니다
- 기존 키는 유지하면서 새 키를 추가할 수 있습니다
- 키가 유출된 경우, 새 키로 DID 문서를 업데이트한 후 기존 키를 제거할 수 있습니다

### 서명 및 검증 보안

**서명 생성:**

- 모든 DID 문서는 생성 시 해당 DID의 개인 키로 서명됩니다
- **정규화 및 직렬화:**
  - DID 문서(`proof` 필드 제외)는 Go의 표준 `json.Marshal` 함수를 사용하여 JSON으로 직렬화됩니다
  - 결과 JSON 바이트가 암호화 서명의 입력으로 사용됩니다
  - 참고: Go의 `json.Marshal`은 동일한 입력 구조에 대해 결정론적 출력을 생성하지만, 구현자는 필드 순서가 구현 간에 다를 수 있음을 인지해야 합니다
- **Proof 구조:**
  - 서명에는 다음 정보가 포함됩니다:
    - `creator`: 서명을 생성한 DID
    - `created`: 서명 생성 시간 (ISO8601 형식, RFC3339Nano)
    - `type`: 검증 방법 타입 (verificationMethod와 일치)
    - `nonce`: 재전송 공격 방지를 위한 난수 16진수 문자열 (32개의 16진수 문자, 16바이트)
    - `signatureValue`: Base64 URL-safe 인코딩된 서명 값 (base64.URLEncoding)
- **서명 프로세스:**
  1. `proof` 필드 없이 DID 문서 구조 생성
  2. `json.Marshal`을 사용하여 JSON 바이트로 직렬화
  3. 개인 키를 사용하여 JSON 바이트에 서명
  4. 서명 및 메타데이터를 포함한 proof 객체 생성
  5. DID 문서에 proof 추가
- 서명 알고리즘은 verificationMethod에 명시된 타입과 일치해야 합니다:
  - ECDSA P-256: SHA-256 해시 사용
  - ECDSA P-384: SHA-384 해시 사용
  - ECDSA P-521: SHA-512 해시 사용
  - Secp256k1: SHA-256 해시 사용
  - RSA: SHA-256 해시 및 PKCS1v15 패딩 사용

**서명 검증:**

- DID 문서 수신 시 반드시 서명 검증을 수행해야 합니다
- **검증 프로세스:**
  1. DID 문서에서 `proof` 필드 추출
  2. `proof` 필드 없이 DID 문서 복사본 생성
  3. 서명과 동일한 방법(Go의 `json.Marshal`)을 사용하여 proof 없는 문서를 JSON 바이트로 직렬화
  4. `authentication[0].PublicKeyBase58` 필드에서 공개 키 추출 (첫 번째 인증 방법)
  5. `signatureValue`를 Base64 URL-safe 인코딩에서 디코딩
  6. 공개 키와 직렬화된 JSON 바이트를 사용하여 서명 검증
- 서명 검증 실패 시 DID 문서는 유효하지 않은 것으로 간주됩니다
- proof의 `nonce` 필드는 구현자가 재전송 공격 방지를 위해 사용할 수 있습니다 (nonce 저장 및 검증은 구현별로 다름)
- 서명 검증은 `authentication` 필드의 DID 공개 키를 사용하여 수행됩니다

**위조 방지:**

- 개인 키를 가진 DID 소유자만 DID 문서를 생성하거나 업데이트할 수 있습니다
- 서명 검증을 통해 DID 문서의 무결성을 보장합니다
- 데이터베이스에 저장된 DID 문서는 서명 검증 후에만 저장됩니다

### 상태 관리 보안

**Active/Revoked 상태 관리:**

- DID 문서는 `Active` 또는 `Revoked` 상태로 관리됩니다
- 상태 변경은 API Key 인증을 통해 보호됩니다
- 폐기된 DID는 해석 시 `404 notFound` 오류를 반환하여 더 이상 사용할 수 없음을 명확히 합니다
- 폐기된 DID는 재활성화할 수 없습니다

**기존 DID 문서 처리:**

- DID 문서 업데이트 시, 기존 문서는 자동으로 `Revoked` 상태로 변경됩니다
- 이를 통해 DID 문서의 버전 관리 및 이력 추적이 가능합니다
- 폐기된 버전의 DID 문서는 데이터베이스에 유지되지만 해석 시 반환되지 않습니다

### 인증 및 접근 제어

**API Key 인증:**

- 모든 DID 관리 작업(Create, Update, Deactivate)은 API Key 기반 인증을 필요로 합니다
- API Key는 HTTP 헤더 `X-API-Key`를 통해 전송됩니다
- API Key는 consumer(조직/사용자)별로 발급되며, 해당 consumer의 DID만 관리할 수 있습니다
- **인증 모델:**
  - **Create/Update/Deactivate 작업:** 다음 두 가지를 모두 요구합니다:
    1. 유효한 API Key 인증 (`X-API-Key` 헤더를 통해) - consumer 신원 및 접근 권한 확인
    2. DID Document의 `proof` 필드에 유효한 암호화 서명 - DID 컨트롤러 권한 확인
  - API Key는 **어떤 consumer**가 작업을 제출할 수 있는지 제어하고, DID 서명은 **누가 DID를 제어하는지** 증명합니다
  - 작업이 성공하려면 둘 다 유효해야 합니다
- **검증 키 선택:**
  - 서명 검증의 경우, `authentication` 배열의 첫 번째 키(`authentication[0]`)가 사용됩니다
  - 여러 인증 키가 존재할 때, 컨트롤러는 서명 키가 `authentication` 배열의 키 중 하나와 일치하는지 확인할 책임이 있습니다
- API Key 유출 시 즉시 취소하고 새로운 키를 발급해야 합니다

**공개 해석 API:**

- DID 해석 API (`GET /1.0/identifiers/{did}`)는 공개적으로 접근 가능합니다
- 이는 W3C DID Resolution 표준을 따르기 위함입니다
- DID 문서 자체는 공개 정보이므로, 인증 없이 조회할 수 있습니다

**Rate Limiting 및 DoS 보호:**

- API 요청에 대한 Rate limiting이 적용됩니다
- 과도한 요청으로부터 서비스를 보호하기 위한 DoS 보호 기능이 제공됩니다
- 요청은 감사 로그에 기록되어 추적할 수 있습니다

### 데이터 최소화 및 프라이버시

**DID Document 내용:**

- DID Document에는 최소한의 정보만 포함됩니다:
  - DID 식별자 (`id`)
  - 공개 키 (`verificationMethod`)
  - 인증 정보 (`authentication`)
  - 서비스 엔드포인트 (`service`, 선택적)
- 개인 정보(PII)나 민감한 데이터는 DID Document에 저장되지 않습니다
- 개인 키는 절대 DID Document에 포함되지 않습니다

**프라이버시 고려사항:**

- DID 식별자는 UUID 기반이므로, 식별자 자체에서 개인 정보를 추론할 수 없습니다
- DID 해석은 공개적으로 가능하므로, DID의 존재 여부는 확인할 수 있습니다
- 민감한 정보는 Verifiable Credential에 저장하고, DID Document에는 포함하지 않아야 합니다
- 서비스 엔드포인트에 민감한 정보를 포함하지 않도록 주의해야 합니다

**GDPR 및 삭제 권리:**

- finger DID 메서드는 최소한의 데이터만 저장합니다: DID 식별자와 공개 키만 저장
- DID 문서에 개인 식별 정보(PII)가 저장되지 않습니다
- DID가 폐기되면 해석할 수 없게 되어(`404 notFound` 반환) 활성 사용에서 제거됩니다
- 폐기된 DID 문서는 감사 및 무결성 목적으로 데이터베이스에 보관되지만, 공개 해석을 통해 접근할 수 없습니다
- 데이터를 완전히 삭제하려는 컨트롤러는 서비스 제공자에게 데이터 삭제 요청을 문의해야 합니다
- 이 메서드는 필수 암호화 및 식별자 정보만 저장하여 데이터 최소화 원칙을 지원합니다

### 전송 보안

**TLS/HTTPS 암호화:**

- 모든 API 통신은 TLS를 통해 암호화됩니다
- HTTPS를 통한 안전한 통신이 보장됩니다
- TLS 1.2 이상의 버전을 사용해야 합니다

**암호화 강도:**

- 지원되는 암호화 알고리즘은 업계 표준을 따릅니다:
  - ECDSA: P-256, P-384, P-521 곡선 사용
  - RSA: 2048비트 이상의 키 길이 사용
- 약한 암호화 알고리즘은 사용하지 않습니다

### 추가 보안 권장사항

**구현자 권장사항:**

- 개인 키는 안전하게 보관해야 합니다 (하드웨어 보안 모듈(HSM) 사용 권장)
- API Key는 환경 변수나 안전한 키 저장소에 보관해야 합니다
- 정기적으로 키 로테이션을 수행하는 것을 권장합니다
- DID 문서 업데이트 시 변경 사항을 신중하게 검토해야 합니다
- 서명 검증 실패 시 로그를 남기고 적절히 처리해야 합니다

**프라이버시 권장사항:**

- DID Document에는 불필요한 정보를 포함하지 마세요
- 서비스 엔드포인트 URL에 개인 정보를 포함하지 마세요
- Verifiable Credential을 사용하여 민감한 정보를 관리하세요
- 사용자에게 DID의 공개 특성에 대해 명확히 알려주세요

## 구현 정보

finger DID 메서드는 Go 언어로 구현되었습니다. 이 섹션은 `did:finger` 메서드와 상호작용하는 DID Resolver나 기타 소프트웨어를 구현하려는 개발자를 위한 정보를 제공합니다.

### DID Resolver 구현

`did:finger` 메서드에 대한 DID Resolver는 다음 단계를 따라 어떤 프로그래밍 언어로도 구현할 수 있습니다:

**1. DID 형식 검증:**

DID가 `did:finger:<method-specific-identifier>` 형식을 따르는지 검증합니다:
- method-specific identifier는 Base64로 인코딩된 UUID 문자열입니다
- 문자셋: A-Z, a-z, 0-9, +, /, =
- 길이: 약 44-48자

**2. 해석 프로세스:**

해석 프로세스를 구현합니다:

1. `did:finger:` 접두사를 제거하여 method-specific identifier를 추출합니다
2. 해석 엔드포인트를 호출합니다: `GET /1.0/identifiers/{did}`
3. 엔드포인트는 W3C DID Resolution Result를 반환합니다:
   ```json
   {
     "didDocument": { /* DID 문서 */ },
     "didResolutionMetadata": {
       "contentType": "application/did+ld+json",
       "driver": "f-did-go",
       "driverVersion": "1.0.0"
     },
     "didDocumentMetadata": {}
   }
   ```
4. 오류 응답을 처리합니다:
   - `400 Bad Request`: 잘못된 DID 형식
   - `404 notFound`: DID 문서를 찾을 수 없음 (폐기되었거나 존재하지 않음)
   - `500 Internal Server Error`: 서버 내부 오류

**3. 구현 가이드:**

- DID 형식 검증 로직 구현 (`did:finger:` 접두사 확인)
- method-specific identifier 추출 로직 구현 (`did:finger:` 제거)
- HTTP 클라이언트를 사용하여 해석 엔드포인트 호출
- 응답을 W3C DID Resolution Result 형식으로 파싱
- 오류 응답 처리 (404, 500 등)

### 통합을 위한 API 엔드포인트

**공개 해석 엔드포인트 (인증 불필요):**

- **엔드포인트:** `GET /1.0/identifiers/{did}`
- **목적:** 모든 `did:finger` DID를 해석하여 DID 문서를 조회
- **인증:** 불필요 (공개 엔드포인트)
- **예시:**
  ```bash
  curl "https://your-server.com/1.0/identifiers/did:finger:ZDNlNGJkZDktYWRhYy00YTgzLWJhMzctMGY2N2Y2OTQ2ZmQy"
  ```

**관리 엔드포인트 (API Key 인증 필요):**

이 엔드포인트들은 API Key 인증이 필요하며 DID 문서 관리에 사용됩니다:

- **등록/업데이트:** `POST /api/v2/did/registDIDDocument`
  - 헤더: `X-API-Key: <your-api-key>`
  - 본문: `userId`, `userDIDID`, `didDocument`를 포함한 JSON
  
- **사용자 ID로 조회:** `GET /api/v2/did/getDIDDocumentByID?userID=<user-id>`
  - 헤더: `X-API-Key: <your-api-key>`
  
- **DID ID로 조회:** `GET /api/v2/did/getDIDDocumentByDIDID?userDIDID=<base64-encoded-did>`
  - 헤더: `X-API-Key: <your-api-key>`
  
- **폐기:** `POST /api/v2/did/revokeDIDDocument`
  - 헤더: `X-API-Key: <your-api-key>`
  - 본문: `userDIDID`를 포함한 JSON

**참고:** API Key는 F-DID 서비스 제공자로부터 발급됩니다. 관리 엔드포인트 접근을 위한 API Key는 서비스 제공자에게 문의하세요.

### 프로그래밍 방식으로 DID 생성

**DID 생성 알고리즘:**

1. UUID v4를 생성합니다 (예: `3d3e4bdd-9adac-04a83-ba37-0f67f69462fd2`)
2. UUID 문자열을 바이트 배열로 변환합니다
3. Base64 표준 인코딩(StdEncoding)을 사용하여 인코딩합니다
4. `did:finger:`를 앞에 붙여 전체 DID를 생성합니다

**구현 예시 (Go):**

```go
package main

import (
    "encoding/base64"
    "fmt"
    "github.com/google/uuid"
)

func GenerateFingerDID() (string, error) {
    // UUID v4 생성
    uuidObj, err := uuid.NewUUID()
    if err != nil {
        return "", err
    }
    uuidStr := uuidObj.String()
    
    // 바이트로 변환 후 Base64 인코딩
    uuidBytes := []byte(uuidStr)
    methodSpecificId := base64.StdEncoding.EncodeToString(uuidBytes)
    
    // DID 생성
    did := fmt.Sprintf("did:finger:%s", methodSpecificId)
    return did, nil
}
```

### DID 문서 구조

DID 문서를 생성하거나 업데이트할 때 다음 구조를 준수해야 합니다:

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

### 지원 암호화 알고리즘

서명 검증을 구현할 때 다음 알고리즘을 지원해야 합니다:

- **ECDSA P-256** with SHA-256 (`EcdsaP256VerificationKey2019`)
- **ECDSA P-384** with SHA-384 (`EcdsaP384VerificationKey2019`)
- **ECDSA P-521** with SHA-512 (`EcdsaP521VerificationKey2019`)
- **ECDSA secp256k1** with SHA-256 (`EcdsaSecp256k1VerificationKey2019`)
- **RSA 2048** with SHA-256 (`RsaVerificationKey2018`)
- **RSA 4096** with SHA-256 (`RsaVerificationKey2018`)

### Universal Resolver 통합

Universal Resolver와 통합하려면:

1. Universal Resolver 드라이버 사양을 따르는 드라이버를 구현합니다
2. 드라이버는 `did:finger` DID에 대한 해석 요청을 처리해야 합니다
3. 결과를 W3C DID Resolution Result 형식으로 반환합니다
4. DID Resolution 사양에 따라 오류를 처리합니다

**드라이버 메타데이터:**

- 드라이버 이름: `f-did-go`
- 버전: `1.0.0`
- 해석 엔드포인트 형식: `GET /1.0/identifiers/{did}`

### 구현 라이브러리

**공식 구현:**

finger DID 메서드는 Go로 구현되었으며 다음을 제공합니다:

- REST API 엔드포인트를 가진 서버 구현
- DID 생성 및 관리를 위한 클라이언트 라이브러리
- W3C 표준과 호환되는 DID Resolver 구현

**서드파티 개발자를 위해:**

서드파티 개발자는 다음 방법으로 `did:finger` 메서드 지원을 구현할 수 있습니다:

1. 이 스펙에 따라 DID 형식 검증 구현
2. 공개 해석 엔드포인트를 호출하여 해석 구현
3. 위에 나열된 암호화 알고리즘 지원
4. W3C DID Core 및 DID Resolution 사양 준수

**참고:** 참조 구현이 제공되지만, 이 스펙을 준수하는 한 다른 언어로의 서드파티 구현을 권장하고 지원합니다.

### 구현 지원 연락처

`did:finger` 메서드 지원 구현에 대한 문의사항은 다음으로 연락하세요:

- **이메일:** youngseoka@finger.co.kr
- **조직:** Finger, Technology Research Institute

본 스펙 문서와 여기서 설명하는 API 엔드포인트는 DID Resolver 및 관련 소프트웨어의 독립적인 구현에 필요한 모든 정보를 제공합니다.

## References

### 표준 문서

- [W3C DID Core Specification](https://www.w3.org/TR/did-core/)
- [W3C DID Resolution](https://www.w3.org/TR/did-resolution/)
- [W3C DID Implementation Guide](https://www.w3.org/TR/did-imp-guide/)


## 연락처 정보

- **이메일:** youngseoka@finger.co.kr
- **조직:** Finger, Technology Research Institute

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

