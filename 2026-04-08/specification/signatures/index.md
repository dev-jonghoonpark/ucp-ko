# 메시지 서명

이 명세는 진위성과 무결성을 보장할 수 있도록 UCP 메시지에 암호학적으로 서명하는 방법을 정의합니다.

## 개요

이 명세는 [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421) HTTP Message Signatures로 UCP 메시지에 서명하고 이를 검증하는 방법을 정의합니다. UCP의 신원 모델, 지원하는 인증 메커니즘, 키 디스커버리 프로토콜은 [신원과 인증](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/overview/#identity-authentication)을 참고하세요.

HTTP Message Signatures는 다음 공격을 막아 줍니다.

- **사칭** — 공격자가 정당한 참여자인 척하며 메시지를 보내는 행위
- **변조** — 전송 중 메시지 내용을 수정하는 행위
- **재전송 공격** — 탈취한 메시지를 다른 엔드포인트나 다른 시점에 다시 보내는 행위
- **메서드/엔드포인트 혼동** — 서명된 페이로드를 다른 HTTP 메서드나 다른 경로로 재전송하는 행위

### 아키텍처

UCP는 모든 HTTP 기반 전송 방식에서 HTTP Message Signatures([RFC 9421](https://www.rfc-editor.org/rfc/rfc9421))를 사용합니다.

```text
+-----------------------------------------------------------------+
|                     SHARED FOUNDATION                           |
+-----------------------------------------------------------------+
|  Signature Format: RFC 9421 (HTTP Message Signatures)           |
|  Body Digest: RFC 9530 (Content-Digest, raw bytes)              |
|  Algorithms: ES256 (required), ES384 (optional)                 |
|  Key Format: JWK (RFC 7517)                                     |
|  Key Discovery: signing_keys[] in /.well-known/ucp              |
|  Replay Protection: idempotency-key (business layer)            |
+-----------------------------------------------------------------+
                              |
                              v
+-----------------------------------------------------------------+
|                     HTTP TRANSPORTS                             |
+-----------------------------------------------------------------+
|  REST API: Standard HTTP requests                               |
|  MCP: Streamable HTTP transport (JSON-RPC over HTTP)            |
+-----------------------------------------------------------------+
|  Headers:                                                       |
|    Signature-Input    (describes signed components)             |
|    Signature          (contains signature value)                |
|    Content-Digest     (body hash, raw bytes)                    |
+-----------------------------------------------------------------+
```

**참고:** UCP는 MCP 전송 방식으로 SSE 기반 전송 대신 streamable HTTP를 지정합니다. 덕분에 같은 RFC 9421 서명 메커니즘을 모든 UCP 전송 방식에 일관되게 적용할 수 있습니다.

## 공통 기반

다음 암호 기본 요소는 모든 UCP HTTP 전송 방식에서 공통으로 사용합니다.

### 서명 알고리즘

UCP는 다음 곡선을 사용하는 ECDSA 서명을 지원합니다.

| 곡선  | JWK `alg` | 해시    |
| ----- | --------- | ------- |
| P-256 | `ES256`   | SHA-256 |
| P-384 | `ES384`   | SHA-384 |

**구현 요건:**

- 모든 구현은 반드시 P-256(`ES256`) 서명 검증을 지원해야 합니다(**MUST**).
- P-384(`ES384`) 지원은 선택 사항입니다(**OPTIONAL**).

**사용 지침:**

- 서명자는 호환성을 극대화하기 위해 P-256을 사용해야 합니다(**SHOULD**).
- 양쪽이 모두 지원한다면 서명자는 P-384를 사용할 수 있습니다(**MAY**).
- 알고리즘은 JWK의 `crv` 필드로 도출하며, `alg`는 `Signature-Input` 파라미터에 포함하지 **않습니다(NOT)**.

### 키 형식 (JWK)

공개 키는 반드시 [RFC 7517](https://datatracker.ietf.org/doc/html/rfc7517)에 정의된 **JSON Web Key(JWK)** 형식으로 표현해야 합니다(**MUST**).

**EC 키 구조:**

| 필드  | 타입   | 필수   | 설명                        |
| ----- | ------ | ------ | --------------------------- |
| `kid` | string | 예     | 키 ID(서명에서 참조)        |
| `kty` | string | 예     | 키 타입(타원 곡선은 `EC`)   |
| `crv` | string | 예\*   | 곡선 이름(`P-256`, `P-384`) |
| `x`   | string | 예\*   | X 좌표(base64url 인코딩)    |
| `y`   | string | 예\*   | Y 좌표(base64url 인코딩)    |
| `use` | string | 아니요 | 키 용도(서명용은 `sig`)     |
| `alg` | string | 아니요 | 알고리즘(`ES256`, `ES384`)  |

\* EC 키에서 필수

**예시:**

```json
{
  "kid": "key-2024-01-15",
  "kty": "EC",
  "crv": "P-256",
  "x": "WKn-ZIGevcwGIyyrzFoZNBdaq9_TsqzGl96oc0CWuis",
  "y": "y77t-RvAHRKTsSGdIYUfweuOvwrvDD-Q3Hv5J0fSKbE",
  "use": "sig",
  "alg": "ES256"
}
```

### 키 디스커버리

공개 키는 각 당사자의 `/.well-known/ucp` UCP 프로필에 있는 `signing_keys` 배열에 공개됩니다. 전체 키 디스커버리 프로토콜, 프로필 신뢰 모델, 프로필 조회 요건은 [신원과 인증 — 키 디스커버리](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/overview/#key-discovery)를 참고하세요.

### 키 교체

서비스 중단 없이 키를 교체하는 절차는 다음과 같습니다.

1. **새 키 추가** — `signing_keys[]`에 기존 키와 함께 새 키를 공개합니다
1. **서명 시작** — 새 키로 서명을 시작합니다
1. **유예 기간** — 기존 키로 만든 서명도 계속 수락합니다(최소 7일)
1. **기존 키 제거** — `signing_keys[]`에서 기존 키를 제거합니다

**권장 사항:**

- 90일마다 키를 교체합니다
- 전환 기간 동안 여러 활성 키를 지원합니다
- 검증자: `signing_keys[]`에 있는 키라면 무엇이든 수락합니다

**키 유출 대응:**

1. 유출된 키를 프로필에서 즉시 제거합니다
1. 다른 `kid`로 새 키를 추가합니다
1. 유출된 키로 만든 서명은 모두 거부합니다

## REST 바인딩

HTTP REST 전송 방식에서 UCP는 [RFC 9421 (HTTP Message Signatures)](https://www.rfc-editor.org/rfc/rfc9421)를 사용합니다.

### 헤더

| 헤더              | 방향      | 필수     | 설명                          |
| ----------------- | --------- | -------- | ----------------------------- |
| `Signature-Input` | 요청/응답 | 예       | 서명된 컴포넌트 기술          |
| `Signature`       | 요청/응답 | 예       | 서명 값                       |
| `Content-Digest`  | 요청/응답 | 조건부\* | 요청/응답 본문의 SHA-256 해시 |

\* 요청/응답에 본문이 있으면 필수

`Content-Digest`는 [RFC 9530](https://www.rfc-editor.org/rfc/rfc9530)을 따르며 원본 본문 바이트를 해시합니다. 덕분에 JSON 정규화 없이도 메시지 본문을 서명에 결합할 수 있습니다. 구현은 반드시 `sha-256`을 사용해야 합니다(**MUST**). 정규화가 필요한 영속적 아티팩트는 [AP2 위임 - 정규화](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/ap2-mandates/#canonicalization)를 참고하세요.

**중개자 경고:** 프록시, API 게이트웨이 같은 중개자는 JSON 본문을 다시 직렬화해서는 안 됩니다(**MUST NOT**). 다시 직렬화하면 서명이 무효가 됩니다. `Content-Digest`는 원본 바이트로 계산하므로 조금이라도 수정하면 검증이 깨집니다.

### REST 요청 서명

**서명 컴포넌트:**

| 컴포넌트          | 필수         | 설명                                   |
| ----------------- | ------------ | -------------------------------------- |
| `@method`         | 예           | HTTP 메서드(GET, POST 등)              |
| `@authority`      | 예           | 대상 호스트(다른 호스트로의 중계 방지) |
| `@path`           | 예           | 요청 경로                              |
| `@query`          | 조건부\*     | 쿼리 문자열(있는 경우)                 |
| `ucp-agent`       | 조건부\*\*   | 프로필 URL(신원 결합)                  |
| `idempotency-key` | 조건부\*\*\* | 멱등성 헤더(상태 변경 요청)            |
| `content-digest`  | 조건부†      | 본문 다이제스트(본문이 있는 경우)      |
| `content-type`    | 조건부†      | Content-Type(본문이 있는 경우)         |

\* 요청에 쿼리 파라미터가 있으면 필수

\*\* `UCP-Agent` 헤더가 있으면 필수

\*\*\* POST, PUT, DELETE, PATCH에서 필수

† 요청에 본문이 있으면 필수

**서명 생성:**

```text
sign_rest_request(method, path, query, body_bytes, idempotency_key, private_key, kid):
    // 1. Compute body digest (if body present)
    if body_bytes:
        digest = sha256(body_bytes)  // Hash raw bytes, no canonicalization
        digest_header = "sha-256=:" + base64(digest) + ":"

    // 2. Build component list
    components = ["@method", "@authority", "@path"]
    if query: components.append("@query")
    if ucp_agent: components.append("ucp-agent")
    if idempotency_key: components.append("idempotency-key")
    if body: components.extend(["content-digest", "content-type"])

    // 3. Build signature base (RFC 9421)
    signature_base = build_signature_base(
        components=components,
        method=method,
        path=path,
        query=query,
        headers={
            "idempotency-key": idempotency_key,
            "content-digest": digest_header,
            "content-type": "application/json"
        },
        keyid=kid
    )

    // 4. Sign
    signature = ecdsa_sign(signature_base, private_key)

    // 5. Return headers
    return {
        "Idempotency-Key": idempotency_key,
        "Content-Digest": digest_header,
        "Signature-Input": format_signature_input(components, kid),
        "Signature": "sig1=:" + base64(signature) + ":"
    }
```

**서명 인코딩:** ECDSA 서명은 반드시 ASN.1/DER이 **아니라** RFC 9421에 따른 고정 길이 원시 `r||s` 인코딩을 사용해야 합니다(**MUST**). 서명 값은 고정 길이 부호 없는 빅엔디언 정수 `r`과 `s`를 이어 붙인 값으로, P-256은 64바이트(32 + 32), P-384는 96바이트(48 + 48)입니다. 많은 암호 라이브러리(OpenSSL, Java, .NET)가 기본으로 DER 인코딩을 사용하므로 명시적으로 변환해야 합니다.

**전체 요청 예시:**

```http
POST /checkout-sessions HTTP/1.1
Host: merchant.example.com
Content-Type: application/json
UCP-Agent: profile="https://platform.example/.well-known/ucp"
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Digest: sha-256=:X48E9qOokqqrvdts8nOJRJN3OWDUoyWxBf7kbu9DBPE=:
Signature-Input: sig1=("@method" "@authority" "@path" "ucp-agent" "idempotency-key" "content-digest" "content-type");keyid="platform-2026"
Signature: sig1=:MEUCIQDTxNq8h7LGHpvVZQp1iHkFp9+3N8Mxk2zH1wK4YuVN8w...:

{"checkout":{"line_items":[{"id":"prod_123","quantity":2}]}}
```

**GET 요청 예시 (본문 없음, 멱등성 키 없음):**

```http
GET /checkout-sessions/chk_123 HTTP/1.1
Host: merchant.example.com
Signature-Input: sig1=("@method" "@authority" "@path");keyid="platform-2026"
Signature: sig1=:MEQCIBx7kL9nM2oP5qR8sT1uV4wX6yZaB3cD...:
```

### REST 응답 서명

응답 서명에는 `@method` 대신 `@status`를 사용합니다.

**서명 컴포넌트:**

| 컴포넌트         | 필수     | 설명                              |
| ---------------- | -------- | --------------------------------- |
| `@status`        | 예       | HTTP 상태 코드(200, 201 등)       |
| `content-digest` | 조건부\* | 본문 다이제스트(본문이 있는 경우) |
| `content-type`   | 조건부\* | Content-Type(본문이 있는 경우)    |

\* 응답에 본문이 있으면 필수

**전체 응답 예시:**

```http
HTTP/1.1 201 Created
Content-Type: application/json
Content-Digest: sha-256=:Y5fK8nLmPqRsT3vWxYzAbCdEfGhIjKlMnO...:
Signature-Input: sig1=("@status" "content-digest" "content-type");created=1738617601;keyid="merchant-2026"
Signature: sig1=:MFQCIH7kL9nM2oP5qR8sT1uV4wX6yZaB3cD...:

{"checkout":{"id":"chk_123","status":"ready_for_complete"}}
```

**응답 서명 생성:**

응답 서명은 `@method` 대신 `@status`를 쓴다는 점만 빼면 요청 서명과 같습니다.

```text
sign_rest_response(status, body_bytes, private_key, kid):
    // 1. Compute body digest (if body present)
    if body_bytes:
        digest = sha256(body_bytes)  // Hash raw bytes, no canonicalization
        digest_header = "sha-256=:" + base64(digest) + ":"

    // 2. Build signature base (RFC 9421)
    signature_base = build_signature_base(
        components=["@status", "content-digest", "content-type"],
        status=status,
        headers={"content-digest": digest_header, "content-type": "application/json"},
        created=current_timestamp(),
        keyid=kid
    )

    // 3. Sign
    signature = ecdsa_sign(signature_base, private_key)

    // 4. Return headers
    return {
        "Content-Digest": digest_header,
        "Signature-Input": 'sig1=("@status" "content-digest" "content-type");created=...;keyid="..."',
        "Signature": "sig1=:" + base64(signature) + ":"
    }
```

### REST 요청 검증

**서명자의 프로필 URL 확인:**

서명자의 프로필 URL은 `UCP-Agent` 헤더에서 얻습니다. 이 헤더는 [RFC 8941 Dictionary](https://www.rfc-editor.org/rfc/rfc8941#section-3.2) 문법을 사용합니다.

```text
UCP-Agent: profile="https://platform.example/.well-known/ucp"
```

**파싱 규칙:**

1. RFC 8941 Dictionary로 파싱합니다
1. `profile` 키를 추출합니다(필수(REQUIRED))
1. 값은 반드시 HTTPS URL을 담은 따옴표 문자열이어야 합니다(MUST)
1. 비즈니스 프로필이라면 URL은 반드시 `/.well-known/ucp`를 가리켜야 합니다(MUST). 플랫폼 프로필 URL에는 경로 제약이 없습니다
1. HTTPS가 아닌 URL은 거부합니다

**예시:**

```text
// Header
UCP-Agent: profile="https://platform.example/.well-known/ucp"

// Parsed
profile_url = "https://platform.example/.well-known/ucp"
```

**적용 범위:**

- **플랫폼 → 비즈니스 요청:** `UCP-Agent` 헤더의 프로필 URL
- **비즈니스 → 플랫폼 웹훅:** `UCP-Agent` 헤더의 프로필 URL

```text
verify_rest_request(request):
    // 1. Parse Signature-Input
    sig_input = parse_signature_input(request.headers["Signature-Input"])
    keyid = sig_input.keyid
    components = sig_input.components

    // 2. Fetch signer's public key
    // See overview.md#profile-requirements for URL validation and fetch rules
    profile_url = get_profile_url_from_ucp_agent(request.headers["UCP-Agent"])
    profile = fetch_profile(profile_url)  // HTTPS-only, no redirects
    public_key = find_key_by_kid(profile.signing_keys, keyid)
    if not public_key:
        return error("key_not_found")

    // 3. Verify body digest (if body present)
    if "content-digest" in components:
        expected = "sha-256=:" + base64(sha256(request.body_bytes)) + ":"
        if request.headers["Content-Digest"] != expected:
            return error("digest_mismatch")

    // 4. Reconstruct signature base
    signature_base = build_signature_base(
        components, request.method, request.path, request.query,
        request.headers, keyid
    )

    // 5. Verify signature
    signature = parse_signature(request.headers["Signature"])
    if not ecdsa_verify(signature_base, signature, public_key):
        return error("signature_invalid")

    return success()

    // Note: Replay protection handled by idempotency keys in request payload
```

### REST 응답 검증

응답 검증은 `@method` 대신 `@status`를 쓴다는 점만 빼면 요청 검증과 같습니다.

```text
verify_rest_response(response, signer_profile_url):
    // 1. Parse Signature-Input
    sig_input = parse_signature_input(response.headers["Signature-Input"])
    keyid = sig_input.keyid
    components = sig_input.components

    // 2. Fetch signer's public key
    profile = fetch_profile(signer_profile_url)
    public_key = find_key_by_kid(profile.signing_keys, keyid)
    if not public_key:
        return error("key_not_found")

    // 3. Verify body digest (if body present)
    if "content-digest" in components:
        expected = "sha-256=:" + base64(sha256(response.body_bytes)) + ":"
        if response.headers["Content-Digest"] != expected:
            return error("digest_mismatch")

    // 4. Reconstruct signature base
    signature_base = build_signature_base(
        components, response.status,
        response.headers, keyid
    )

    // 5. Verify signature
    signature = parse_signature(response.headers["Signature"])
    if not ecdsa_verify(signature_base, signature, public_key):
        return error("signature_invalid")

    return success()
```

### 재전송 방지

UCP는 재전송 방지를 서명 계층이 아니라 **비즈니스 계층**에서 멱등성 키로 처리합니다. 관심사는 다음과 같이 나뉩니다.

| 계층       | 책임                       |
| ---------- | -------------------------- |
| **서명**   | 인증(누가), 무결성(무엇을) |
| **멱등성** | 안전한 재시도, 재전송 방지 |

**동작 방식:**

1. 상태를 변경하는 작업은 요청에 `idempotency-key`를 포함합니다
1. 멱등성 키는 서명된 페이로드의 일부입니다
1. 공격자는 서명을 무효화하지 않고서는 키를 수정할 수 없습니다
1. 중복 요청에는 캐시된 응답을 반환합니다(새로운 부수 효과 없음)

**멱등성 키 위치:**

`Idempotency-Key` 헤더는 서명 컴포넌트에 포함됩니다.

```http
POST /checkout-sessions HTTP/1.1
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Signature-Input: sig1=("@method" "@authority" "@path" "idempotency-key" ...);keyid="platform-2026"
Signature: sig1=:MEUCIQD...:
```

**멱등성 키 요건:**

| 요건               | 값                                            |
| ------------------ | --------------------------------------------- |
| **엔트로피**       | 최소 128비트(예: UUID v4, 22자 이상의 영숫자) |
| **고유성**         | 클라이언트별, 작업 유형별                     |
| **서버 저장 기간** | 최소 24시간, 권장 48시간                      |
| **중복 시**        | 캐시된 응답 반환(다시 실행하지 않음)          |
| **저장소 장애 시** | 실패 시 차단(fail closed)(503으로 요청 거부)  |

**참고:** RFC 9421 `created` 파라미터는 선택 사항입니다(**OPTIONAL**). UCP는 재전송 방지를 서명 타임스탬프가 아니라 비즈니스 계층의 멱등성 키로 처리합니다. 오래된 서명을 무효화하는 수단은 키 교체(`signing_keys`에서 유출된 키 제거)입니다.

### 서명 적용 대상

**요청:** HTTP Message Signatures를 사용한다면 플랫폼은 모든 요청에 서명해야 합니다(**SHOULD**). 다른 인증 메커니즘(API 키, OAuth, mTLS)을 대신 사용할 수도 있습니다.

**웹훅:** 웹훅 알림에는 반드시 서명해야 합니다(**MUST**). 그러지 않으면 수신자가 서버에서 시작된 푸시 메시지의 진위를 검증할 수 없습니다.

**그 밖의 응답:** 다음 응답에는 서명을 권장합니다(**RECOMMENDED**).

- 결제 승인 응답
- 체크아웃 완료 응답

다음 응답에는 서명이 선택 사항입니다(**OPTIONAL**).

- 장바구니 작업(저가치, 동기식)
- 카탈로그 쿼리(읽기 전용)
- 에러 응답(4xx, 5xx)

## MCP 전송 방식

UCP는 MCP 전송 방식으로 SSE 기반 전송 대신 **streamable HTTP**를 지정합니다. MCP 요청은 JSON-RPC 본문을 담은 표준 HTTP 요청이므로 같은 RFC 9421 서명 메커니즘이 적용됩니다.

- `Content-Digest` 헤더가 JSON-RPC 메시지 본문을 대상으로 합니다
- `Signature-Input`과 `Signature` 헤더로 인증합니다
- `UCP-Agent`와 `Idempotency-Key` 헤더는 REST와 똑같이 동작합니다

**서명이 포함된 MCP 요청 예시:**

```http
POST /mcp HTTP/1.1
Host: business.example.com
Content-Type: application/json
UCP-Agent: profile="https://platform.example/.well-known/ucp"
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Digest: sha-256=:RK/0qy18MlBSVnWgjwz6lZEWjP/lF5HF9bvEF8FabDg=:
Signature-Input: sig1=("@method" "@authority" "@path" "content-digest" "content-type" "ucp-agent" "idempotency-key");keyid="platform-2026"
Signature: sig1=:MEUCIQDXyK9N3p5Rt...:

{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"complete_checkout","arguments":{"id":"chk_123","checkout":{...}}}}
```

JSON-RPC 메시지가 곧 HTTP 본문입니다. `Content-Digest`가 이 본문을 서명에 결합하므로 JSON 정규화는 필요 없습니다.

## 에러 처리

서명 검증 에러에는 표준 UCP 에러 코드를 사용합니다. 전체 에러 코드 레지스트리와 전송 바인딩은 명세 개요의 [에러 처리](https://dev-jonghoonpark.github.io/ucp-ko/2026-04-08/2026-04-08/specification/overview/#error-handling)를 참고하세요.

**서명 관련 에러:**

| 코드                    | HTTP | 설명                                                    |
| ----------------------- | ---- | ------------------------------------------------------- |
| `signature_missing`     | 401  | 필수 서명 헤더/필드가 없음                              |
| `signature_invalid`     | 401  | 서명 검증 실패                                          |
| `key_not_found`         | 401  | 서명자의 `signing_keys`에서 키 ID를 찾을 수 없음        |
| `digest_mismatch`       | 400  | 본문 다이제스트가 `Content-Digest` 헤더와 일치하지 않음 |
| `algorithm_unsupported` | 400  | 지원하지 않는 서명 알고리즘                             |

**프로필 관련 에러** (기능 협상에도 사용):

| 코드                  | HTTP | 설명                                                  |
| --------------------- | ---- | ----------------------------------------------------- |
| `invalid_profile_url` | 400  | 프로필 URL의 형식이 잘못되었거나 스킴이 유효하지 않음 |
| `profile_unreachable` | 424  | 서명자의 프로필을 가져올 수 없음                      |
| `profile_not_trusted` | 403  | 프로필 URL이 사전 승인된 플랫폼 레지스트리에 없음     |

**참고:** 재전송 방지는 서명 계층이 아니라 비즈니스 계층의 멱등성 키로 처리합니다. 중복 요청에는 서명 에러가 아니라 캐시된 응답이 반환됩니다.

### REST 에러 응답

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "code": "signature_invalid",
  "content": "Request signature verification failed for key kid=platform-2026"
}
```

### MCP 에러 응답

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "error": {
    "code": -32000,
    "message": "Signature verification failed",
    "data": {
      "code": "signature_invalid",
      "content": "Signature verification failed for key kid=platform-2026"
    }
  }
}
```

## 참고 자료

- [RFC 7517](https://datatracker.ietf.org/doc/html/rfc7517) — JSON Web Key (JWK)
- [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421) — HTTP Message Signatures
- [RFC 9530](https://www.rfc-editor.org/rfc/rfc9530) — Digest Fields (Content-Digest)
