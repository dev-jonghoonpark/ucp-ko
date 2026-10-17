<!--
   Copyright 2026 UCP Authors

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->

# Universal Commerce Protocol(UCP) 공식 명세 { #universal-commerce-protocol-ucp-official-specification }

## 전반 지침 { #overarching-guidelines }

이 문서에서 **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, **OPTIONAL** 키워드는
[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.html){ target="_blank" } 및
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174.html){ target="_blank" }에
기술된 대로 해석한다.

스키마 참고 사항:

- 날짜 형식: 별도 명시가 없으면 항상
    [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339.html){ target="_blank" }
    형식으로 지정한다
- 금액 형식: 최소 단위(센트)

## 디스커버리, 거버넌스, 협상 { #discovery-governance-and-negotiation }

UCP는 [프로토콜 버전 선택](#protocol-version)과
[기능 협상](#capability-versions)을 분리한다. 비즈니스는 현재 프로토콜 버전을
광고하고, 지원하는 이전 버전의 프로필로 연결되는 링크를 제공한다. 플랫폼이
정확한 버전 하나를 선택하면, 비즈니스는 양측이 광고한 버전을 바탕으로 활성
기능을 결정한다. 이전 버전을 언제 제거할지를 포함한 버전 수명 주기는 비즈니스의
정책 결정 사항이며, UCP는 지원 중단(deprecation) 일정을 규정하지 않는다.
비즈니스 프로필과 플랫폼 프로필은 양측 모두 캐시할 수 있다.

### 네임스페이스 거버넌스 { #namespace-governance }

UCP는 역도메인(reverse-domain) 명명 방식을 사용해 거버넌스 권한을 기능
식별자에 직접 담는다. 따라서 중앙 레지스트리가 필요 없다.

#### 명명 규칙 { #naming-convention }

모든 기능 이름과 서비스 이름은 반드시 다음 형식을 사용해야 한다(**MUST**).

```text
{reverse-domain}.{service}.{capability}
```

**구성 요소:**

- `{reverse-domain}` - 도메인 소유권에서 도출되는 권한 주체 식별자
- `{service}` - 서비스/버티컬 분류(예: `shopping`, `common`)
- `{capability}` - 구체적인 기능 이름

**예시:**

| 이름                                | 권한 주체   | 서비스   | 기능             |
| ----------------------------------- | ----------- | -------- | ---------------- |
| `dev.ucp.shopping.checkout`         | ucp.dev     | shopping | checkout         |
| `dev.ucp.shopping.fulfillment`      | ucp.dev     | shopping | fulfillment      |
| `dev.ucp.lodging.booking`           | ucp.dev     | lodging  | booking          |
| `dev.ucp.common.identity_linking`   | ucp.dev     | common   | identity_linking |
| `com.example.payments.installments` | example.com | payments | installments     |

#### 권한 바인딩 { #authority-binding }

역도메인 이름은 두 가지 용도로 쓰인다. 하나는 충돌이 없는 **식별자**(키와
참조)이고, 다른 하나는 자신을 기술하는 `schema` URL을 선언하고 이를 가져오게
하는 **엔티티**(기능, 서비스, 결제 핸들러)다. 권한 바인딩은 원격 `schema`를 가진
모든 엔티티에 적용된다. 선언된 `schema` URL의 오리진은 반드시 이름에 담긴
네임스페이스 권한 주체와 일치해야 한다(**MUST**).

기능은 반드시 `schema`를 선언해야 한다(**MUST**). 서비스와 결제 핸들러는 해당
전송 방식이나 핸들러가 스키마를 정의하는 경우에 선언한다. 각 엔티티는 `spec`
URL(사람이 읽는 문서)도 선언할 수 있다(**MAY**).

이 바인딩이 보장하는 것은 **출처(provenance)이지 신뢰가 아니다**. 유효한
바인딩은 역도메인 이름을 해당 도메인 소유자가 통제한다는 사실만 증명한다. 즉
작성자가 통제하지 않는 네임스페이스로는 엔티티를 게시할 수 없다. 이 바인딩은
엔티티가 신뢰할 만하거나, 올바르거나, 지원할 가치가 있다고 주장하지
**않는다**. 협상할지, 신뢰할지, 구현할지는 언제나 클라이언트가 결정하며, 이
바인딩은 그 주장을 *누가* 하는지만 알려준다. 출처는 도메인 소유권에서 확립되고
협상 시점에 평가된다.

`spec` URL은 문서일 뿐 기계적 신뢰 경로에 속하지 않으므로, 그 오리진은 권한
바인딩 대상이 **아니다**. `spec` URL은 반드시 `https`여야 하지만(**MUST**) 어느
호스트에서든 제공할 수 있다(**MAY**)(예: 문서용 서브도메인이나 서드파티 문서
호스트). 아래에 정의하는 권한 바인딩은 `schema` URL에만 적용된다.

##### 도출 알고리즘 { #derivation-algorithm }

권한 주체는 **`schema` URL 호스트에서** 도출한다. 호스트는 소유 도메인을 직접
나타내므로 도메인이 어디서 끝나는지 모호할 일이 없다. 그런 다음 이 권한 주체가
엔티티 이름의 레이블 단위 접두사이거나 이름과 정확히 일치하는지 검증한다. 이름이
`name`인 엔티티의 `schema` URL에 대해 플랫폼은 반드시 다음을 적용해야
한다(**MUST**).

1. 표준을 준수하는(WHATWG) URL 파서로 URL을 파싱한다. URL은 반드시 파싱 가능해야
   하고(**MUST**), 반드시 `https` 스킴을 사용해야 하며(**MUST**), userinfo(
   `user:pass@` 구성 요소)를 포함해서는 안 된다(**MUST NOT**). 원시 URL에 대한
   부분 문자열 매칭은 허용되지 **않는다**(**NOT**). 예를 들어
   `https://ucp.dev@evil.example/x.json`의 호스트는 `ucp.dev`가 아니라
   `evil.example`이다.
2. 호스트는 반드시 레이블이 두 개 이상인 등록된 도메인 이름이어야 한다(**MUST**).
   IP 리터럴 호스트(`https://203.0.113.10/...`)와 단일 레이블 호스트
   (`https://localhost/...`)는 유효한 권한 주체가 아니다.
3. URL의 호스트 이름(포트를 제외한 호스트)을 가져와 정규화하고(소문자로 변환,
   끝의 `.` 제거, 국제화 도메인은 A-label / punycode 형식으로 변환), **레이블
   순서를 뒤집어** `authority_prefix`를 만든다(호스트 `ucp.dev` → `dev.ucp`).
4. 바인딩은 다음 중 **하나**가 성립할 때, 그리고 그때에만 유효하다.
   - **정확히 일치** — `name`이 `authority_prefix`와 같다. 이름 자체가 뒤집은
     호스트이므로, 게시자가 네임스페이스 전체를 통제한다는 사실이 입증된다.
     엔티티의 정체성이 통제하는 도메인 그 자체인 경우의 형태로, 예를 들어
     `pay.example.com`에서 제공되는 결제 핸들러 `com.example.pay`가 여기에
     해당한다(뒤집은 호스트 `com.example.pay`가 이름과 같다).
   - **접두사 일치** — `name`이 `authority_prefix` 뒤에 `.`이 오고 그 뒤에 하나
     이상의 레이블이 이어지는 형태다. 즉 `name`에서 `authority_prefix` 바로 다음
     문자가 `.`이다. 구분자 `.`을 요구하면 매칭이 레이블 경계에서 이루어진다.
     그래서 `com.example`(호스트 `example.com`)이 `com.examplecorp.*` 같은 이웃
     네임스페이스와 매칭되지 않는다. 이 경우 `com.example`은 문자열상 접두사이긴
     하지만 레이블 단위 접두사는 아니다.

권한 바인딩이 확립하는 것은 **출처뿐**이다. 즉 `schema`를 제공하는 당사자가 그
이름을 통제한다는 사실이다. 권한 바인딩은 권한 주체 이외의 레이블을 요구하지
**않는다**. `{reverse-domain}.{service}.{capability}` 형태는 기능 이름과 서비스
이름에 적용되는 별도의 [명명 규칙](#naming-convention)이며(결제 핸들러에는
적용되지 않는다), 이 검사와는 독립적으로 검증한다.

권한 접두사 뒤에 오는 레이블은 이 검사에서 불투명한 값으로 취급하며, 검사하거나
분리하지 않는다.

| 엔티티 이름                         | `schema` 호스트    | `authority_prefix` | 결과                |
| ----------------------------------- | ------------------ | ------------------ | ------------------- |
| `dev.ucp.shopping.checkout`         | `ucp.dev`          | `dev.ucp`          | **허용**(접두사)    |
| `dev.ucp.shopping.checkout`         | `shopping.ucp.dev` | `dev.ucp.shopping` | **허용**(접두사)    |
| `com.example.payments.installments` | `example.com`      | `com.example`      | **허용**(접두사)    |
| `com.example.pay`                   | `pay.example.com`  | `com.example.pay`  | **허용**(정확히 일치) |
| `com.example.pay`                   | `example.com`      | `com.example`      | **허용**(접두사)    |
| `com.example.pay`                   | `evil.example`     | `example.evil`     | **거부**            |
| `dev.ucp.shopping.checkout`         | `evil.example`     | `example.evil`     | **거부**            |
| `com.examplecorp.pay`               | `example.com`      | `com.example`      | **거부**            |
| `com.example.pay`                   | `cdn.example.com`  | `com.example.cdn`  | **거부**            |

엔티티의 `schema`는 뒤집은 레이블이 엔티티 이름과 **같거나** 이름의 **레이블
단위 접두사**인 호스트에서 제공된다. 뒤집은 레이블이 이름과 정확히 같은
호스트(`com.example.pay`의 경우 `pay.example.com`)는 정확히 일치 조건을
충족한다. 정식 apex 호스트(`com.example.*`의 경우 `example.com`)는 접두사 조건을
충족한다. 서브도메인은 그 레이블이 네임스페이스 경로와 맞아떨어질 때만 접두사
조건을 충족한다(`dev.ucp.shopping.*`의 경우 `shopping.ucp.dev`). 상위 도메인의
뒤집은 레이블도 접두사이므로, `com.example.pay` 같은 이름은 정확히 일치하는
호스트(`pay.example.com`)에서든 상위 권한 주체(`example.com`)에서든 똑같이
바인딩된다. 둘 다 통제권을 증명하기 때문이다. 공유 CDN처럼 관련 없는
서브도메인은 어떤 조건도 충족하지 **않는다**. 정식 스키마는 이름과 맞아떨어지는
오리진에서 호스팅해야 한다.

이 검사는 `schema` URL 호스트를 그대로 사용하며
[Public Suffix List](https://publicsuffix.org/)를 참조하지 않는다. 따라서
**공개 접미사(public suffix)**, 즉 서로 무관한 당사자들이 이름을 등록할 수 있는
도메인을 일반 권한 주체와 똑같이 취급한다. 공개 접미사에는 `co.uk`부터, 서드파티가
서브도메인이나 버킷을 등록할 수 있게 해주는 서비스(`github.io`, 객체 스토리지,
앱 플랫폼)가 운영하는 목록의 private 섹션 접미사까지 포함된다. 이런 접미사 아래의
공동 입주자들은 같은 접두사를 충족하므로, 엔티티는 자신이 독점적으로 통제하는
**등록 가능 도메인**(공개 접미사에 레이블 하나를 더한 것) 아래에서만 선언해야
한다.

##### 적용 { #enforcement }

플랫폼은 비즈니스가 선언한 각 `schema` URL을 가져오기 전에 반드시 검증해야
한다(**MUST**). URL의 오리진이 엔티티의 네임스페이스 권한 주체와 일치하지
않으면([도출 알고리즘](#derivation-algorithm) 기준), 플랫폼은 해당 URL을
가져와서는 안 되며(**MUST NOT**) 반드시 해당 엔티티를 거부해야 한다(**MUST**).
거부된 엔티티는 존재하지 않는 것으로 취급하며 절대 활성화하지 않는다. `spec`
URL은 반드시 유효한 `https` URL이어야 한다(**MUST**). 플랫폼은 프로필을 가져올
때와 마찬가지로 `schema` URL을 가져올 때 리디렉션(`3xx`)을 따라가서는 안
된다(**MUST NOT**).

플랫폼은 비즈니스가 선언한 스키마를 가져와 조합한 뒤 모든 요청과 응답을
검증한다. 따라서 바인딩을 검증하면 조합되는 각 스키마가 엔티티 네임스페이스를
소유한 당사자에게서 왔다는 것이 보장된다. 비즈니스도 플랫폼 프로필에 같은 검사를
적용하고 바인딩에 실패한 엔티티를 제외해야 한다(**SHOULD**).

바인딩은 출처 확인을 위해 선언된 호스트 이름을 검증할 뿐, 가져오기 안전성
통제가 **아니며** 역참조를 허가하지도 않는다. `schema` URL을 가져오는 일은
디스커버리 과정에서 가져오는 다른 URL과 마찬가지로 프로토콜의 URL 가져오기 안전성
요구 사항을 추가로 따른다. 이 요구 사항은 호스트 이름만이 아니라 *해석된* 주소를
대상으로, 특수 용도 주소나 클라우드 메타데이터 주소를 노리는 서버 측 요청 위조와
DNS 리바인딩을 막는다. 호스트 이름 검사와 해석된 주소 검사는 서로 독립적이며 둘
다 적용된다.

#### 거버넌스 모델 { #governance-model }

| 네임스페이스 패턴 | 권한 주체    | 거버넌스            |
| ----------------- | ------------ | ------------------- |
| `dev.ucp.*`       | ucp.dev      | UCP 운영 기구       |
| `com.{vendor}.*`  | {vendor}.com | 벤더 조직           |
| `org.{org}.*`     | {org}.org    | 조직                |

`dev.ucp.*` 네임스페이스는 UCP 운영 기구가 승인한 기능용으로 예약되어 있다.
벤더는 커스텀 기능에 반드시 자체 역도메인 네임스페이스를 사용해야 한다(**MUST**).

### 서비스 { #services }

**서비스**는 하나의 버티컬(shopping, common 등)에 대한 API 표면을 정의한다.
서비스는 표준 형식으로 정의되는 오퍼레이션, 이벤트, 전송 바인딩을 포함한다.

- **REST**: OpenAPI 3.x(JSON 형식)
- **MCP**: OpenRPC(JSON 형식)
- **A2A**: Agent Card 명세
- **EP(embedded)**: OpenRPC(JSON 형식)

서비스는 역도메인 레지스트리 키(예: `dev.ucp.shopping`)로 식별한다. 프로필에서
서비스는 그 이름을 키로 삼으며, `services[name][]`의 각 항목은 서비스를 하나의
전송 바인딩과 짝짓고 서비스 `version`을 선언한다. 릴리스 `D`에서 그 버전은
`D`다. 이것은 서비스 버전이지 전송 방식 버전이 아니다. 바인딩에는 별도의 버전이
없다. 바인딩이 참조하는 OpenAPI 또는 OpenRPC 아티팩트에는 자체 `info.version`이
있지만, 이는 릴리스 메타데이터일 뿐 따로 협상하는 버전이 아니다.
[컴포넌트 버전 관리와 릴리스 스냅숏](#component-versioning-and-release-snapshots)을
참고한다.

#### 서비스 정의 { #service-definition }

{{ extension_schema_fields('service.json#/$defs/platform_schema', 'overview') }}

전송 방식 정의는 반드시 얇아야 한다(**MUST**). 즉 메서드 이름을 선언하고 기본
스키마를 참조하기만 한다. 자세한 내용은 [요구 사항](#requirements)을 참고한다.

#### 엔드포인트 해석 { #endpoint-resolution }

`endpoint` 필드는 API 호출의 기본 URL을 제공한다. OpenAPI 경로를 이 엔드포인트
뒤에 붙여 전체 URL을 만든다.

**예시:**

<!-- ucp:example schema=service def=business_schema -->
```json
{
  "version": "{{ ucp_version }}",
  "transport": "rest",
  "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/rest.openapi.json",
  "endpoint": "https://business.example.com/api/v2"
}
```

OpenAPI 경로가 `/checkout-sessions`라면 해석된 URL은 다음과 같다.

```text
POST https://business.example.com/api/v2/checkout-sessions
```

**규칙:**

- `endpoint`는 반드시 스킴(https)을 포함한 유효한 URL이어야 한다(**MUST**)
- `endpoint`는 끝에 슬래시를 두지 않아야 한다(**SHOULD NOT**)
- OpenAPI 경로는 상대 경로이며 endpoint 뒤에 바로 붙는다
- JSON-RPC 호출을 위한 MCP 엔드포인트에도 같은 해석 방식이 적용된다
- A2A 전송 방식의 `endpoint`는 에이전트의 Agent Card URL을 가리킨다

### 기능 { #capabilities }

**기능**(capability)은 서비스 안의 한 가지 기능 단위다. 어떤 기능을 지원하는지,
문서와 스키마를 어디서 찾을 수 있는지를 선언한다.

#### 기능 정의 { #capability-definition }

{{ extension_schema_fields('capability.json#/$defs/platform_schema', 'overview') }}

#### 확장 { #extensions }

**확장**(extension)은 다른 기능을 보강하는 선택적 모듈이다. 확장은 `extends`
필드로 자신의 부모(들)를 선언한다.

<!-- ucp:example schema=profile def=business_schema target=$.ucp.capabilities -->
```json
{
  "dev.ucp.shopping.fulfillment": [
    {
      "version": "{{ ucp_version }}",
      "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/extensions/fulfillment",
      "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/fulfillment.json",
      "extends": "dev.ucp.shopping.checkout"
    }
  ]
}
```

##### 다중 부모 확장 { #multi-parent-extensions }

확장은 배열을 사용해 여러 부모 기능을 확장할 수 있다(**MAY**).

<!-- ucp:example schema=profile def=business_schema target=$.ucp.capabilities -->
```json
{
  "dev.ucp.shopping.discount": [
    {
      "version": "{{ ucp_version }}",
      "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/extensions/discount",
      "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/discount.json",
      "extends": ["dev.ucp.shopping.checkout", "dev.ucp.shopping.cart"]
    }
  ]
}
```

확장이 여러 부모를 선언하는 경우:

- 확장은 자신이 확장하는 기능마다 서로 다른 필드를 정의할 수 있다(**MAY**)
    (예: 체크아웃에는 `loyalty_earned`, 장바구니에는 `loyalty_preview`)
- 협상 규칙은 [교집합 알고리즘](#intersection-algorithm)을 참고한다

확장의 종류는 다음과 같다.

- **공식**: `dev.ucp.shopping.fulfillment`는 `dev.ucp.shopping.checkout`을 확장한다
- **벤더**: `com.example.installments`는 `dev.ucp.shopping.checkout`을 확장한다

### 스키마 조합 { #schema-composition }

확장은 새 필드를 추가하고 공유 구조를 수정할 수 있다(예: 할인은 `totals`를
수정하고, 풀필먼트는 `totals.type`에 fulfillment를 추가한다).

#### 요구 사항 { #requirements }

- 전송 방식 정의(OpenAPI/OpenRPC)는 반드시 기본 스키마만 참조해야
    한다(**MUST**). 필드를 나열하거나 페이로드 형태를 인라인으로 정의해서는 안
    된다(**MUST NOT**).
- 확장은 반드시 자기 기술적이어야 한다(**MUST**). 각 확장 스키마는 자신이
    도입하는 타입과, `allOf` 조합으로 기본 타입을 어떻게 수정하는지를 반드시
    선언해야 한다(**MUST**).
- 플랫폼은 기본 스키마와 활성 확장 스키마를 가져와 조합함으로써 반드시
    클라이언트 측에서 스키마를 해석해야 한다(**MUST**).

#### 확장 스키마 패턴 { #extension-schema-pattern }

확장 스키마는 `allOf`로 조합 타입을 정의한다. 결정적인 스키마 해석이 가능하도록
`$defs` 키에는 반드시 부모 기능의 전체 이름(역도메인 형식)을 사용해야
한다(**MUST**).

<!-- ucp:example skip reason="schema definition" -->
```json
{
  "$defs": {
    "discounts_object": { ... },
    "dev.ucp.shopping.checkout": {
      "title": "Checkout with Discount",
      "allOf": [
        {"$ref": "checkout.json"},
        {
          "type": "object",
          "properties": {
            "discounts": {
              "$ref": "#/$defs/discounts_object"
            }
          }
        }
      ]
    }
  }
}
```

**요구 사항:**

- 확장 스키마에는 반드시 `extends`에 선언한 각 부모에 대한 `$defs` 항목이
    있어야 한다(**MUST**)
- `$defs` 키는 반드시 부모 기능의 전체 이름과 정확히 일치해야 한다(**MUST**)

이 규칙으로 다음이 보장된다.

- **자기 문서화**: 스키마가 어떤 부모를 확장하는지 정확히 선언한다
- **결정적 해석**: `extends` 값이 `$defs` 키에 직접 대응한다
- **검증 가능성**: 빌드 시점 검사로 각 `extends` 항목에 대응하는 `$defs` 키가
    있는지 확인할 수 있다

##### 버전 요구 사항 { #version-requirements }

확장 작성자는 확장 스키마에 `requires` 객체를 선언해(`name`, `title`,
`description`과 나란히) 확장이 의존하는 버전을 명시해야 한다(**SHOULD**).
서드파티 확장 스키마는 작성자가 관리하는 자체 `version`을 선언하며, 이 버전은
`ucp.version`과 독립적으로 올라간다. UCP가 작성한 `dev.ucp.*` 확장은 릴리스
`D`에서 버전 `D`를 선언한다. `requires.protocol`은 선택된 `ucp.version`을
제약하고, `requires.capabilities`는 이름이 지정된 기능의 선택된 버전을 제약한다.

<!-- ucp:example skip reason="schema definition" -->
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://acme.com/ucp/schemas/loyalty.json",
  "name": "com.acme.shopping.loyalty",
  "version": "2026-06-15",
  "title": "Acme Loyalty Points",
  "requires": {
    "protocol": { "min": "2026-01-23" },
    "capabilities": {
      "dev.ucp.shopping.checkout": { "min": "2026-06-01" }
    }
  },
  "$defs": {
    "dev.ucp.shopping.checkout": { ... }
  }
}
```

버전 요구 사항은 프로필 게시자가 아니라 스키마 작성자가 선언한다. 프로필
게시자는 호환되는 버전을 골라 자신의 프로필에 광고한다.

각 제약은 필수인 `min`(포함)과 선택 사항인 `max`(포함) 버전을 가진 객체다.
`max`가 없으면 상한이 없다.

<!-- ucp:example skip reason="schema definition" -->
```json
"requires": {
  "protocol": { "min": "2026-01-23", "max": "2026-09-01" },
  "capabilities": {
    "dev.ucp.shopping.checkout": { "min": "2026-06-01" }
  }
}
```

`requires.capabilities`의 키는 반드시 확장의 `$defs` 키의 부분집합이어야
한다(**MUST**). 이 범위는 정확한 버전이 선택된 뒤 의존성을 검증하는 용도이며,
버전을 선택하지는 않는다. `ucp.version`은 먼저
[프로필 선택](#protocol-version)으로 확정되고, 기능 버전은
[교집합 알고리즘](#intersection-algorithm)으로 확정된다. 그런 다음 `requires`가
있으면 플랫폼과 비즈니스는 스키마 해석 중에 선택된 `ucp.version`과 기능 버전이
선언된 제약을 충족하는지 반드시 검증해야 한다(**MUST**). 호환되지 않는 확장은
활성 기능 집합에서 제외된다([해석 흐름](#resolution-flow) 참고). `requires`가
없으면 해당 확장은 프로필이 선언한 버전과 호환된다고 간주한다.

#### 스키마 해석 규칙 { #schema-resolution-convention }

페이로드를 검증하려면 구현체는 다음과 같이 확장 스키마를 해석한다.

1. 오퍼레이션에서 루트 기능을 결정한다(예: 체크아웃 오퍼레이션은
    `dev.ucp.shopping.checkout`을 사용한다)
2. 각 활성 확장에 대해 해당 확장의 `$defs[{root_capability}]`를 해석해 적용한다

**예시:** 체크아웃 응답에 할인 확장이 포함되는 경우.

- 루트 기능: `dev.ucp.shopping.checkout`
- 확장 스키마: `discount.json`
- 해석 대상: `discount.json#/$defs/dev.ucp.shopping.checkout`

#### 해석 흐름 { #resolution-flow }

플랫폼은 반드시 다음 순서에 따라 스키마를 해석해야 한다(**MUST**).

1. **디스커버리**: `/.well-known/ucp`에서 비즈니스 프로필을 가져온다
2. **협상**: 기능 교집합을 계산한다
    ([교집합 알고리즘](#intersection-algorithm) 참고)
3. **스키마 가져오기**: 기본 스키마와 모든 활성 확장 스키마를 가져온다
4. **버전 호환성**: 가져온 각 확장 스키마에 `requires`가 있으면, 협상된
    프로토콜 버전과 기능 버전이 선언된 제약을 충족하는지 검증한다. 호환되지 않는
    확장을 제외하고, 고아가 된 확장을 다시 정리한다
    ([교집합 알고리즘](#intersection-algorithm)의 3-4단계)
5. **조합**: 활성 확장을 기준으로 `allOf` 체인을 통해 스키마를 병합한다
6. **검증**: 조합된 스키마로 요청과 응답을 검증한다

### 프로필 구조 { #profile-structure }

프로필 문서는 기계가 읽을 수 있는 디스커버리 문서다. 비즈니스는
`/.well-known/ucp`에 프로필을 게시하고, 플랫폼은 `UCP-Agent`에 광고한 URI에
프로필을 게시한다.

프로필 문서는 필수 멤버 `ucp`를 가진 JSON 객체다. `ucp` 멤버에는 프로토콜
메타데이터, 즉 프로토콜 버전, 서비스, 선택적 기능, 결제 핸들러가 담긴다.

비즈니스 프로필과 플랫폼 프로필 모두 `ucp.version`, `ucp.services`,
`ucp.payment_handlers`는 필수다. `services`와 `payment_handlers` 레지스트리는
비어 있더라도 반드시 존재해야 한다(**MUST**). `ucp.capabilities`는 선택 사항이며
생략할 수 있다(**MAY**). 다만 실제로 쓸모 있는 커머스 프로필이라면 보통 기능을
하나 이상 광고한다.

프로필에는 HTTP Message Signatures와 서명된 웹훅에 쓰이는 공개 JSON Web Key를
포함할 수 있다(**MAY**). 프로필이 서명 키를 게시하는 경우, 키는 반드시 최상위
`keys[]` 배열에 있어야 한다(**MUST**). 이 배열은 모든 UCP 검증자가 읽는 정식 UCP
프로필 필드다. `keys[]`는
[RFC 7517](https://datatracker.ietf.org/doc/html/rfc7517)에 따른 JWK Set이므로,
같은 문서가 UCP 프로필이면서 동시에 유효한 JWK Set이 된다. 서명자는 이를 Web Bot
Auth 키 소스로 재사용할 수 있다. 아래의
[WBA 상호 운용을 위한 배포 패턴](#deployment-patterns-for-wba-interop)을
참고한다.

키를 추가하거나, 교체하거나, 제거하면 이 배열 하나만 갱신하면 된다. 보안상 가장
중요한 경우는 제거다. 폐기되었거나 유출된 키는 `keys[]`에서 빠지기 전까지는
실질적으로 폐기된 것이 아니다.

UCP는 두 가지 잘 알려진 키 타입, 즉 **EC**(ECDSA P-256, P-384)와
**OKP**(EdDSA Ed25519)를 정의한다. 키 타입, 곡선, 알고리즘 어휘는 열려 있으며,
검증자는 인식하지 못하는 키를 건너뛴다. 키 형식, 알고리즘, 조회, 교체에 대해서는
[메시지 서명](../signatures.md)을 참고한다.

#### 비즈니스 프로필 { #business-profile }

비즈니스는 `/.well-known/ucp`에 프로필을 게시한다. 예시는 다음과 같다.

<!-- ucp:example schema=profile def=business_schema -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview/",
          "transport": "rest",
          "endpoint": "https://business.example.com/ucp/v1",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/rest.openapi.json"
        },
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview/",
          "transport": "mcp",
          "endpoint": "https://business.example.com/ucp/mcp",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/mcp.openrpc.json"
        },
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview/",
          "transport": "a2a",
          "endpoint": "https://business.example.com/.well-known/agent-card.json"
        },
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview/",
          "transport": "embedded",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/embedded.openrpc.json"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.checkout": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/checkout",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/checkout.json"
        }
      ],
      "dev.ucp.shopping.fulfillment": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/extensions/fulfillment",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/fulfillment.json",
          "extends": "dev.ucp.shopping.checkout"
        }
      ],
      "dev.ucp.shopping.discount": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/extensions/discount",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/discount.json",
          "extends": "dev.ucp.shopping.checkout"
        }
      ],
      "dev.ucp.common.identity_linking": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/common/identity-linking/",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/common/identity_linking.json",
          "config": {
            "providers": {
              "com.example.idp": [
                { "type": "oauth2", "auth_url": "https://accounts.example.com/" }
              ]
            },
            "scopes": {
              "dev.ucp.shopping.order:read":   {},
              "dev.ucp.shopping.order:manage": {}
            }
          }
        }
      ]
    },
    "payment_handlers": {
      "com.example.processor_tokenizer": [
        {
          "id": "processor_tokenizer",
          "version": "{{ ucp_version }}",
          "spec": "https://example.com/specs/payments/processor_tokenizer",
          "schema": "https://example.com/specs/payments/merchant_tokenizer.json",
          "available_instruments": [
            {
              "type": "card",
              "constraints": {
                "properties": { "brand": { "enum": ["visa", "mastercard", "amex"] } }
              }
            }
          ],
          "config": {
            "type": "CARD",
            "tokenization_specification": {
              "type": "PUSH",
              "parameters": {
                "token_retrieval_url": "https://api.psp.example.com/v1/tokens"
              }
            }
          }
        }
      ]
    }
  },
  "keys": [
    {
      "kid": "poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U",
      "kty": "OKP",
      "crv": "Ed25519",
      "x": "JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw3c5D0bs",
      "use": "sig",
      "alg": "EdDSA"
    },
    {
      "kid": "business_2025",
      "kty": "EC",
      "crv": "P-256",
      "x": "qIVYZVLCrPZHGHjP17CTW0_-D9Lfw0EkjqF7xB4FivA",
      "y": "Mc4nN9LTDOBhfoUeg8Ye9WedFRhnZXZJA12Qp0zZ6F0",
      "use": "sig",
      "alg": "ES256"
    }
  ]
}
```

비즈니스 프로필은 비즈니스가 제공하는 전송 방식, 기능, 결제 핸들러, 공개 검증
키를 광고한다. 이 예시는 서명 키를 정식 최상위 `keys[]` 배열(RFC 7517 JWK
Set)에 게시하므로, 같은 문서가 유효한 JWK Set이기도 하며 Web Bot Auth 키
소스로 재사용할 수 있다. 모든 UCP 검증자는 키를 `UCP-Agent`로 해석했든
`Signature-Agent`로 해석했든 `keys[]`를 읽는다.

WBA 형태의 검증자는 **`Signature-Agent` 헤더가** 프로필 URL을 가리키는
`type=jwks_uri`(또는 `type=cimd`)로 **이 프로필을 선택한 경우에만** 여기서
`keys[]`를 읽는다. 기본값인 `type=directory`(`type`을 생략한 경우)는 정적
프로필이 아니라 `/.well-known/http-message-signatures-directory`에 있는 *서명된*
디렉터리 문서를 기대하므로, 정적 `/.well-known/ucp`에서 `keys[]`를 읽지 않는다.
[WBA 상호 운용을 위한 배포 패턴](#deployment-patterns-for-wba-interop)을
참고한다.

이 예시는 키 두 개를 사용한다. 배포 환경에 키가 하나 필요한지 두 개 필요한지는
상대방이 받아들이는 알고리즘에 달려 있으며, 대부분은 하나면 충분하다.
[서명 알고리즘](../signatures.md#signature-algorithms)을 참고한다. 여기서 쓴 두
키는 다음과 같다.

- HTTP 전송 계층 신원 확인용 **Ed25519** 키(OKP)로, WBA와 호환된다.
  `kid`는 RFC 7638에 따른 JWK SHA-256 Thumbprint다.
- AP2 위임(mandate) 서명(`ap2.merchant_authorization`)용 **ECDSA P-256**
  키(EC).

AP2나 WBA와 연동하지 않는 비즈니스는 `keys[]`에 ES256 키 하나(범용 기준선)만
게시해도 된다. 키 조회와 해석은 [키 디스커버리](#key-discovery)를, 호스팅 선택지는
[WBA 상호 운용을 위한 배포 패턴](#deployment-patterns-for-wba-interop)을, 서명
방식은 [메시지 서명](../signatures.md)을 참고한다.

이전 프로토콜 버전을 지원하는 비즈니스는 각 이전 버전을 버전별 프로필 URI에
매핑하는 `supported_versions` 객체를 포함해야 한다(**SHOULD**). 자세한 내용은
[프로토콜 버전](#protocol-version)을 참고한다.

#### 플랫폼 프로필 { #platform-profile }

플랫폼 프로필도 비슷하며, 암호학적 검증이 필요한 기능을 위한 서명 키를
포함한다. 기능에는 기능별 설정(예: 콜백 URL, 기능 플래그)을 위한 `config` 객체를
포함할 수 있다(**MAY**). 예시는 다음과 같다.

<!-- ucp:example schema=profile def=platform_schema -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "services": {
      "dev.ucp.shopping": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/overview/",
          "transport": "rest",
          "schema": "https://ucp.dev/{{ ucp_version }}/services/shopping/rest.openapi.json",
          "endpoint": "https://platform.example.com/ucp/v1"
        }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.checkout": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/checkout",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/checkout.json"
        }
      ],
      "dev.ucp.shopping.fulfillment": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/extensions/fulfillment",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/fulfillment.json",
          "extends": "dev.ucp.shopping.checkout"
        }
      ],
      "dev.ucp.shopping.order": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/shopping/order",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/shopping/order.json",
          "config": {
            "webhook_url": "https://platform.example.com/webhooks/ucp/orders"
          }
        }
      ],
      "dev.ucp.common.identity_linking": [
        {
          "version": "{{ ucp_version }}",
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/common/identity-linking/",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/common/identity_linking.json"
        }
      ]
    },
    "payment_handlers": {
      "com.google.pay": [
        {
          "id": "gpay_1234",
          "version": "2024-12-03",
          "spec": "https://developers.google.com/merchant/ucp/guides/gpay-payment-handler",
          "schema": "https://pay.google.com/gp/p/ucp/2026-01-11/schemas/gpay_config.json"
        }
      ],
      "dev.shopify.shop_pay": [
        {
          "id": "shop_pay_1234",
          "version": "{{ ucp_version }}",
          "spec": "https://shopify.dev/ucp/shop-pay-handler",
          "schema": "https://shopify.dev/ucp/schemas/shop-pay-config.json",
          "available_instruments": [
            {"type": "shop_pay"}
          ]
        }
      ],
      "com.example.processor_tokenizer": [
        {
          "id": "processor_tokenizer",
          "version": "{{ ucp_version }}",
          "spec": "https://example.com/specs/payments/processor_tokenizer-payment",
          "schema": "https://example.com/schemas/payments/delegate-payment.json",
          "available_instruments": [
            {"type": "card", "constraints": {"properties": {"brand": {"enum": ["visa", "mastercard"]}}}}
          ]
        }
      ]
    }
  },
  "keys": [
    {
      "kid": "platform_2025",
      "kty": "EC",
      "crv": "P-256",
      "x": "MKBCTNIcKUSDii11ySs3526iDZ8AiTo7Tu6KPAqv7D4",
      "y": "4Etl6SRW2YiLUrN5vfvVHuhp7x8PxltmWWlbbM4IFyM",
      "use": "sig",
      "alg": "ES256"
    }
  ]
}
```

### `ucp` 프로토콜 네임스페이스 { #the-ucp-protocol-namespace }

멤버 이름 `ucp`는 모든 구조화된 UCP 객체 스코프(멤버가 스키마로 정의된 필드인
객체)에서 **프로토콜 네임스페이스**로 예약되어 있다. 프로필과 응답이 담는 최상위
`ucp` 멤버(앞의 [프로필 구조](#profile-structure)에서 설명)는 특별한 래퍼가
아니라, 이 예약이 루트에서 드러난 형태, 즉 루트 객체의 예약 멤버다. 이 예약은
딕셔너리 컨테이너에는 적용되지 않는다. 딕셔너리의 키는 필드가 아니라 데이터이기
때문이다. `ucp`라는 이름의 딕셔너리 키는 평범한 데이터다. 딕셔너리 값으로 쓰인
구조화된 객체는 여전히 적격 스코프다. 스키마 작성자는 구조화된 객체 스키마나
확장에 `ucp`라는 이름의 도메인 필드를 정의해서는 안 된다(**MUST NOT**).

적격한 각 구조화 스코프에서 `ucp`는 그 스코프에 대한 프로토콜의 진술을 담는다.
루트에서는 프로토콜 메타데이터(버전, 서비스, 기능, 결제 핸들러)를, 그 밖에서는
[`map_order`](#map_order) 같은 구조적 주석을 담는다.

**개방성.** `ucp` 컨테이너는 열려 있다. 소비자는 `ucp` 안에서 인식하지 못하는
멤버를 반드시 무시해야 한다(**MUST**)(관대한 독자). 개방성은 더 새로운 UCP
버전으로 만든 문서를 이전 소비자도 읽을 수 있게 하려고 존재하며, 확장 공간이
*아니다*. `ucp` 안의 멤버는 UCP 코어만 정의하며, 확장 작성자는 그곳에 확장
데이터를 넣어서는 안 된다(**MUST NOT**). `ucp` 안의 인식하지 못한 멤버는 "더
새로운 UCP 버전이 정의한 것"을 뜻하며, 절대 "확장 데이터"를 뜻하지 않는다.

**직접 재귀 금지.** 생산자는 `ucp` 멤버를 다른 `ucp` 멤버의 직계 자식(`ucp.ucp`)으로
내보내서는 안 된다(**MUST NOT**). 그런 멤버가 있으면, 이를 받은 비즈니스나
플랫폼은 그것을 또 다른 프로토콜 네임스페이스로 해석해서는 안 되며(**MUST NOT**)
반드시 그 자식을 무시해야 한다(**MUST**). 네임스페이스 아래의 구조화된 객체(예:
기능의 `config`)는 여전히 자체 `ucp` 멤버를 가질 수 있다.

**주변 어휘.** 프로토콜 네임스페이스는 구조화된 UCP 객체 안 어디에나 존재할 수
있다. 비즈니스나 플랫폼은 적격한 어느 구조화 스코프에든 `ucp` 멤버를 포함할 수
있으며(**MAY**), 그 내용은 오직 UCP 코어 어휘로만 정의된다. 이 멤버는 이름 예약
자체와 마찬가지로 UCP 문서 문법의 일부다. 예약은 딕셔너리 컨테이너에서 멈춘다.
비즈니스나 플랫폼은 `ucp`라는 이름의 딕셔너리 키를 프로토콜 네임스페이스로
해석해서는 안 된다(**MUST NOT**). 그 키와 값은 평범한 딕셔너리 데이터다. 예를
들어 `attribution`은 문자열 값으로 이루어진 딕셔너리이므로, `ucp`라는 이름의
attribution 키는 프로토콜 네임스페이스 멤버가 아니라 평범한 attribution 데이터다.
이 예약 안에서 작업하는 스키마 작성자를 위한 지침은 스키마 작성 가이드의
[예약된 `ucp` 멤버](/documentation/schema-authoring/#the-reserved-ucp-member)
섹션에 있다. 적격한 구조화 스코프에서 `ucp` 멤버를 만난 비즈니스나 플랫폼은
인식하는 멤버를 각각의 정의에 따라 처리하고, 인식하지 못하는 멤버는 반드시
무시해야 한다(**MUST**)(위의 *개방성* 참고). 어떤 멤버가 어휘에 들어가려면 무시해도
안전해야 한다. 그 멤버를 처리하지 않는 비즈니스나 플랫폼은 그 멤버가 주는 이점만
잃을 뿐, 정확성을 잃어서는 안 된다. 예를 들어 `map_order`를 무시하는 비즈니스나
플랫폼은 그냥 맵을 순서 없이 순회한다. 순서 지정이 생기기 전의 기존 상태와 같다.

**스코프가 의무를 결정한다.** 프로필과 응답의 루트에서 `ucp` 봉투는 이 문서의
다른 곳에서 명시한 그대로 필수 프로토콜 메타데이터를 추가로 담는다. 이 섹션은 그
의무를 하나도 바꾸지 않는다. 비즈니스나 플랫폼은 그 밖의 모든 적격 구조화
스코프에서 이 멤버를 생략할 수 있다(**MAY**). 딕셔너리 컨테이너는 적격 스코프가
아니며 프로토콜 네임스페이스 의무를 지지 않는다. 어휘 준수 여부는 열린 UCP 소스
스키마에 대한 일반적인 인스턴스 검증이 아니라 이 명세의 처리 규칙으로 정의된다.
일반 검증은 주변 `ucp` 멤버를 무시되는 알 수 없는 객체로 취급한다.

**어휘 적용 범위.** 등록된 각 프로토콜 네임스페이스 멤버는 자신이 적용되는 문서
맥락과 메시지 방향을 정의한다.

**스키마 처리.** UCP 소스 스키마는 기본적으로 열려 있으므로, 이에 대한 일반
검증은 프로토콜 어휘를 적용하지 않은 채 주변 `ucp`를 받아들일 수 있다. 선택된
메시지 방향에 대해, UCP를 인식하는 해석기는 반드시 모든 적격 구조화 스코프에서
주변 `ucp`를 인식하고 `ucp.json#/$defs/members`의 중앙 어휘(선택된 방향에서의
해당 어휘 적용 범위를 따름)에 대해 검증하는 해석된 스키마를 만들어야
한다(**MUST**). 해석된 스키마가 다른 알 수 없는 도메인 필드를 거부하더라도, `ucp`
네임스페이스는 전방 호환성을 위해 인식하지 못한 멤버에 열린 채로 남는다. 그 결과물은
표준 검증기와 코드 생성기가 사용할 수 있는 일반 JSON Schema다.

#### `map_order` { #map_order }

JSON 객체 멤버에는 순서가 없다. 멤버 순서는 파싱 후에도 유지된다는 보장이 없으며,
UCP 서명이 의존하는
[RFC 8785](https://www.rfc-editor.org/rfc/rfc8785.html){ target="_blank" }
(JSON Canonicalization Scheme)는 배열 요소 순서는 보존하지만 객체 멤버 이름은
정렬한다. `map_order`는 배열을 사용하므로 선언된 순서가 정규화와 서명을 거쳐도
유지된다.

`map_order`는 자신을 담고 있는 `ucp` 멤버가 주석을 다는 스코프 안의 맵 값 필드에
대해 선호하는 키 순회 순서를 선언한다. 중첩 스코프에서는 `map_order`의 각 키가
`ucp`를 담고 있는 객체의 맵 필드 이름을 가리킨다. 문서 루트에서는 각 키가 루트
`ucp` 봉투 안의 형제 맵 필드 이름을 가리킨다. 체크아웃 응답의 `actions`처럼 `ucp`
밖에 있는 루트 도메인 필드는 대상이 아니다. 각 값은 대상 맵의 키를 선호 순회
순서대로 나열한 배열이다.

`map_order`는 UCP 오퍼레이션 요청에는 적용되지 않는다.

대상 맵 필드 `<field>`와 그에 짝을 이루는 배열 `map_order.<field>`에 대해:

1. 생산자는 UCP의 맵 값 레지스트리에 대해 JSON 객체 멤버 순서에 의존해서는 안
    된다(**MUST NOT**). 순서를 전달하는 것은 `map_order`다.
2. `map_order.<field>`는 대상 맵 필드 `<field>`의 키를 선호 순회 순서대로 담는다.
3. 순서 배열은 일부만 담을 수 있다(**MAY**). 나열된 키를 배열 순서대로 먼저
    순회한다.
4. 나열되지 않은 맵 키도 여전히 유효하고 사용할 수 있다. 소비자는 나열된 키 다음에
    이 키들을 순회하며, 필드가 정의한 대체 순서를 사용하거나, 필드에 정의된 순서가
    없으면
    [RFC 8785가 정의하는 속성 이름 순서](https://www.rfc-editor.org/rfc/rfc8785.html#section-3.2.3){ target="_blank" }를
    사용한다.
5. 순서 배열은 허용 목록이 아니다. 소비자는 키가 빠진 것을 제거, 부적격, 지원 축소로
    해석해서는 안 된다(**MUST NOT**).
6. 생산자는 반드시 존재하는 맵 값 대상을 지정해야 하고(**MUST**), 그 대상 맵에
    존재하는 키만 나열해야 한다(**MUST**). 이를 받는 비즈니스나 플랫폼은 항목이
    존재하지 않거나, 인식하지 못하거나, 맵이 아닌 대상을 지정했다는 이유만으로, 또는
    대상 맵에 없는 키를 지정했다는 이유만으로 그 항목을 담은 문서를 거부해서는 안
    된다(**MUST NOT**). `map_order`를 처리한다면 사용할 수 없는 항목은 반드시
    무시해야 한다(**MUST**).
7. 생산자는 한 순서 배열에 같은 맵 키를 두 번 이상 나열해서는 안 된다(**MUST NOT**).
    소비자는 순서 배열에 맵 키가 반복된다는 이유만으로 그 배열을 담은 문서를
    거부해서는 안 된다(**MUST NOT**). `map_order`를 처리하는 소비자는 반드시 처음
    나온 항목을 따르고 이후 반복은 무시해야 한다(**MUST**).
8. `map_order`가 없거나 해당 필드에 대한 항목이 없으면 순서가 선언되지 않은
    것이다. 소비자는 객체 멤버 순서로 대신해서는 안 된다(**MUST NOT**).
9. 순서대로 순회한다는 것이 해당 필드에서 무엇을 *의미*하는지(표시 순서, 협상
    우선순위 등)는 그 필드 자체의 명세가 정의한다. `map_order`는 순서만 전달하고
    그 밖에는 아무것도 전달하지 않는다.

규칙 3–5는 나열되지 않은 키를 오류로 보거나 버리는 관례와 의도적으로 다르다. 일부만
담은 목록은 언제나 유효하며, 나열되지 않은 키는 언제나 유지된다.

결제 핸들러의 순서를 지정하는 비즈니스 프로필 예시:

<!-- ucp:example schema=profile def=business_schema -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "services": { ... },
    "payment_handlers": {
      "com.google.pay": [
        { "id": "gpay", "version": "{{ ucp_version }}" }
      ],
      "dev.shopify.shop_pay": [
        { "id": "shop_pay", "version": "{{ ucp_version }}" }
      ]
    },
    "map_order": {
      "payment_handlers": ["dev.shopify.shop_pay", "com.google.pay"]
    }
  }
}
```

`map_order`는 스코프에 구애받지 않는다. 같은 메커니즘으로 어느 적격 구조화
스코프에서든 형제 맵의 순서를 지정할 수 있다. 예를 들어 비즈니스가 기능의
`config` 안에 있는 신원 제공자 레지스트리의 순서를 지정하는 경우는 다음과 같다.

<!-- ucp:example skip reason="illustrative fragment" -->
```json
{
  "providers": {
    "app.example.login": [
      {"type": "oauth2", "auth_url": "https://login.example.app"}
    ],
    "com.google": [
      {"type": "oauth2", "auth_url": "https://accounts.google.com"}
    ]
  },
  "ucp": {
    "map_order": {"providers": ["app.example.login", "com.google"]}
  }
}
```

순서가 무엇을 *의미*하는지는 여전히 필드마다 다르다(규칙 9). 현재 정의된 순회
의미는 `payment_handlers`에 대한 비즈니스의 표시 선호 순서 하나뿐이다.
[결제 핸들러](#payment-handlers)를 참고한다.

### 요청 시 플랫폼 광고 { #platform-advertisement-on-request }

플랫폼은 기능 협상이 가능하도록 모든 요청에 반드시 자신의 프로필 URI를 전달해야
한다(**MUST**).

**HTTP 전송 방식:** 플랫폼은 UCP-Agent 헤더에 반드시 Dictionary Structured Field
문법([RFC 8941](https://datatracker.ietf.org/doc/html/rfc8941){ target="_blank" })을
사용해야 한다(**MUST**).

```text
POST /checkout HTTP/1.1
UCP-Agent: profile="https://agent.example/profiles/shopping-agent.json"
Content-Type: application/json

{"line_items": [...]}
```

**MCP 전송 방식:** 플랫폼은 반드시 요청 메타데이터를 담은 `meta` 객체를 포함해야
한다(**MUST**).

<!-- ucp:example schema=shopping/checkout op=create direction=request extract=$.params.arguments.checkout -->
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "create_checkout",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://agent.example/profiles/shopping-agent.json"
        }
      },
      "checkout": {
        "line_items": [...]
      }
    }
  },
  "id": 1
}
```

### 협상 프로토콜 { #negotiation-protocol }

#### 플랫폼 요구 사항 { #platform-requirements }

1. **프로필 광고**: 플랫폼은 전송 방식에 맞는 메커니즘을 사용해 모든 요청에
    반드시 자신의 프로필 URI를 포함해야 한다(**MUST**).
2. **디스커버리**: 플랫폼은 요청을 시작하기 전에 `/.well-known/ucp`에서 비즈니스
    프로필을 가져올 수 있다(**MAY**). 가져왔다면 플랫폼은 HTTP cache-control
    지시어에 따라 프로필을 캐시해야 한다(**SHOULD**).
3. **네임스페이스 검증**: 플랫폼은 가져오기 전에 각 기능의 `schema` URL 오리진이
    네임스페이스 권한 주체와 일치하는지 반드시 검증해야 하며(**MUST**)
    ([권한 바인딩](#authority-binding) 참고), 이 바인딩에 실패한 기능은 반드시
    거부해야 한다(**MUST**).
4. **스키마 해석**: 플랫폼은 요청을 보내기 전에 협상된 기능의 스키마를 반드시
    가져와 조합해야 한다(**MUST**).

#### 비즈니스 요구 사항 { #business-requirements }

1. **프로필 해석**: 플랫폼 프로필 URI가 담긴 요청을 받으면, 비즈니스는 이미 캐시된
    경우가 아니라면 반드시 플랫폼 프로필을 가져와 검증해야 한다(**MUST**).
    비즈니스는 기능 이름으로 협상하고 자체 스키마를 제공하므로, 보통은 플랫폼이
    선언한 `schema` URL을 역참조하지 않는다. 그렇더라도 심층 방어 차원에서
    네임스페이스 바인딩을 검증해야 한다(**SHOULD**)
    ([권한 바인딩](#authority-binding) 참고).
2. **기능 교집합**: 비즈니스는 반드시 플랫폼 기능과 비즈니스 기능의 교집합을
    계산해야 한다(**MUST**).
3. **확장 검증**: 교집합에 부모 기능이 없는 확장은 반드시 제외해야 한다(**MUST**).
4. **응답 요구 사항**: 비즈니스는 모든 응답에 다음을 담은 `ucp` 필드를 반드시
    포함해야 한다(**MUST**).
    - `version`: 요청 처리에 사용한 UCP 버전
    - `capabilities`: 이 응답에서 활성화된 기능의 배열

#### 교집합 알고리즘 { #intersection-algorithm }

기능 교집합 알고리즘은 세션에서 어떤 기능이 활성화될지 결정한다.

1. **교집합 계산**: 각 비즈니스 기능에 대해, 같은 `name`을 가진 플랫폼 기능이
    있으면 결과에 포함한다.

2. **버전 선택**: 교집합에 있는 각 기능에 대해, 비즈니스 배열과 플랫폼 배열
    **양쪽에** 존재하는 버전 문자열의 집합을 계산한다. 집합이 비어 있지 않으면
    **가장 높은** 버전(가장 최근 날짜)을 선택한다. 집합이 비어 있으면(공통 버전
    없음) 해당 기능을 교집합에서 **제외한다**.

3. **고아 확장 정리**: `extends`가 설정되어 있지만 부모 기능 중 **어느 것도**
    교집합에 없는 기능을 제거한다.
    - 단일 부모 확장(`extends: "string"`): 부모가 반드시 있어야 한다
    - 다중 부모 확장(`extends: ["a", "b"]`): 부모가 적어도 하나는 있어야
        한다

4. **정리 반복**: 더 이상 제거되는 기능이 없을 때까지 3단계를 계속한다(전이적인
    확장 체인을 처리한다).

그 결과는 양측이 서로 호환되는 버전으로 지원하면서 확장 의존성까지 충족된 기능의
집합이다.

#### 오류 처리 { #error-handling }

UCP 협상은 두 가지 방식으로 실패할 수 있다.

1. **디스커버리 실패**: 비즈니스가 플랫폼의 프로필을 가져오거나 파싱할 수 없다.

2. **협상 실패**: 제공된 프로필은 유효하지만 기능 교집합이 비어 있거나 버전이
   호환되지 않는다.

디스커버리 실패는 전송 계층 오류다. 필요한 입력을 가져올 수 없었거나 입력 형식이
잘못된 경우다. 협상 실패는 비즈니스 결과다. 핸들러가 제공된 입력으로 실행된 뒤 그
결과를 UCP 응답으로 보고한 것이다.

- **디스커버리 또는 버전 실패** → 선택적 `continue_url`을 담은 전송 계층 오류
- **기능 협상 실패** → 선택적 `continue_url`을 담은 UCP 응답

##### 오류 코드 { #error-codes }

**협상 오류:**

| 코드                        | 설명                                                 | REST | MCP    |
| --------------------------- | ---------------------------------------------------- | ---- | ------ |
| `invalid_profile_url`       | 프로필 URL 형식이 잘못되었거나, 없거나, 해석할 수 없음 | 400  | -32001 |
| `profile_unreachable`       | URL은 해석했지만 가져오기 실패(타임아웃, 2xx 외 응답) | 424  | -32001 |
| `profile_malformed`         | 가져온 내용이 유효한 JSON이 아니거나 스키마를 위반함 | 422  | -32001 |
| `version_unsupported`       | 플랫폼의 프로토콜 버전을 지원하지 않음               | 422  | -32001 |
| `capabilities_incompatible` | 교집합에 호환되는 기능이 없음                        | 200  | result |

**서명 오류:**

| 코드                   | 설명                                                   | REST | MCP    |
| ---------------------- | ------------------------------------------------------ | ---- | ------ |
| `signature_missing`    | 필요한 서명 헤더/필드가 없음                           | 401  | -32000 |
| `signature_invalid`    | 서명 검증 실패                                         | 401  | -32000 |
| `key_not_found`        | 서명자가 게시한 키 세트에서 키 ID를 찾을 수 없음       | 401  | -32000 |
| `digest_mismatch`      | 본문 다이제스트가 `Content-Digest` 헤더와 일치하지 않음 | 400  | -32600 |
| `algorithm_unsupported`| 지원하지 않는 서명 알고리즘                            | 400  | -32600 |

서명 검증에 대한 자세한 내용은 [메시지 서명](../signatures.md)을 참고한다.

**프로토콜 오류:**

| HTTP | 설명                                            | MCP        |
| ---- | ----------------------------------------------- | ---------- |
| 401  | 인증이 필요하거나 자격 증명이 유효하지 않음     | -32000     |
| 403  | 인증되었지만 권한이 부족함                      | -32000     |
| 409  | 멱등성 키를 다른 페이로드로 재사용함            | -32000     |
| 429  | 요청이 너무 많음                                | -32000     |
| 500  | 예기치 않은 서버 오류                           | -32603     |
| 503  | 서버가 일시적으로 요청을 처리할 수 없음         | -32000     |

HTTP 위의 MCP에서는 HTTP 상태 코드가 1차 신호이고, JSON-RPC `error.code`는 2차
신호다. 두 전송 방식 모두 429와 503 응답에 `Retry-After` 헤더(REST) 또는
`error.data.retry_after`(MCP)를 포함해야 한다(**SHOULD**).

임베디드 프로토콜은 호스트와 임베디드 컨텍스트 간의 피어 투 피어 통신에 같은
JSON-RPC 오류 코드를 사용한다. 서버에 특화된 상황(요청 속도 제한, 멱등성)은
임베디드 전송 방식에 적용되지 않는다. 전체 오류 처리 명세는
[임베디드 프로토콜 — 응답 처리](../embedded-protocol.md#response-handling)를
참고한다.

##### `continue_url` 필드 { #the-continue_url-field }

UCP 협상이 실패하면 `continue_url`이 대체 웹 경험을 제공한다. 비즈니스는 맥락상
가장 관련성 높은 URL을 제공해야 한다(**SHOULD**).

- 체크아웃 오퍼레이션: 장바구니 또는 체크아웃 페이지로 연결
- 카탈로그 오퍼레이션: 상품 또는 검색 결과로 연결
- 대체 수단: 스토어프런트 홈페이지로 연결

이렇게 하면 우아한 성능 저하(graceful degradation)가 가능하다. 에이전트는
구매자를 표준 웹 인터페이스로 리디렉션해 작업을 마무리하게 할 수 있다.

##### 전송 바인딩 { #transport-bindings }

=== "REST"

    **디스커버리 실패(424):**

    ```http
    HTTP/1.1 424 Failed Dependency
    Content-Type: application/json

    {
      "code": "profile_unreachable",
      "content": "Unable to fetch agent profile: connection timeout",
      "continue_url": "https://merchant.com/cart"
    }
    ```

    **지원하지 않는 버전(422):**

    ```http
    HTTP/1.1 422 Unprocessable Content
    Content-Type: application/json

    {
      "code": "version_unsupported",
      "content": "Protocol version 2026-01-12 is not supported. This business supports versions 2026-01-11 and 2026-01-23.",
      "continue_url": "https://merchant.com/cart"
    }
    ```

    **호환되지 않는 기능(200):**

    ```http
    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "status": "error",
        "capabilities": {}
      },
      "messages": [
        {
          "type": "error",
          "code": "capabilities_incompatible",
          "content": "No compatible capabilities in the intersection",
          "severity": "unrecoverable"
        }
      ],
      "continue_url": "https://merchant.com"
    }
    ```

    **프로토콜 오류 — 요청 속도 제한(429):**

    ```http
    HTTP/1.1 429 Too Many Requests
    Retry-After: 60
    ```

    **프로토콜 오류 — 인증 실패(401):**

    ```http
    HTTP/1.1 401 Unauthorized
    WWW-Authenticate: Bearer realm="ucp"
    ```

    프로토콜 오류는 표준 HTTP 상태 코드와 헤더를 사용한다. 응답 본문은 선택
    사항이다.

=== "MCP"

    **디스커버리 실패(JSON-RPC error):**

    <!-- ucp:example schema=transports/jsonrpc def=error_response -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "error": {
        "code": -32001,
        "message": "UCP discovery failed",
        "data": {
          "code": "profile_unreachable",
          "content": "Unable to fetch agent profile: connection timeout",
          "continue_url": "https://merchant.com/cart"
        }
      }
    }
    ```

    **지원하지 않는 버전(JSON-RPC error):**

    <!-- ucp:example schema=transports/jsonrpc def=error_response -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "error": {
        "code": -32001,
        "message": "Protocol version not supported",
        "data": {
          "code": "version_unsupported",
          "content": "Protocol version 2026-01-12 is not supported. This business supports versions 2026-01-11 and 2026-01-23.",
          "continue_url": "https://merchant.com/cart"
        }
      }
    }
    ```

    **호환되지 않는 기능(JSON-RPC result):**

    <!-- ucp:example schema=common/types/error_response extract=$.result.structuredContent -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "structuredContent": {
          "ucp": {
            "version": "{{ ucp_version }}",
            "status": "error"
          },
          "messages": [
            {
              "type": "error",
              "code": "capabilities_incompatible",
              "content": "No compatible capabilities in the intersection",
              "severity": "unrecoverable"
            }
          ],
          "continue_url": "https://merchant.com"
        },
        "content": [
          {"type": "text", "text": "{\"ucp\":{…},…}"}
        ]
      }
    }
    ```

    **프로토콜 오류 — 요청 속도 제한(JSON-RPC error):**

    <!-- ucp:example schema=transports/jsonrpc def=error_response -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "error": {
        "code": -32000,
        "message": "Rate limit exceeded",
        "data": {
          "retry_after": 60
        }
      }
    }
    ```

    **프로토콜 오류 — 인증 실패(JSON-RPC error):**

    <!-- ucp:example schema=transports/jsonrpc def=error_response -->
    ```json
    {
      "jsonrpc": "2.0",
      "id": 1,
      "error": {
        "code": -32000,
        "message": "Unauthorized"
      }
    }
    ```

    Streamable HTTP 전송 방식을 사용할 때, 서버는 JSON-RPC 오류와 함께 그에
    대응하는 HTTP 상태 코드(예: 요청 속도 제한이면 `429`)를 반드시 반환해야
    한다(**MUST**). 오류 유형을 나타내는 1차 신호는 HTTP 상태 코드다.

#### 응답 내 기능 선언 { #capability-declaration-in-responses }

응답의 `capabilities` 레지스트리는 활성화된 기능을 나타낸다.

<!-- ucp:example schema=shopping/checkout op=read -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": {
      "dev.ucp.shopping.checkout": [
        {"version": "{{ ucp_version }}"}
      ],
      "dev.ucp.shopping.fulfillment": [
        {"version": "{{ ucp_version }}"}
      ]
    },
    "payment_handlers": {
      "com.example.processor_tokenizer": [
        {"id": "processor_tokenizer", "version": "{{ ucp_version }}", "available_instruments": [{"type": "card"}]}
      ]
    }
  },
  "id": "checkout_123",
  "status": "incomplete",
  "currency": "USD",
  "line_items": [ ... ],
  "totals": [ ... ],
  "links": [ ... ]
}
```

#### 응답 기능 선택 { #response-capability-selection }

비즈니스는 `ucp.capabilities`에 다음 조건을 모두 충족하는 기능만 반드시 포함해야
한다(**MUST**).

1. 이 세션에서 협상된 교집합에 속하고, 그리고(AND)
2. 이 응답의 오퍼레이션 유형과 관련이 있다

**루트 기능의 관련성:**

루트 기능은 오퍼레이션 유형과 일치할 때 관련이 있다.

- `create_checkout` / `update_checkout` / `complete_checkout` →
    `dev.ucp.shopping.checkout`
- `create_cart` / `update_cart` → `dev.ucp.shopping.cart`
- 주문 웹훅 → `dev.ucp.shopping.order`

**확장의 관련성:**

확장은 `extends` 값 중 **하나라도** 관련 있는 루트 기능과 일치하면 관련이 있다.

**선택 예시:**

| 응답 유형     | 포함                            | 포함하지 않음                |
| ------------- | ------------------------------- | ---------------------------- |
| 체크아웃      | checkout, discount, fulfillment | cart, order                  |
| 장바구니      | cart, discount                  | checkout, fulfillment, order |
| 주문          | order                           | checkout, cart, discount     |

## 신원 확인과 인증 { #identity-authentication }

UCP 프로필은 두 가지 용도로 쓰인다. 협상을 위해 당사자의 **기능**을
선언하고([프로필 구조](#profile-structure) 참고), 신원 검증을 위한 **서명 키**를
게시한다. 따라서 문서 하나로 기능 협상과 암호학적 인증을 모두 할 수 있다.

비즈니스는 디스커버리 진입점으로 `/.well-known/ucp`에 프로필을 게시한다.
플랫폼은 이를 가져와 프로토콜 지원 여부를 확인하고, 엔드포인트를 찾고, 기능을
협상한다. 플랫폼은 요청마다 `UCP-Agent` 헤더로 자신의 프로필 URL을 광고하며,
비즈니스는 이를 통해 기능을 협상하고 신원을 검증한다. 이 설계 덕분에 **허가 없는
온보딩**(permissionless onboarding)이 가능하다. 디스커버리 가능한 프로필을 가진
플랫폼이라면 사전 등록 없이 어떤 비즈니스와도 상호작용할 수 있다.

**Web Bot Auth 상호 운용.** WBA 형태의 서명을 택한 서명자는 자신의 키를 광고하는
`Signature-Agent` 헤더를 추가로 내보낸다. 검증자가 신원을 해석하는 방법은
[신원 해석 알고리즘](#identity-resolution-algorithm)을, 서명 형태는
[메시지 서명 — WBA 상호 운용](../signatures.md#wba-interop)을 참고한다.

### 인증 메커니즘 { #authentication-mechanisms }

비즈니스는 사칭을 막고 메시지 무결성을 보장하기 위해 플랫폼을 인증해야
한다(**SHOULD**). UCP는 여러 인증 메커니즘과 호환된다.

- **API 키** — 대역 외(out-of-band)로 교환한 사전 공유 비밀
- **OAuth 2.0** — 클라이언트 자격 증명 또는 그 밖의 OAuth 흐름
- **mTLS** — 클라이언트 인증서를 사용하는 상호 TLS
- **HTTP Message Signatures** — [RFC 9421](https://www.rfc-editor.org/rfc/rfc9421)에
  따른 암호학적 서명(전체 명세는 [메시지 서명](../signatures.md) 참고)

HTTP Message Signatures를 쓰면 허가 없는 온보딩이 가능하다. 비즈니스는 공유
비밀을 협상하지 않고도 플랫폼이 광고한 공개 키로 플랫폼을 검증할 수 있다. 나머지
메커니즘은 사전에 자격 증명을 교환해야 하므로 미리 맺어진 관계를 전제로 한다.

비즈니스에서 플랫폼으로 보내는 웹훅은 반드시 서명해야 한다(**MUST**).
[메시지 서명 — 서명 적용 대상](../signatures.md#when-signatures-apply)을 참고한다.

#### 신원 바인딩 { #identity-binding }

인증 메커니즘과 관계없이, 검증자는 인증된 신원이 `UCP-Agent` 헤더와 일치하는지
반드시 확인해야 한다(**MUST**).

- **HTTP Message Signatures** — 서명자의 프로필(`UCP-Agent`에서 얻음)은 서명
    검증으로 확인되므로 추가 검사가 필요 없다.
- **API 키 / OAuth / mTLS** — 검증자는 인증된 주체가 `UCP-Agent`로 식별되는
    프로필을 대신해 행동할 권한이 있는지 반드시 확인해야 한다(**MUST**). 인증된
    신원과 주장된 프로필이 충돌하는 요청은 거부한다.

### 키 디스커버리 { #key-discovery }

양측 모두 UCP 프로필에 공개 키를 게시한다. 플랫폼은 `/.well-known/ucp`에서
비즈니스 프로필을 가져오고, 비즈니스는 `UCP-Agent` 헤더(Web Bot Auth 상호 운용을
사용하는 경우 `Signature-Agent` 헤더)에서 플랫폼 프로필을 가져온다. 기능을
제공하는 바로 그 프로필이 검증 키도 제공한다. 이것이
[RFC 9421](https://www.rfc-editor.org/rfc/rfc9421) HTTP Message Signatures를 위한
UCP의 키 해석 메커니즘이다.

게시 규약(정식 최상위 `keys[]` JWK Set)은 [프로필 구조](#profile-structure)를
참고한다. 두 해석 경로 모두 같은 목록을 읽는다.

- **`UCP-Agent`로 해석**(기본 UCP 키 조회) — `keys[]`를 읽는다.
- **`Signature-Agent`로 해석**(Web Bot Auth, 선택 사항) — `keys[]`를 읽는다.
  `cimd`/`directory` 변형은 각자의 문서를 거쳐 JWK Set에 도달한다.

기능 기반 키 해석, 프로필 가져오기, 커버 컴포넌트 강제를 포함한 전체 검증자
알고리즘은 아래의 [신원 해석 알고리즘](#identity-resolution-algorithm)을 참고한다.
키 형식(JWK), 지원 알고리즘, 키 교체 절차, Web Bot Auth 상호 운용 서명 형태는
[메시지 서명](../signatures.md)을 참고한다.

### 프로필 요구 사항 { #profile-requirements }

#### 호스팅 { #hosting }

프로필과, 프로필이 참조하는 스키마 및 전송 방식 기술 아티팩트는 안정적으로
호스팅되어야 한다. 엔드포인트가 불안정하거나 잘못 구성되면 상대방이 요청을
처리하지 못할 수 있다.

1. 게시된 아티팩트는 반드시 HTTPS로 제공해야 한다(**MUST**).
2. 프로필 엔드포인트는 다른 URL로 리디렉션해서는 안 된다(**MUST NOT**)(`301`,
   `302`, `303`, `307`, `308`). 조건부 요청에 `304 Not Modified`로 응답하는 것은
   리디렉션이 아니므로 허용된다.
3. 게시된 아티팩트는 반드시 `public`과 60초 이상의 `max-age`를 담은
   `Cache-Control` 헤더를 포함해야 하며(**MUST**), `private`, `no-store`,
   `no-cache` 지시어와 함께 제공해서는 안 된다(**MUST NOT**).
4. 게시된 아티팩트는 소비자가 캐시된 사본을 효율적으로 재검증할 수 있도록 검증자
   (`ETag` 또는 `Last-Modified`)를 포함해야 한다(**SHOULD**).

프로필은 당사자의 안정적인 신원과 기능을 나타낸다. 프로필 URL은 요청이 바뀌어도
일관되게 유지되어야 하며, 트랜잭션별 또는 세션별 설정을 담아서는 안 된다. 위의
캐싱 정책은 최소 TTL을 둔 공유 캐시 지원을 요구함으로써 이를 강제한다.

#### 가져오기 { #fetching }

비즈니스는 기능 협상과 신원 검증을 위해 플랫폼 프로필을 가져온다. UCP는 허가
없는 온보딩을 가능하게 하는 모범 사례를 정의하지만, 접근 정책에 대한 통제권은
전적으로 비즈니스에 있다. 비즈니스는 확립된 신뢰, 관찰된 행동, 운영 요구 사항에
따라 추가 규칙을 강제할 수 있다(**MAY**).

비즈니스는 사전 승인된 플랫폼의 레지스트리를 유지해야 한다(**SHOULD**). 사전
승인된 플랫폼이란 프로필이 검증되었고 대역 외 메커니즘(API 키, OAuth 자격 증명,
mTLS 인증서, 사전 심사)으로 신뢰가 확립된 플랫폼을 말한다. 알려진 플랫폼은 캐시된
신원과 기능을 바탕으로 효율적으로 처리할 수 있으며, 디스커버리 예산 제약을 받지
않는다.

플랫폼을 *인식하지 못하면* 동적 프로필 디스커버리가 시작된다. 비즈니스는 고정된
디스커버리 사용량을 정해 두어, 접근을 요청하는 플랫폼이 아무리 많아도 인식하지
못한 플랫폼을 해석하는 데 드는 자원 소비가 일정하게 유지되도록 해야
한다(**SHOULD**). 전략은 다음과 같다.

- **고정 크기 프로필 캐시**(예: LRU) — 마주치는 고유 프로필 URL의 수와
  관계없이 메모리 사용량에 상한을 둔다
- 디스커버리 가져오기에 대한 **전역 속도 제한** — 오리진별 상태를 추적하지 않고도
  외부로 나가는 네트워크 사용량에 상한을 둔다
- **반복 실패 시 백오프** — 계속 사용할 수 없거나 악의적인 프로필 엔드포인트로의
  재시도를 줄인다
- **비동기 디스커버리** — `503` 상태 코드와 `Retry-After` 헤더로 응답해 프로필
  해석을 미루고, 백그라운드에서 프로필을 해석한다. 플랫폼이 재시도할 때는 검증된
  프로필이 캐시되어 있으므로 기능 협상이 동기적으로 진행된다

다음 규칙은 신원 해석 중 역참조하는 모든 URL, 즉 프로필과 검증자가 따라가는 모든
`jwks_uri` 또는 CIMD 문서에 적용된다.

1. 구현체는 HTTPS로 제공되지 않는 URL을 반드시 거부해야 한다(**MUST**).
2. 구현체는 다른 URL로의 리디렉션(`301`, `302`, `303`, `307`, `308`)을 따라가서는
   안 된다(**MUST NOT**). 조건부 요청에 대한 응답으로 받은 `304 Not Modified`는
   리디렉션이 아니며, 구현체는 캐시된 사본을 계속 사용하면 된다.
3. 구현체는 연결 타임아웃과 응답 타임아웃을 강제해야 한다(**SHOULD**).
4. 구현체는 오리진의 `Cache-Control` 헤더와 관계없이 최소 60초의 TTL 하한을 두고
   프로필을 캐시해야 한다(**SHOULD**).
5. 구현체는 stale-while-revalidate 방식으로 프로필을 비동기적으로 갱신할 수
   있다(**MAY**).
6. 알 수 없는 `kid` 때문에 서명 검증에 실패하면, 구현체는 캐시된 프로필을 한 번
   강제로 갱신해야 한다(**SHOULD**). 단, 오리진별로 TTL 하한 기간마다 두 번 이상
   그렇게 해서는 안 된다(**MUST NOT**).
7. 구현체는 특수 용도 IP 주소([RFC 6890](https://www.rfc-editor.org/rfc/rfc6890)
   — 루프백, 클라우드 메타데이터 주소 `169.254.169.254`를 포함한 링크 로컬,
   사설 대역, 그 밖의 예약 대역)로 해석되는 URL을 반드시 거부해야 한다(**MUST**).
   단, 검증자 자신이 같은 루프백 인터페이스에서 실행되는 경우(로컬 개발)의 루프백
   대상은 예외다. 검증자는 DNS 리바인딩에 대비해 호스트 이름만이 아니라 해석된
   주소를 검증해야 하며(**SHOULD**), 가져온 문서 안에 담긴 URL(예: CIMD의
   `jwks_uri`)이 그런 주소로 해석되면 역참조하지 않아야 한다(**SHOULD NOT**).
8. 구현체는 응답이 무한정 커져 자원이 고갈되지 않도록 응답 본문 크기에 상한을 두어야
   한다(**SHOULD**). UCP 프로필은 데이터 페이로드가 아니라 신원/기능 매니페스트다
   (문서화된 프로필은 5 KiB 미만이다). 스키마에는 크기 제한이 없으므로 이 상한은
   배포 환경의 보호 장치이며, 검증자는 규격을 준수하는 프로필을 거부하지 않도록 이
   값을 128 KiB 이상으로 설정해야 한다(**SHOULD**).

프로필을 가져올 수 없거나(타임아웃, DNS 실패, 5xx) 검증에 실패하면(유효하지 않은
스키마, 서명 키, 서명 불일치), 비즈니스는 적절한 오류와 상태 코드로 요청을 반드시
거부해야 한다(**MUST**)([오류 처리](#error-handling) 참고).

### WBA 상호 운용을 위한 배포 패턴 { #deployment-patterns-for-wba-interop }

최상위 `keys[]` 배열을 담은 UCP 프로필은 유효한 RFC 7517 JWK Set이며, 서명자는
원한다면 이를 Web Bot Auth 키 소스로 재사용할 수 있다. `Signature-Agent` 헤더의
`type` 파라미터는 검증자가 광고된 키를 어떻게 해석할지 선택한다. 이 파라미터와
`jwks_uri`/`cimd`/`directory` 값은
[draft-meunier-webbotauth-httpsig-directory-00](https://datatracker.ietf.org/doc/draft-meunier-webbotauth-httpsig-directory/00/)의
§4.1에 정의되어 있다. 각 변형은 단독으로 쓸 수도 있고 UCP 프로필을 다시 가리킬
수도 있다.

- **`type=jwks_uri`** — 멤버 값은 JWK Set URL이며 직접 가져온다. 이 값이 UCP
  프로필 URL을 가리키게 하면 프로필의 `keys[]`가 JWK Set 역할을 한다. 즉 문서
  하나가 프로필이자 키 소스가 된다. 무결성은 프로필 오리진에 대한 TLS에서 나오며,
  키별 자체 서명은 없다. `type=jwks_uri`는 명시적으로 설정해야 한다. `type`을
  생략하면 기본값이 `directory`(아래 참고)가 되는데, 이 방식은 정적 프로필이 아니라
  서명된 디렉터리를 기대한다.

  ```text
  Signature-Agent: sig1="https://platform.example/.well-known/ucp";type=jwks_uri
  ```

- **`type=cimd`** — 멤버 값은 Client ID Metadata Document
  ([draft-ietf-oauth-client-id-metadata-document](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/))이며,
  그 문서의 `jwks_uri`가 UCP 프로필을 다시 가리킬 수 있다(**MAY**). 상대방이
  CIMD 형태의 에이전트 신원을 사용할 때 쓴다.

  ```text
  Signature-Agent: sig1="https://platform.example/agent";type=cimd
  ```

- **`type=directory`** *(기본값)* — `type`을 생략하거나 `directory`로 설정하면,
  멤버 값은 전체 URL이 아니라 **오리진**이다. 검증자는 그 오리진에 등록된 well-known
  경로(`/.well-known/http-message-signatures-directory`)를 붙여 그곳에서 서명된
  디렉터리를 가져온다. 그 형식, 키별 자체 서명, 미디어 타입은 디렉터리 초안의
  §5.2에 정의되어 있다.

  ```text
  Signature-Agent: sig1="https://platform.example"  # type omitted -> directory
  Signature-Agent: sig1="https://platform.example";type=directory  # explicit
  ```

### 신원 해석 알고리즘 { #identity-resolution-algorithm }

UCP와 Web Bot Auth는 두 가지 키 해석 메커니즘을 정의한다. 검증자가 어느 쪽을
쓸지는 서명의 `tag`가 아니라 **검증자의 역량과 존재하는 헤더**로 정해진다.
`tag`는 힌트일 뿐 관문이 아니다. 기본 UCP 키 조회(`UCP-Agent`)는 모든 UCP
검증자가 지원하며 어떤 UCP 서명에도 동작한다. Web Bot Auth 키 조회
(`Signature-Agent`)는 선택적인 추가 계층이다.

요청은 [RFC 9421 §4.3](https://www.rfc-editor.org/rfc/rfc9421#section-4.3)에 따라
여러 서명을 담을 수 있다(MAY). 검증자는 각 서명을 독립적으로 검증하며, 적어도
하나의 서명이 검증되면 요청이 인증된다. 아래 알고리즘은 서명 하나를 처리한다.

1. **서명 키를 해석한다.** 검증자는 자신이 지원하면서 해당 헤더가 존재하는 해석
   메커니즘을 사용한다.
    - **`UCP-Agent` — 기본 UCP 키 조회로, 모든 UCP 검증자가 지원한다.**
      `UCP-Agent` 프로필 URL을 해석하고 `keys[]`를 읽는다. 이 경로는 태그가 없는
      UCP 서명(기본 UCP)이나 `tag="web-bot-auth"`를 담은 UCP 서명(이중 대상
      형태)에 적용된다. 검증자는 이런 서명을 `UCP-Agent`로 해석하고,
      `signature-agent`를 평범한 커버 컴포넌트로 취급하며, Web Bot Auth 키
      디스커버리를 구현할 필요가 없다. (다만 이중 대상 서명을 검증하려면, 서명자가
      어떤 알고리즘을 썼든 [서명 알고리즘](../signatures.md#signature-algorithms)에
      따라 그 키의 알고리즘을 지원해야 하고, `signature-agent;key="<label>"`을
      커버하기 위한 RFC 9421 §2.1.2 Dictionary 멤버 컴포넌트 선택을 지원해야
      한다.) `web-bot-auth` 이외의 태그를 가진 서명은 UCP가 그 태그를 정의하거나
      명시적으로 허용하지 않는 한 건너뛴다.
    - **`Signature-Agent` — Web Bot Auth 키 조회로, 선택 사항(OPTIONAL)이다
      (WBA를 인식하는 검증자).** `tag="web-bot-auth"`를 담은 서명에 대해, WBA를
      인식하는 검증자는 대신
      [Signature-Agent 파싱 규칙](../signatures.md#rest-request-verification)에
      따라 파싱한 `Signature-Agent` 멤버로 해석할 수 있다(**MAY**). 그런 서명은
      반드시
      [draft-meunier-webbotauth-httpsig-protocol-00](https://datatracker.ietf.org/doc/draft-meunier-webbotauth-httpsig-protocol/00/)
      §4.2의 WBA 에이전트 서명 요구 사항을 충족해야 한다(**MUST**)(서명자 측
      형태는 [WBA 상호 운용](../signatures.md#wba-interop) 참고). 멤버 값은
      반드시 HTTPS URL이어야 한다(**MUST**). 그 `type`이 해석 방식을 선택한다.
      `jwks_uri`와 `cimd`는 서명자의 프로필을 거쳐 키에 도달하며 아래 단계로
      해석된다. `directory` 메커니즘은 디렉터리 초안이 정의한다
      ([배포 패턴](#deployment-patterns-for-wba-interop) 참고).
      `data:` URI 인라인 형식은 UCP-WBA 상호 운용 범위에 포함되지 않는다.
   검증자가 지원하는 어떤 메커니즘으로도 키를 해석할 수 없으면 이 서명을
   **건너뛴다**. 필요한 헤더가 없거나, 서명 레이블과 일치하는 `Signature-Agent`
   멤버가 없거나, URL이 HTTPS가 아니거나, `tag`가 이 검증자가 다루지 않는 용도로
   한정된 경우가 그렇다.
2. **문서를 가져온다.** [§가져오기](#fetching)에 따른다. 가져오기에
   실패하면(DNS 오류, 네트워크 장애, 2xx 외 응답, 파싱 실패) 이 서명을
   **건너뛴다**.
3. **키 목록을 찾는다.** `UCP-Agent`로 해석했거나 `Signature-Agent`
   `type=jwks_uri`로 해석했다면 프로필의 최상위 `keys[]`(RFC 7517 JWK Set)를
   사용한다. `type=cimd`라면 문서의 `jwks_uri`를 역참조해 JWK Set을 얻는다.
   무결성은 해석된 오리진에 대한 TLS에서 나온다.
4. **서명의 `keyid`를** 해석된 키 목록의 `kid`와 **매칭한다.** 해석할 때
   **서명 검증에 쓸 수 없는 키는 건너뛴다**. `use:"enc"`로 표시된 키, 또는
   `key_ops`가 있지만 `"verify"`를 포함하지 않는 키가 여기에 해당한다
   ([RFC 7517](https://www.rfc-editor.org/rfc/rfc7517) §4.2, §4.3). `use:"sig"`를
   설정했거나 두 멤버를 모두 생략한 키는 적격으로 남는다. 일치하는 적격 키가 없으면
   이 서명을 **건너뛴다**. WBA 형태의 서명(`tag="web-bot-auth"`)이라면 검증자는
   `keyid`가 매칭된 JWK의 [RFC 7638](https://www.rfc-editor.org/rfc/rfc7638)
   SHA-256 thumbprint와 같은지도 반드시 확인해야 한다(**MUST**). WBA 아키텍처
   초안 §4.2가 이를 요구하며, 이로써 광고된 키 식별자가 키 바이트에 묶인다. 확인에
   실패하면 이 서명을 건너뛴다.
5. **모든 방식에서 커버 컴포넌트 요구 사항을 강제한다.** `tag`와 전송 방식에
   관계없이, 서명은 반드시 요청 대상(`@method`, `@authority`, `@path`, 쿼리
   문자열이 있으면 `@query`), 본문이 있으면 본문(`content-digest`,
   `content-type`), 그리고 다음 요청 헤더가 있으면 각 헤더를 커버해야
   한다(**MUST**): `ucp-agent`, `signature-agent`, `idempotency-key`,
   `webhook-id`, `webhook-timestamp`(닫힌 집합이다. 나중에 UCP에 추가되는 헤더는
   그 헤더를 정의하는 섹션이 그렇게 명시할 때만 필수가 된다). 이런 컴포넌트 중
   하나라도 서명의 커버 집합에 없으면 이 서명을 **건너뛴다**. 서명이 커버하지 않는
   대상, 본문, 헤더는 서명되지 않은 것으로 취급한다. 이렇게 하면 Web Bot Auth의
   최소 커버 집합(`@authority`, `signature-agent`)만 충족하는 서명으로 본문,
   메서드, 경로가 묶이지 않은 UCP 요청을 인증하는 일을 막을 수 있다. (요청이
   `Idempotency-Key`를 반드시 담아야 하는지는 이 커버리지 검사와 별개인 바인딩
   수준의 규칙이다.)
6. 매칭된 키로 **서명을 검증한다.** 서명 알고리즘은 JWK의 `kty`/`crv`에서
   도출한다. 검증자가 매칭된 키의 `kty`, `crv`, `alg`를 지원하지 않으면 이 서명을
   **건너뛴다**. 다른 서명이 요청을 인증하지 못하면 결과는
   `algorithm_unsupported`가 된다.

요청은 모든 서명이 건너뛰어졌거나 검증에 실패한 경우에만 반드시
거부해야 한다(**MUST**)(`key_not_found`, `algorithm_unsupported` 또는 관련 오류).

**인증된 신원.** 서명이 검증되면, 인증된 서명자는 검증에 쓰인 키를 제공한 URL로
식별된다. WBA 형태의 서명이라면 `Signature-Agent` URL, 기본 UCP 서명이라면
`UCP-Agent` URL이다. 두 URL이 같은 요청에 함께 있을 수 있다. 요청에 붙는 신원은
어떤 헤더를 보냈는지가 아니라 어떤 서명이 검증되었는지로 결정된다. 여러 서명이
검증되면, 각 서명은 자신의 키를 제공한 URL로만 서명자를 식별한다.
`Signature-Agent`로 해석한 키는 그 키 소스에 대한 통제권을 증명할 뿐, `UCP-Agent`
프로필(그 URL은 서명된 헤더 값에 불과하다)에 대한 통제권을 증명하지는 않는다.
구현체는 그 URL들이 정규화 후 같을 때에만 요청을 하나의 인증된 신원으로 취급해야
한다(**MUST**). 그렇지 않으면 서로 다른 신원이며, 어느 쪽이든 충분한지는 정책이
결정한다.

이 규칙은 **HTTP 전송 계층 신원**에 적용된다. 페이로드 계층의 주장(예: 요청
본문에 담긴 AP2 위임 JWT)에는 별도의 신원 바인딩과 키 해석 규칙이 있다.
[AP2 위임](../payment/extensions/ap2-mandates.md)을 참고한다.

## 결제 아키텍처 { #payment-architecture }

UCP는 결제에 분리형 아키텍처를 채택해 **플랫폼**, **비즈니스**, **결제 자격 증명
제공자** 사이의 "N 대 N" 복잡성 문제를 해결한다. 이 설계는 **결제 수단**(무엇을
받는가)과 **결제 핸들러**(결제 수단을 어떻게 처리하는지에 대한 명세)를 분리해
보안과 확장성을 보장한다.

### 보안 및 신뢰 모델 { #security-and-trust-model }

결제 아키텍처는 "설계에 의한 신뢰(Trust-by-Design)" 철학 위에 세워졌다. 비즈니스와
결제 자격 증명 제공자는 신뢰할 수 있는 법적 관계를 맺고 있지만, 플랫폼(클라이언트)은
중개자로서 원본 금융 자격 증명을 다루지 않아야 한다(**SHOULD NOT**)고 가정한다.

#### 신뢰 삼각형 { #the-trust-triangle }

1. **비즈니스 ↔ 결제 자격 증명 제공자:** 기존에 맺어진 법적·기술적 관계다. 비즈니스는 결제 자격 증명 제공자와의 계약과 API 키를 보유한다.
2. **플랫폼 ↔ 결제 자격 증명 제공자:** 플랫폼은 데이터를 토큰화하기 위해 결제 자격 증명 제공자의 인터페이스(예: iframe 또는 API)와 상호작용하지만, 자금의 "소유자"는 아니다.
3. **플랫폼 ↔ 비즈니스:** 플랫폼은 주문을 확정할 수 있도록 결과(토큰 또는 위임)를 비즈니스에 전달한다.

#### 자율 커머스를 위한 강화된 보안 { #enhanced-security-for-autonomous-commerce }

사용자 승인에 대한 암호학적 증명이 필요한 시나리오(예: 자율 AI 에이전트)를 위해
UCP는 **AP2 위임 확장**(`dev.ucp.common.payment.ap2_mandate`)을 지원한다. 이 선택적
확장은 검증 가능한 디지털 자격 증명으로 부인 방지가 가능한 승인을 제공한다.

이 확장을 언제, 어떻게 쓰는지에 대한 자세한 내용은
[트랜잭션 무결성](#transaction-integrity-and-non-repudiation)과
[AP2 위임 확장](../payment/extensions/ap2-mandates.md)을 참고한다.

#### 자격 증명 흐름과 PCI 범위 { #credential-flow-pci-scope }

규정 준수(PCI-DSS) 부담을 최소화하기 위해:

1. **단방향 흐름:** 자격 증명은 **플랫폼 → 비즈니스** 방향으로만 흐른다. 비즈니스는 응답에 자격 증명을 되돌려 보내서는 안 된다(**MUST NOT**).
2. **불투명한 자격 증명:** 플랫폼은 원본 PAN이 아니라 토큰(네트워크 토큰 등), 암호화된 페이로드, 위임을 다룬다.
3. **핸들러 ID 라우팅:** 페이로드의 `handler_id` 덕분에 비즈니스는 복호화/청구에 어떤 결제 자격 증명 제공자 키를 써야 하는지 정확히 알 수 있으며, 이로써 키 혼동 공격을 막는다.

### 역할과 책임: 누가 무엇을 구현하는가? { #roles-responsibilities-who-implements-what }

흔히 혼동하는 부분이 역할 분담이다. UCP 결제 모델은 책임을 다음과 같이 나눈다.

| 역할                            | 책임                       | 행동                                                                                                                                                                                                                                                                |
| :------------------------------ | :------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **결제 자격 증명 제공자**       | **명세를 정의한다**        | **핸들러 정의**를 만든다. 카드를 어떻게 토큰화하고 어떤 설정 입력이 필요한지 규정하는 "청사진"(JSON 스키마)을 게시한다.<br>*예: "'com.psp-x.tokenization' 핸들러의 스키마는 이것입니다."*                                                                                |
| **비즈니스**                    | **핸들러를 설정한다**      | 사용할 핸들러를 고르고, UCP 체크아웃 응답에 자신만의 **설정**(공개 키, 판매자 ID)을 제공한다. *예: "이 Publishable Key와 함께 'com.psp-x.tokenization'을 사용해 Visa를 받습니다."*                                                                                     |
| **플랫폼**                      | **프로토콜을 실행한다**    | 비즈니스의 설정을 읽고, 결제 자격 증명 제공자의 명세가 정의한 로직을 실행해 토큰을 얻는다. *예: "이 비즈니스는 결제 자격 증명 제공자를 사용하는군요. 비즈니스의 키로 제공자의 SDK를 호출해 토큰을 받겠습니다."*                                                         |

### 체크아웃 수명 주기에서의 결제 { #payment-in-the-checkout-lifecycle }

결제가 필요하면, 결제 과정은 UCP 안에서 표준 3단계 수명 주기를 따른다. 바로
**협상**, **획득**, **완료**다.

![상위 수준 결제 흐름 시퀀스 다이어그램](site:specification/images/ucp-payment-flow.png)

1. **협상(비즈니스 → 플랫폼):** 비즈니스는 UCP 프로필에 사용 가능한 결제 핸들러를 광고한다. 이로써 플랫폼에 *어떻게* 결제할지 알려준다(예: "이 공개 키로 이 특정 결제 자격 증명 제공자 엔드포인트를 사용하라").
2. **획득(플랫폼 ↔ 결제 자격 증명 제공자):** 플랫폼이 핸들러의 로직을 실행한다. 이 과정은 클라이언트 측 또는 에이전트 측에서 결제 자격 증명 제공자와 직접 이루어진다(예: 자격 증명을 네트워크 토큰으로 교환). 비즈니스는 관여하지 않으므로, 원본 데이터가 비즈니스의 프런트엔드 API에 닿는 일이 없다.
3. **완료(플랫폼 → 비즈니스):** 플랫폼은 불투명한 자격 증명(토큰)을 비즈니스에 제출한다. 비즈니스는 결제 자격 증명 제공자와의 백엔드 연동을 통해 이를 사용해 대금을 수납한다.

### 결제 핸들러 { #payment-handlers }

결제 핸들러는 결제 수단이 어떻게 처리되는지를 정의하는 (엔티티가 아닌)
**명세**다. 결제 핸들러는 세 참여자를 하나로 묶는 계약이다.

**중요한 구분:**

- **결제 자격 증명 제공자** = 참여자(Google Pay, Shop Pay 같은 엔티티)
- **결제 핸들러** = 제공자가 작성하는 명세(예: `com.google.pay`, `dev.shopify.shop_pay`)

결제 핸들러를 사용하면 네트워크 토큰을 포함해 다양한 결제 수단과 토큰 유형을
지원할 수 있다. 결제 핸들러는 보통 결제 자격 증명 제공자나 UCP 운영 기구가
작성하는 표준화된 정의다.

**동적 필터링:** 비즈니스는 장바구니의 맥락에 따라 `handlers` 목록을 반드시
필터링해야 한다(**MUST**)(예: 구독 상품에는 "선구매 후결제(Buy Now Pay Later)"를
제거하거나, 배송 주소에 따라 지역별 결제 방식을 필터링한다).

**표시 순서:** 비즈니스는 프로필과 응답 봉투의 `map_order.payment_handlers`를
통해 광고하는 핸들러의 선호 표시 순서를 선언할 수 있다(**MAY**)
([`ucp` 프로토콜 네임스페이스](#the-ucp-protocol-namespace) 참고). 이 선호는
권고적이다. 비즈니스가 선호하는 표시 방식(보통 전환율이나 위험도에 대한 판단)을
전달할 뿐이며, 플랫폼은 이를 고려해야 하지만(**SHOULD**) 자체 순서를 적용할 수도
있다(**MAY**). 핸들러 안에서도 같은 권고적 선호가 적용된다. 비즈니스가 광고한
`available_instruments` 배열의 순서는 앞에 올수록 먼저 보여 주기를 선호하는 결제
수단 표시 순서를 전달한다. 이는 플랫폼이 `context.payment[]`로 제출하는 구매자 측
선호(요청 측의 구매자 선호 핸들러)와는 별개이며 그것을 덮어쓰지 않는다. 둘
사이의 조정은 플랫폼이 한다.

**사용 가능한 결제 수단 해석:** 각 활성 핸들러 안에서 플랫폼과 비즈니스는 각자
`available_instruments`, 즉 자신이 지원하는 결제 수단 유형과 제약의 집합을
독립적으로 광고한다. 이를 체크아웃 응답에서 권위 있는 값으로 해석하는 책임은
비즈니스에 있다. 플랫폼의 선언(프로필에서 가져온 것)은 플랫폼이 무엇을 처리할 수
있는지 알린다. 비즈니스는 이를 자신의 `business_schema` 선언 및 장바구니 맥락과
교차시킨 뒤 해석된 결과를 반환한다. 플랫폼은 응답의 `available_instruments`를 해당
체크아웃에 대한 권위 있는 값으로 반드시 취급해야 한다(**MUST**). 전체 해석
의미론은 [결제 핸들러 가이드](../payment/guide.md#resolving-available_instruments)를
참고한다.

**결제 수단 개수:** `dev.ucp.common.payment.split_payments` 기능이 활성화되지 않은
한, 체크아웃 제출에는 반드시 정확히 하나의 결제 수단이 담겨야 한다(**MUST**).
비즈니스는 이 제약을 위반한 제출을 `messages[]`의 `payment_failed` 오류로 반드시
거부해야 한다(**MUST**). 이 제약을 완화하는 확장은
[분할 결제](../payment/extensions/split-payments.md)를 참고한다.

### 구현 시나리오 { #implementation-scenarios }

다음 시나리오는 구체적인 데이터 예시를 통해 다양한 결제 핸들러와 결제 수단이 어떻게
협상되고 실행되는지 보여 준다.

#### 시나리오 A: 디지털 지갑 { #scenario-a-digital-wallet }

이 시나리오에서 플랫폼은 결제 자격 증명 제공자(예: `com.google.pay`,
`dev.shopify.shop_pay`)를 식별하고 그 API를 사용해 암호화된 결제 토큰을 획득한다.

##### 1. 비즈니스 광고(체크아웃 생성 응답) { #1-business-advertisement-response-from-create-checkout }

<!-- ucp:example schema=shopping/checkout target=$.ucp -->
```json
{
  "version": "{{ ucp_version }}",
  "payment_handlers": {
      "com.google.pay": [
        {
          "id": "8c9202bd-63cc-4241-8d24-d57ce69ea31c",
          "version": "{{ ucp_version }}",
          "config": {
            "api_version": 2,
            "api_version_minor": 0,
            "environment": "TEST",
            "merchant_info": {
              "merchant_name": "Example Merchant",
              "merchant_id": "01234567890123456789",
              "merchant_origin": "checkout.merchant.com"
            },
            "allowed_payment_methods": [
              {
                "type": "CARD",
                "parameters": {
                  "allowed_auth_methods": ["PAN_ONLY"],
                  "allowed_card_networks": ["VISA", "MASTERCARD"]
                },
                "tokenization_specification": {
                  "type": "PAYMENT_GATEWAY",
                  "parameters": {
                    "gateway": "example",
                    "gatewayMerchantId": "exampleGatewayMerchantId"
                  }
                }
              }
            ]
          }
        }
      ],
      "dev.shopify.shop_pay": [
        {
          "id": "shop_pay_1234",
          "version": "{{ ucp_version }}",
          "available_instruments": [
            {"type": "shop_pay"}
          ],
          "config": {
            "shop_id": "shopify-559128571",
            "environment": "production"
          }
        }
      ]
    }
}
```

##### 2. 토큰 실행(플랫폼 측) { #2-token-execution-platform-side }

플랫폼은 `com.google.pay` 또는 `dev.shopify.shop_pay`를 인식한다. 플랫폼은 `config`를
해당 핸들러 API에 전달하고, 핸들러는 암호화된 토큰 데이터를 반환한다.

##### 3. 체크아웃 완료(비즈니스로 보내는 요청) { #3-complete-checkout-request-to-business }

플랫폼은 결제 핸들러 응답을 결제 수단으로 감싼다.

<!-- ucp:example schema=shopping/checkout op=complete direction=request -->
```json
POST /checkout-sessions/{id}/complete

{
  "payment": {
    "instruments": [
      {
        "id": "pm_1234567890abc",
        "handler_id": "8c9202bd-63cc-4241-8d24-d57ce69ea31c",
        "type": "card",
        "selected": true,
        "display": {
          "brand": "visa",
          "last_digits": "4242"
        },
        "billing_address": {
          "street_address": "123 Main Street",
          "extended_address": "Suite 400",
          "address_locality": "Charleston",
          "address_region": "SC",
          "postal_code": "29401",
          "address_country": "US",
          "first_name": "Jane",
          "last_name": "Smith"
        },
        "credential": {
          "type": "PAYMENT_GATEWAY",
          "token": "{\"signature\":\"...\",\"protocolVersion\":\"ECv2\"...}"
        }
      }
    ]
  },
  "signals": {
    "dev.ucp.buyer_ip": "203.0.113.42",
    "dev.ucp.user_agent": "Mozilla/5.0 ..."
  }
}
```

#### 시나리오 B: 챌린지가 있는 직접 토큰화(SCA) { #scenario-b-direct-tokenization-with-challenge-sca }

이 시나리오에서 플랫폼은 범용 토크나이저를 사용해 세션 토큰이나 네트워크 토큰을
요청한다. 은행이 강력한 고객 인증(SCA/3DS)을 요구하므로, 비즈니스는 완료 처리를
멈추고 챌린지를 요청해야 한다.

##### 1. 비즈니스 광고 { #1-business-advertisement }

<!-- ucp:example schema=shopping/checkout target=$.ucp -->
```json
{
  "version": "{{ ucp_version }}",
  "payment_handlers": {
    "com.example.tokenizer": [
        {
          "id": "merchant_tokenizer",
          "version": "{{ ucp_version }}",
          "spec": "https://example.com/specs/tokenizer",
          "schema": "https://example.com/schemas/tokenizer.json",
          "available_instruments": [
            {
              "type": "card",
              "constraints": {
                "properties": { "brand": { "enum": ["visa", "mastercard"] } }
              }
            }
          ],
          "config": {
            "token_url": "https://api.psp.com/tokens",
            "public_key": "pk_123"
          }
        }
      ]
  }
}
```

##### 2. 토큰 실행(플랫폼 측) { #2-token-execution-platform-side_1 }

플랫폼은 `https://api.psp.com/tokens`를 호출하고(이 엔드포인트의 주체는 플랫폼과
사전에 법적 구속력이 있는 관계를 맺고 있어야 한다(**SHOULD**)) `tok_visa_123`을
받는다(이 값은 보관된 카드나 네트워크 토큰을 나타낼 수 있다).

##### 3. 체크아웃 완료(비즈니스로 보내는 요청) { #3-complete-checkout-request-to-business_1 }

<!-- ucp:example schema=shopping/checkout op=complete direction=request -->
```json
POST /checkout-sessions/{id}/complete

{
  "payment": {
    "instruments": [
      {
        "id": "pi_tok_visa",
        "handler_id": "merchant_tokenizer",
        "type": "card",
        "credential": { "type": "card", "token": "tok_visa_123" }
      }
    ]
  },
  "signals": {
    "dev.ucp.buyer_ip": "203.0.113.42",
    "dev.ucp.user_agent": "Mozilla/5.0 ..."
  }
}
```

##### 4. 챌린지 필요(비즈니스의 응답) { #4-challenge-required-response-from-business }

비즈니스가 청구를 시도하지만, PSP가 3DS를 요구하는 "소프트 거절(Soft Decline)"을
반환한다.

<!-- ucp:example schema=shopping/checkout extract=$.messages target=$.messages -->
```json
HTTP/1.1 200 OK

{
  "status": "requires_escalation",
  "messages": [{
    "type": "error",
    "code": "requires_3ds",
    "content": "bank requires verification.",
    "severity": "requires_buyer_input"
  }],
  "continue_url": "https://psp.com/challenge/123"
}
```

*이제 플랫폼은 사용자가 은행 확인을 마칠 수 있도록 반드시 WebView/창에서
`continue_url`을 열어야 하며(**MUST**), 그 뒤 완료 요청을 재시도해야 한다.*

#### 시나리오 C: 자율 에이전트(AP2) { #scenario-c-autonomous-agent-ap2 }

이 시나리오는 **에이전트에게 권장되는 흐름**을 보여 준다. 에이전트는 세션 토큰
대신 암호학적 위임을 생성한다.

##### 1. 비즈니스 광고 { #1-business-advertisement_1 }

<!-- ucp:example schema=shopping/checkout target=$.ucp -->
```json
{
  "version": "{{ ucp_version }}",
  "payment_handlers": {
    "dev.ucp.ap2_mandate_compatible_handlers": [
        {
          "id": "ap2_234352",
          "version": "{{ ucp_version }}",
          "spec": "https://example.com/specs/ap2-handler",
          "schema": "https://example.com/schemas/ap2-handler.json",
          "available_instruments": [
            {"type": "ap2_mandate"}
          ]
        }
      ]
  }
}
```

##### 2. 에이전트 실행 { #2-agent-execution }

에이전트는 에이전트가 아닌 화면(non-agentic surface)에서 사용자의 개인 키로
객체에 암호학적으로 서명한다.

##### 3. 체크아웃 완료 { #3-complete-checkout }

<!-- ucp:example schema=shopping/checkout op=complete direction=request -->
```json
POST /checkout-sessions/{id}/complete

{
  "payment": {
    "instruments": [
      {
        "id": "pi_ap2_card",
        "handler_id": "ap2_234352",
        "type": "card",
        "credential": {
          "type": "card",
          "token": "eyJhbGciOiJ..." // Token would contain payment_mandate, the signed proof of funds auth
        }
      }
    ]
  },
  "signals": {
    "dev.ucp.buyer_ip": "203.0.113.42",
    "com.example.risk_score": 0.95
  },
  "ap2": {
    "checkout_mandate": "eyJhbGciOiJFUzI1NiIsInR5cCI6InZjK3NkLWp3dCJ9.payload.signature" // Signed proof of checkout terms
  }
}
```

*이로써 비즈니스는 사용자가 이 특정 트랜잭션을 승인했다는 부인할 수 없는 증거를
얻고, 안전하게 자율 처리를 할 수 있다.*

### PCI-DSS 범위 관리 { #pci-dss-scope-management }

#### 플랫폼 범위 { #platform-scope }

대부분의 플랫폼 구현은 다음 방법으로 **PCI-DSS 범위를 피할 수 있다**.

- 불투명한 자격 증명(암호화된 데이터, 토큰 참조 등)을 제공하는 핸들러를 사용한다
- 원본 결제 데이터(카드 번호, CVV 등)에 절대 접근하거나 저장하지 않는다
- 자격 증명을 직접 사용할 수 없는 상태로 전달만 한다
- 원본 자격 증명이 플랫폼을 거치지 않는 PSP 토큰화 결제 핸들러를 사용한다
- FPAN 대신 미리 발급된 카드 네트워크 토큰(`network_token_credential.json`)을
    제시한다. 플랫폼은 원래 계좌 번호를 공유하지 않으며, 토큰은 일치하는
    크립토그램 없이는 쓸 수 없다

#### 비즈니스 범위 { #business-scope }

비즈니스는 다음 방법으로 PCI 범위를 최소화할 수 있다.

- 결제 자격 증명 제공자가 호스팅하는 토큰화를 사용한다(제공자가 자격 증명을
    저장하고, 비즈니스는 토큰 참조를 받는다)
- 암호화된 자격 증명을 제공하는 지갑 제공자(Google Pay, Shop Pay)를 사용한다
- 원본 자격 증명을 절대 로그에 남기지 않는다
- 자격 증명 처리를 PCI 인증을 받은 결제 자격 증명 제공자에게 위임한다

#### 결제 자격 증명 제공자 범위 { #payment-credential-provider-scope }

결제 자격 증명 제공자(PSP, 지갑)는 보통 PCI-DSS Level 1 인증을 받았으며 다음을
처리한다.

- 원본 자격 증명 수집
- 자격 증명 보호(토큰화, 암호화, 안전한 저장)
- 자격 증명 검증과 처리
- PCI 규정을 준수하는 인프라

### 보안 모범 사례 { #security-best-practices }

**비즈니스:**

1. 처리 전에 handler_id를 검증한다(핸들러가 광고한 집합에 속하는지 확인)
2. TEST 환경과 PRODUCTION 환경에 서로 다른 PSP 자격 증명을 사용한다
3. 결제 처리에 멱등성을 구현한다(이중 청구 방지)
4. 결제 이벤트는 기록하되 자격 증명은 기록하지 않는다
5. 적절한 자격 증명 타임아웃을 설정한다
6. 암호학적 증명이 필요한 자율 커머스 시나리오라면
    `dev.ucp.common.payment.ap2_mandate` 확장 지원을 고려한다
    ([AP2 위임 확장](../payment/extensions/ap2-mandates.md) 참고)

**플랫폼:**

1. 체크아웃 API 호출에는 항상 HTTPS를 사용한다
2. 프로토콜을 실행하기 전에 핸들러 설정을 검증한다
3. 자격 증명 획득에 대한 타임아웃 처리를 구현한다
4. 제출 후에는 메모리에서 자격 증명을 지운다
5. 자격 증명 만료를 우아하게 처리한다(필요하면 다시 획득)
6. 자율 에이전트라면 승인에 대한 암호학적 증명을 위해
    `dev.ucp.common.payment.ap2_mandate` 확장 사용을 고려한다
    ([AP2 위임 확장](../payment/extensions/ap2-mandates.md) 참고)

**결제 자격 증명 제공자:**

1. 특정 비즈니스용으로 자격 증명을 보호한다(암호화, 토큰화 또는 핸들러별
    방식)
2. 자격 증명 획득에 요청 속도 제한을 구현한다
3. 자격 증명을 제공하기 전에 플랫폼의 권한을 검증한다
4. 합리적인 자격 증명 만료 시간을 설정한다(예: 토큰은 15분, 시간 제한이 있는
    암호화 페이로드)
5. 플랫폼이 자격 증명을 직접 사용할 수 없도록 보장한다(의도된 비즈니스만 사용
    가능)

### 사기 방지 연동 { #fraud-prevention-integration }

UCP는 [시그널](#signals)과 결제 아키텍처를 통해 사기 방지를 지원한다.

- 플랫폼은 카탈로그, 장바구니, 체크아웃 요청에 트랜잭션 환경
    [시그널](#signals)(IP, 사용자 에이전트)을 제공한다
- 비즈니스는 핸들러 설정에서 추가 필드를 요구할 수 있다(예: 3DS 요구 사항)
- 결제 자격 증명 제공자는 자격 증명 획득 과정에서 위험도를 평가할 수 있다
- 비즈니스는 고위험 트랜잭션을 거부하고 시그널 피드백을 통해 추가 검증을 요청할
    수 있다

### 결제 아키텍처 확장 { #payment-architecture-extensions }

앞서 설명한 핵심 결제 아키텍처는 특수한 사용 사례에 맞게 확장할 수 있다.

- **AP2 위임 확장**(`dev.ucp.common.payment.ap2_mandate`): 부인할 수 없는 증거가
    필요한 자율 커머스 시나리오를 위해 사용자 승인에 대한 암호학적 증명을 추가한다.
    [AP2 위임 확장](../payment/extensions/ap2-mandates.md)을 참고한다.

- **커스텀 핸들러 유형**: 결제 자격 증명 제공자는 새로운 결제 수단을 지원하기
    위해 커스텀 핸들러를 정의할 수 있다. 자세한 내용은
    [결제 핸들러 가이드](../payment/guide.md)를 참고한다.

이 확장 모델 덕분에 핵심 아키텍처는 단순하게 유지하면서도, 필요할 때 고급 보안과
규정 준수 요구 사항을 지원할 수 있다.

## 전송 계층 { #transport-layer }

UCP는 여러 전송 프로토콜을 지원한다. 플랫폼과 비즈니스는 사실상 프로필의
`services`를 통해 전송 방식을 협상한다.

### REST 전송 방식(코어) { #rest-transport-core }

UCP는 RESTful 패턴을 사용하는 **HTTP/1.1**(또는 그 이상)을 지원한다.

- **Content-Type:** 요청과 응답은 반드시 `application/json`을 사용해야
    한다(**MUST**).
- **메서드:** 구현체는 반드시 표준 HTTP 동사를 사용해야 한다(**MUST**)(예: 생성에는
    `POST`, 조회에는 `GET`).
- **상태 코드:** 구현체는 반드시 표준 HTTP 상태 코드를 사용해야 한다(**MUST**)
    (예: 200, 201, 400, 401, 500).

### Model Context Protocol(MCP) { #model-context-protocol-mcp }

UCP는 **[MCP 프로토콜](https://modelcontextprotocol.io/specification/)**(JSON-RPC
위에서 동작)을 지원한다.

#### 요청 형식 { #request-format }

MCP 요청은 `tools/call` 메서드를 사용하며, `params.name`에 오퍼레이션 이름을,
`params.arguments`에 UCP 페이로드를 담는다.

<!-- ucp:example schema=shopping/checkout op=create direction=request extract=$.params.arguments.checkout -->
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "create_checkout",
    "arguments": {
      "meta": {"ucp-agent": {"profile": "https://..."}},
      "checkout": {"line_items": [...]}
    }
  },
  "id": 1
}
```

#### 응답 형식 { #response-format }

MCP 도구 응답은 하위 호환성을 위해 이중 출력 패턴을 사용한다. UCP MCP 서버는:

- 반드시 `structuredContent`에 UCP 응답 페이로드를 반환해야 한다(**MUST**)
- 도구 정의에 해당 기능에 맞는 UCP JSON Schema를 참조하는 `outputSchema`를
    선언해야 한다(**SHOULD**)
- `structuredContent`를 지원하지 않는 클라이언트와의 하위 호환성을 위해
    `content[]`에 직렬화된 JSON도 반환해야 한다(**SHOULD**). 문서 예시에서는
    가독성을 위해 이 직렬화된 JSON 문자열을 `…`로 줄여 표기한다.

<!-- ucp:example schema=shopping/checkout extract=$.result.structuredContent.ucp target=$.ucp -->
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "structuredContent": {
      "ucp": {
        "version": "{{ ucp_version }}",
        "payment_handlers": {},
        "capabilities": {...}
      },
      "id": "checkout_abc123",
      "status": "incomplete"
      // ... other checkout fields
    },
    "content": [
      {"type": "text", "text": "{\"ucp\":{…},…}"}
    ]
  }
}
```

### Agent-to-Agent Protocol(A2A) { #agent-to-agent-protocol-a2a }

비즈니스는 UCP를 A2A 확장(A2A Extension)으로 지원하는 A2A 에이전트를 노출할 수
있으며(**MAY**), 이를 통해 구조화된 UCP 데이터 타입으로 플랫폼과 연동할 수 있다.

### 임베디드 프로토콜(EP) { #embedded-protocol-ep }

비즈니스는 적격 호스트에 인터페이스를 임베드할 수 있다(**MAY**). 이 호스트는
사용자가 인터페이스와 상호작용할 때 이벤트를 받고, 핵심 사용자 행동을 위임받는다.

시작은 비즈니스가 반환하는 `continue_url`을 통해 이루어진다.

## 표준 기능 { #standard-capabilities }

UCP는 다음과 같은 표준 기능 집합을 정의한다.

| 기능 이름            | ID(URI)                                                               | 설명                                                                                                         |
| :------------------- | :-------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------- |
| **Ask**              | [schemas/common/ask.json](site:schemas/common/ask.json)               | 비즈니스의 리소스, 정책, 서비스에 관한 자연어 질문에 답한다. 읽기 전용이다.                                  |
| **Cart**.            | [schemas/shopping/cart.json](site:schemas/shopping/cart.json)         | 구매 의도가 확정되기 전에 장바구니를 구성할 수 있게 한다.                                                    |
| **Checkout**         | [schemas/shopping/checkout.json](site:schemas/shopping/checkout.json) | 장바구니 관리와 세금 계산을 포함해 체크아웃 세션의 생성과 관리를 돕는다.                                     |
| **Identity Linking** | -                                                                     | 플랫폼이 OAuth 2.0으로 권한을 얻어 사용자를 대신해 행동할 수 있게 한다.                                      |
| **Order**            | [schemas/shopping/order.json](site:schemas/shopping/order.json)       | 비즈니스가 주문 수명 주기(배송, 배달, 반품)에 대한 비동기 업데이트를 푸시할 수 있게 한다.                    |

### 정의와 확장 { #definition-extensions }

각 기능의 엔드포인트, 스키마, 유효한 확장에 대한 상세 정의는 각 명세 파일에 있다.
확장은 보통 버전이 매겨지며 부모 기능과 함께 정의된다.

## 수량과 단위 { #quantities-and-units }

UCP는 스키마에 정수 `quantity`, `quantity_unit`, 또는 공유 측정값 타입이 있는 곳
어디서나 공통 수량 표현을 사용한다.

`quantity`는 **스텝(step)** 수를 나타내는 정수다. 단위 기술자(unit descriptor)는
다음으로 구성된다.

- `unit` — 필수이며 안정적인 기계용 식별자.
- `scale` — 선택 사항인 0 이상의 정수로, 최댓값은 15다(정수 범위에서 도출한
    상한이다. 아래 참고). 유효값은 제공된 값이며, 생략하면 `0`이다.
- `display_text` — 필수이며 단위를 나타내는 출력 가능한 레이블.

한 스텝은 `unit`의 `10^-scale`이다. 공유 측정값 타입에는 필수 정수 `value`가
추가되며, 이 역시 해당 스텝의 수다. 이 수는 정수이므로 `scale`이 표현의 단위
간격을 고정한다. 단위 기술자의 기계적 정체성은 (`unit`, 유효 `scale`) 쌍이며,
`display_text`는 그 정체성에 포함되지 않는다. 이 정체성은 단위 기술자에만
적용된다. 구매 가능한 상품을 식별하거나 판매 단위 하나를 완전히 기술하지는 않는다.

기본 판매 기준은 `each`이며, 기계적 정체성은 (`C62`, `0`)이다. `C62`는 유엔 무역
촉진 및 전자 비즈니스 센터(UN/CEFACT) 권고 20(Rec20)에서 one/each를 나타내는 공통
코드다. 비즈니스는 이 기본값을 나타내기 위해 권위 있는 비즈니스 표현에서
`quantity_unit`을 생략할 수 있다(**MAY**). 비즈니스나 플랫폼이 `unit`이 `C62`인
기술자를 포함할 때는 반드시 유효 `scale`로 `0`을 사용해야 한다(**MUST**).
`scale`은 생략하거나 명시적으로 `0`으로만 설정할 수 있다.

UCP는 부동소수점 수를 전송하지 않는다. 수량 연산은 금액으로 이어진다.
`price × quantity × 10^-scale`이 라인의 가격을 정하고, `fulfilled`는 풀필먼트
이벤트를 거치며 누적되며, 상태는 `fulfilled == total`에서 도출된다. 그래서 수량은
금액과 같은 방식으로 표현한다. `amount`가 `currency`와 짝을 이루듯, 정수 개수에
선언된 해석을 더하는 방식이다. 정수 개수를 쓰면 어느 언어에서든 모든 합계와 비교가
정확하게 유지되므로, UCP는 수량 수명 주기 어디에서도 반올림 허용 오차나 엡실론
비교를 정의하지 않는다. 풀필먼트된 수량이 주문 수량과 정당하게 다른 경우(2.00 lb
주문에 1.90 lb를 담은 경우)는 비교의 오차 범위로 흡수하는 수치 오류가 아니라,
금액과 수량을 함께 조정하는
[조정(adjustments)](../shopping/order/index.md#adjustments)으로 정산하는 상거래상의
사실이다.

수량을 읽을 때는 연산도 단위 지식도 필요 없다. 소수점을 `scale` 자리만큼 옮기고
`display_text`를 붙이면 된다. `{ "scale": 2, "display_text": "kg" }`과 함께 쓰인
`150`은 `1.50 kg`으로 표시되며, Rec20 코드든 커스텀 단위든 같은 코드 경로로
처리된다. 통화 지수와 달리 `scale`은 정적 테이블이 아니라 품목별 데이터다. 그래서
권위 있는 응답은 `each`가 아닌 모든 라인에 항상 자체 기술자를 담는다.

### 정수 범위와 수집 { #integer-range-and-ingestion }

UCP의 모든 정수 값 필드(금액, 수량 스텝 수, 측정값)는 JSON 정수다. 각 필드의
스키마가 부호와 범위를 선언하며, 모두 ±(2^53 − 1)(±9,007,199,254,740,991)로
상한이 정해져 있다. 이 범위 안에서는 모든 JSON 구현이 정수 값에 대해 정확히
일치하며
([RFC 8259](https://www.rfc-editor.org/rfc/rfc8259.html){ target="_blank" },
섹션 6),
[AP2 위임 서명](../payment/extensions/ap2-mandates.md#canonicalization)에 필요한
[JCS](https://www.rfc-editor.org/rfc/rfc8785.html){ target="_blank" }
정규화도 이 범위 안에서 정의된다. `scale`의 최댓값 15도 같은 상한에서 도출된다.
scale이 16이면 한 단위 전체(10^16 스텝)를 표현할 수 없기 때문이다. 범위를 벗어난
값은 스키마상 유효하지 않으며, 다른 유효하지 않은 페이로드와 마찬가지로 거부된다.

이 값들에 대한 연산은 반드시 정확해야 한다(**MUST**). 전송 범위 안에서는 IEEE 754
binary64(`JSON.parse`가 만드는 JavaScript `Number`)가 모든 정수를 정확히 담는다.
`amount × quantity` 같은 곱은 64비트를 넘을 수 있으므로 더 넓은 정수나 임의 정밀도
정수를 쓰거나 오버플로 검사를 한다. 정확하고 범위 안에 드는 결과를 만들 수 없는
구현체는 근사값이나 랩어라운드된 값을 내보내거나, 표시하거나, 그에 따라 동작하는
대신 반드시 오류를 드러내야 한다(**MUST**).

UCP 데이터가 외부 시스템으로 넘어갈 때, 구현체는 수집 시점에 한 번만 변환해야
한다(**SHOULD**). 즉 선언된 scale(또는 통화 지수)을 적용해 정확한 십진 타입(SQL
`NUMERIC`, Java `BigDecimal`, Python `Decimal`)으로 바꾸거나, (value, scale) 쌍을
그대로 전달한다. scale 적용이나 값을 다루는 연산을 이진 부동소수점으로 수행하지
않아야 한다(**SHOULD NOT**).

### 단위 어휘 { #unit-vocabulary }

비즈니스는 단위를 정확히 식별하는 코드가 없는 경우가 아니라면 정확한 Rec20 공통
코드를 사용해야 한다(**SHOULD**). 단위를 정확히 식별하는 Rec20 코드가 없으면
비즈니스는 커스텀 단위 식별자를 사용할 수 있다(**MAY**). 그렇게 한다면 비즈니스는
같은 단위에 그 식별자를 반드시 일관되게 사용해야 한다(**MUST**). 플랫폼은 인식하지
못한 `unit` 값을 반드시 불투명한 값으로 취급해야 한다(**MUST**). 다음 표는 전체
목록이 아니다.

| 코드  | 단위         |
| :---- | :----------- |
| `C62` | 하나 / `each` |
| `KGM` | 킬로그램     |
| `GRM` | 그램         |
| `LBR` | 파운드       |
| `MLT` | 밀리리터     |
| `LTR` | 리터         |
| `MTR` | 미터         |
| `INH` | 인치         |
| `YRD` | 야드         |
| `FTK` | 제곱피트     |
| `MTK` | 제곱미터     |
| `HUR` | 시간         |
| `MIN` | 분           |

Rec20에는 UN/CEFACT 권고 21(Rec21)에서 파생된 X 접두사 포장 단위가 포함되어 있다.
UCP는 이 값들을 `quantity_unit`에서 의도적으로 제외한다. 비즈니스는 반드시 포장
형태를 구매 가능한 변형(variant)의 정체성에 포함시키고 포장 개수를 `each`로 세야
한다(**MUST**). 비즈니스는 Rec21에서 파생된 X 접두사 포장 코드를 `quantity_unit`으로
사용해서는 안 된다(**MUST NOT**).

### 표시 텍스트 { #display-text }

단위 기술자를 보낼 때 비즈니스나 플랫폼은 반드시 `display_text`를 포함해야
한다(**MUST**). 플랫폼은 `unit`을 인식하지 못할 때 반드시 그 값을 사용해야
한다(**MUST**). 인식한 Rec20 코드라면 플랫폼은 자체 현지화 레이블로 대체할 수
있다(**MAY**). 비즈니스와 플랫폼은 기계적 정체성을 매칭하거나 수량을 변환할 때
`display_text`를 입력으로 사용해서는 안 된다(**MUST NOT**).

### 주문 증분 { #ordering-increment }

판매 기준 기술자(`quantity_unit`)는 `increment`를 선언할 수 있다(**MAY**).
`increment`는 스텝 단위로 표시하는 선택적 양의 정수이며, 유효값은 제공된 값이고
생략하면 `1`이다. 증분은 판매 기준에만 있다. 단순 단위 기술자와 공유 측정값
타입에는 없다. 증분은 비즈니스가 판매하는 주문 단위 간격을 선언한다. 예를 들어
`scale`이 `2`이고 `increment`가 `25`인, 파운드 단위로 파는 상품은 0.25 lb의
배수로 판매된다.

`scale`과 `increment`는 역할이 다르다. `scale`은 어떤 수량이든 표현할 수 있는
범위를 제한하고, `increment`는 플랫폼이 요청하는 수량의 형태를 정한다. 증분은
표현상의 한계가 아니라 권고적인 머천다이징 정책이다. 플랫폼이 작성하는 수량은
라인의 유효 증분의 정수배여야 하지만(**SHOULD**), 비즈니스가 작성하는 수량(체크아웃
수정, 풀필먼트 이벤트, 조정)은 `scale`의 제약만 받는다. `increment`는 단위 기술자의
기계적 정체성에 포함되지 않으며, 불일치 비교에 참여해서는 안 된다(**MUST NOT**).

요청 측 주장, 불일치 처리, 응답 에코, 증분에 맞지 않는 요청 처리, 가격 책정, 수명
주기 동작은 공유 표현을 사용하는 기능이 정의한다.

## 요청 제약 { #request-constraints }

기능과 확장이 협상되고 나면, 해석된 UCP 요청 스키마가 오퍼레이션에 허용되는 필드와
구조를 정의한다. 권위 있는 응답에서 비즈니스는 `ucp.request_constraints`를 사용해,
다음 플랫폼 요청의 데이터를 평가할 때 적용할 추가 규칙을 알릴 수 있다. 예를 들어
라인 아이템 수량을 정확히 판매 기준 `100` 스텝으로 제한하거나, 제출되는 결제
수단에 `billing_address`를 포함하도록 요구할 수 있다. 플랫폼은 제출 전에 이 제약을
평가해, 비즈니스가 이미 거부하겠다고 밝힌 요청 데이터 때문에 왕복 요청을 낭비하지
않을 수 있다.

`ucp.request_constraints`는 권위 있는 오퍼레이션 응답에서만 사용하며, 디스커버리
프로필이나 오퍼레이션 요청에서는 효과가 없다.

### 검증 모델 { #validation-model }

제출된 요청은 해석된 요청 스키마를 충족하고, 요청 제약(Request Constraints)이
선택하는 모든 객체가 대응하는 제약 표현식(Constraint Expression)을 충족할 때에만
요청 제약에 따라 유효하다. 요청 제약은 해석된 요청 스키마를 좁힐 뿐이며, 그
스키마가 거부하는 요청을 유효하게 만들 수는 없다. 비즈니스는 다음과 같이 요청을
평가한다.

```text
valid = validate(resolved_request_schema, request)

for each constraint:
  objects = select(request, effective_path(constraint))
  valid = valid AND validate_all(
    constraint_expression(constraint),
    objects
  )

return valid
```

플랫폼은 같은 검사를 사전 점검(preflight)으로 수행할 수 있다(**MAY**). 선택한 각
값에 대해 플랫폼은 반드시 그 값의 유효 경로와 전체 제약 표현식을 사용해야
한다(**MUST**). 플랫폼은 사전 점검 결과와 관계없이 요청을 제출할 수 있다(**MAY**).
권위 있는 판단은 비즈니스의 평가다.

### 제약 표현식 { #constraint-expression }

`request_constraints` 값과 그 안에 중첩된 모든 제약 객체는 내장된 JSON Schema Draft
2020-12 언어를 사용한다. 바깥 값에는 선택적인 `path`가 추가로 들어갈 수 있지만,
중첩된 제약 객체에는 들어갈 수 없다. 이 문법은 `ucp`를 허용하지 않는다.
`properties`의 키는 선택된 요청 객체의 필드 이름을 가리킨다.

제약은 객체 제약(Object Constraint)에서 시작한다. 객체 제약은 `properties`와
`anyOf`를 통해 중첩될 수 있다. 값 제약(Value Constraint)은 객체 제약의 `properties`
맵 안에서 값으로만 나타난다.

| 위치 | 허용 멤버 | 형태와 동작 |
| :-- | :-- | :-- |
| 객체 제약 | `required`, `properties`, `anyOf` | `required`는 고유한 필드 이름의 배열이다. `properties`는 필드 이름을 객체 제약 또는 값 제약에 매핑한다. `anyOf`는 비어 있지 않은 객체 제약으로 이루어진 비어 있지 않은 배열이며, 객체는 그중 적어도 하나를 충족해야 한다. 빈 객체 제약은 `anyOf` 분기를 제외한 모든 객체 제약 위치에서 아무 효과 없는 유효한 값이다. |
| 값 제약 | `enum`, `const` | `enum`은 고유한 JSON 값으로 이루어진 비어 있지 않은 배열이다. `const`는 임의의 JSON 값이다. 적어도 한 멤버가 있어야 하며, 둘 다 있으면 둘 다 적용된다. |

어느 문법 위치에서도 그 밖의 멤버는 허용되지 않는다.

같은 객체 제약에 있는 멤버는 모두 적용된다. `anyOf`는 형제 멤버를 좁히거나,
덮어쓰거나, 대체하지 않는다. 객체는 모든 형제 멤버와 적어도 하나의 분기를 충족해야
한다. 각 분기는 평범한 객체 제약이며 `path`를 가질 수 없으므로, 모든 분기는 같은
선택된 객체를 대상으로 평가된다.

분기는 대안일 뿐 분할이 아니다. 둘 이상의 분기를 충족하는 객체도 유효하다. 분기
안에서 `properties`는 해당 멤버가 있을 때만 그 멤버를 제약한다. 따라서
`properties`만으로 판별자(discriminator)를 고정한 분기는 그 판별자를 생략한
객체로도 충족된다. 판별자를 분기의 `required`에 명시하면 그 분기는 자신이 기술하는
형태에만 매칭된다.

이 문법은 자신이 묶이는 객체와 독립적으로 정의된다. 요청 제약은 이 문법을 다음
요청의 객체에 묶고 `path`를 추가한다. 다른 UCP 스키마는 선언이 이미 제약 대상
객체를 식별하는 곳에서 이 문법을 재사용한다. 예를 들어
[`available_instruments[].constraints`](site:schemas/common/types/available_payment_instrument.json)의
대상 객체는 해당 항목의 `type`에 대해 결제 수단 스키마가 선언한
`constraint_target`이다.

### 경로 { #path }

모든 `request_constraints` 값에는 정확히 하나의 유효 경로가 있다. `path`를 생략하면
유효 경로는 권위 있는 응답 루트에서, `request_constraints`를 담은 `ucp` 멤버를 가진
구조화된 응답 객체까지의
[RFC 9535](https://www.rfc-editor.org/rfc/rfc9535.html){ target="_blank" }
Normalized Path다. 비즈니스가 `path`를 제공하면 그 값이 유효 경로가 되며, 원래라면
도출되었을 Normalized Path를 대체한다. 비즈니스는 반드시 이를 완전한 RFC 9535
JSONPath 쿼리로 만들어야 한다(**MUST**). 두 경우 모두 유효 경로는 해당 리소스에 대한
다음 논리적 UCP 요청을 대상으로 평가된다. 따라서 유효 경로는 `messages[].path`처럼
UCP의 다른 곳에서 응답을 대상으로 하는 경로(자신을 담은 응답을 대상으로 평가됨)와
다르다.

경로를 생략하면 다음 요청에 대한 위치상 대응 관계가 성립할 뿐, 안정적인 정체성이
성립하지는 않는다. 그 요청 전에 항목 순서가 바뀌어도 배열 인덱스는 여전히 같은
위치를 가리킨다. 구조화된 응답 객체의 Normalized Path가 의도한 요청 객체를
식별하지 못하면, 비즈니스는 반드시 명시적인 `path`를 제공해야 한다(**MUST**).
제약이 다음 요청에서도 안정적인 정체성을 따라가야 한다면, 비즈니스는 반드시 그
연관 관계를 담은 명시적 쿼리를 사용해야 한다(**MUST**).

데이터로 쿼리를 구성하는 비즈니스는 반드시 각 동적 값을 유효한 RFC 9535 리터럴로
직렬화하고 이스케이프해야 하며(**MUST**), 안전하지 않은 문자열 연결을 사용해서는 안
된다(**MUST NOT**).

값을 내보내기 전에 비즈니스는 반드시 전체 값을 공유
[요청 제약](site:schemas/common/types/request_constraints.json) 스키마에 대해
검증해야 하며(**MUST**), 플랫폼도 사전 점검에 그 값을 사용하기 전에 반드시 같은
검증을 해야 한다(**MUST**). `path`는 객체를 선택하고, 나머지 멤버는
[제약 표현식](site:schemas/common/types/constraint_expression.json)을 이룬다.

제약은 오퍼레이션 이름에 묶이지 않는다. 제약은 경로가 선택하는 모든 객체에
적용된다. 객체를 하나도 선택하지 않는 경로는 효과가 없다. `request_constraints`가
응답 루트의 `ucp` 멤버에 있고 `path`가 생략되었다면 유효 경로는 `$`다.

여러 경로가 같은 객체를 선택하면, 대응하는 모든 제약 표현식이 적용된다. 공유 요청
제약 스키마는 각 값을 독립적으로 검증하므로, 겹치는 경로에 걸친 표현식들이 모두
충족 가능하다고 보장할 수 없다. 비즈니스는 같은 객체에 적용될 수 있는 표현식들이
함께 충족 가능하도록 반드시 보장해야 한다(**MUST**). 같은 속성에 대한 모순된 요구
사항은 비즈니스의 작성 오류다.

겹치는 값에 대해 사전 점검을 수행하는 플랫폼은 자신이 선택한 모든 값을 평가한다.
선택한 값 중 하나라도 구체적인 요청에 대해 실패하면 사전 점검은 실패한다. 우선순위나
덮어쓰기는 없다.

### 지침 { #guidelines }

#### 비즈니스 { #business }

비즈니스는 다음 플랫폼 요청의 데이터에 대해 강제할 규칙을 기술하기 위해 권위 있는
응답에 `ucp.request_constraints`를 포함할 수 있다(**MAY**). 비즈니스는 반드시 공유
요청 제약 스키마를 준수하는 요청 제약을 내보내야 하고(**MUST**), 객체만 선택하는
유효한 경로를 사용해야 하며(**MUST**), 자신이 내보낸 모든 제약을 그다음 요청에
반드시 강제해야 한다(**MUST**).

비즈니스는 광고한 제약을 계속 강제할 수 있는 경우가 아니라면, 다음 요청 전에 바뀔
수 있는 규칙에 대해서는 요청 제약을 내보내지 않아야 한다(**SHOULD NOT**). 재고
가용성, 사기 판정, 결제 승인 같은 실행 시점 조건은 여전히 오퍼레이션의 기존 결과와
`messages`가 다룬다.

#### 플랫폼 { #platform }

플랫폼은 요청 데이터를 제출하기 전에 `ucp.request_constraints`를 사전 점검에 사용할
수 있다(**MAY**). 플랫폼은 지원하는 값의 어떤 부분집합이든 평가할 수 있다(**MAY**).
이 사전 점검은 선택 사항이며 권고적이다.

형식이 잘못되었거나 지원하지 않는 값, 경로가 객체가 아닌 것을 선택하는 값, 플랫폼이
자원 한도 안에서 평가할 수 없는 값은 사전 점검에 사용할 수 없는 값이며, 통과나
실패로 간주하지 않는다. 사전 점검을 수행하는 플랫폼은 사용할 수 없는 값 전체를
반드시 건너뛰고(**MUST**) 선택한 다른 값으로 계속 진행해야 한다. 완전한 평가에서
위반이 발견되면 그 값은 사전 점검에 실패한다.

사전 점검 성공 결과는 플랫폼이 평가한 값에만 해당한다. UCP는 사전 점검 결과를 위한
새로운 전송 상태나 이슈 표시 필드를 정의하지 않는다.

### 범위와 수명 주기 { #scope-and-lifecycle }

비즈니스의 권위 있는 각 리소스 응답은 해당 리소스에 대한 다음 요청의 요청 제약을
제공한다. 그 요청에 대한 권위 있는 응답은 그다음 요청의 제약을 제공하며 이전 집합을
대체한다. 생략하면 집합이 비워진다. 새 집합에 유효하지 않은 값이 있다고 해서 이전
값이 남지는 않으며, 집합은 `path` 기준으로 병합되지 않는다.

[예약된 `ucp` 멤버](#the-ucp-protocol-namespace) 규칙에 따라 적격한 구조화 응답
객체에 붙은 `request_constraints` 값만 집합을 이룬다. 딕셔너리 키는 데이터다.

권위 있는 리소스가 없는 응답은 다음 요청에 대한 요청 제약 집합을 제공하지 않는다.
권위 있는 리소스를 담은 응답은 애플리케이션 오류를 보고하더라도 집합을 제공한다.
권위 있는 리소스의 일부만 담은 표현은 리소스 계약이 그 범위를 정의하는 경우에만
집합을 제공한다.

### 오퍼레이션 결과 { #operation-outcomes }

요청 제약은 다음 요청에 대해 사전에, 기계가 평가할 수 있는 사전 점검을 제공한다.
`messages`는 제출된 요청의 결과를 보고하며, 여기에는 런타임 결과도 포함된다. 해석된
요청 스키마와 요청 제약 양쪽의 검증을 통과하더라도 확립되는 것은 스키마상의 유효성뿐이다.
요청은 여전히 다른 비즈니스 규칙에 따라 실패할 수 있다. 제출된 요청과 런타임 결과는
`messages`를 포함해 해당 오퍼레이션의 기존 의미론과 결과/오류 계약이 계속 다룬다.
요청 제약은 결과나 오류 코드를 추가하지 않는다.

### 예시 { #examples }

#### 장바구니 전체 수량과 대상 지정 수량 { #basket-wide-and-targeted-quantities }

다음 장바구니 응답들은 서로 다른 범위를 보여 주는 대안적인 예시다.

=== "장바구니 전체"

    <!-- ucp:example schema=shopping/cart op=read direction=response -->
    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "request_constraints": {
          "path": "$['line_items'][*]",
          "properties": {
            "quantity": {"const": 1}
          }
        }
      },
      "id": "cart_123",
      "line_items": [
        {
          "id": "line_123",
          "item": {
            "id": "sku_123",
            "title": "Bulk screws",
            "price": 1200
          },
          "quantity": 1,
          "totals": [
            {"type": "subtotal", "amount": 1200},
            {"type": "total", "amount": 1200}
          ]
        }
      ],
      "currency": "USD",
      "totals": [
        {"type": "subtotal", "amount": 1200},
        {"type": "total", "amount": 1200}
      ]
    }
    ```

=== "대상 지정"

    <!-- ucp:example schema=shopping/cart op=read direction=response -->
    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}"
      },
      "id": "cart_123",
      "line_items": [
        {
          "id": "line_123",
          "item": {
            "id": "sku_123",
            "title": "Bulk screws",
            "price": 1200
          },
          "quantity": 100,
          "totals": [
            {"type": "subtotal", "amount": 120000},
            {"type": "total", "amount": 120000}
          ],
          "ucp": {
            "request_constraints": {
              "path": "$['line_items'][?@['id'] == 'line_123']",
              "properties": {
                "quantity": {"const": 100}
              }
            }
          }
        }
      ],
      "currency": "USD",
      "totals": [
        {"type": "subtotal", "amount": 120000},
        {"type": "total", "amount": 120000}
      ]
    }
    ```

장바구니 전체 응답은 다음 요청의 모든 라인 아이템에 수량 `1`을 적용하는 루트 제약을
내보낸다. 대상 지정 응답은 주변(ambient) 로컬 작성 방식과 안정적 ID 경로를 사용해
`id`가 `line_123`인 라인 아이템에만 수량 `100`을 적용한다. 안정적 ID로 다시 묶으면
순서가 바뀌어도 유지된다. 그 경로에 일치하는 항목이 없으면 객체를 하나도 선택하지
않으므로 효과가 없다. 두 응답은 서로 별개인 대안이다. 이를 그대로 합치면
`line_123`에 모순된 제약이 생기며, 이는 비즈니스의 작성 오류다.

#### 고정된 협상 할인 코드 { #locked-negotiated-discount-codes }

이 체크아웃 응답에는 할인 확장이 활성화되어 있으며, 다음 체크아웃 업데이트 요청에
대한 요청 제약을 광고한다. `request_constraints`가 체크아웃 응답 루트의 `ucp`에
있으므로, `path`를 생략하면 그 구조화된 응답 객체의 Normalized Path가 도출된다. 이처럼
루트에 둔 경우 도출된 경로는 `$`이며, 이는 다음 요청의 루트를 선택한다.

<!-- ucp:example schema=shopping/discount def=dev.ucp.shopping.checkout op=update direction=response -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "status": "success",
    "capabilities": {
      "dev.ucp.shopping.checkout": [
        {"version": "{{ ucp_version }}"}
      ],
      "dev.ucp.shopping.discount": [
        {"version": "{{ ucp_version }}"}
      ]
    },
    "payment_handlers": {},
    "request_constraints": {
      "required": ["discounts"],
      "properties": {
        "discounts": {
          "required": ["codes"],
          "properties": {
            "codes": {"const": ["ACME-X7Q9-L2M4"]}
          }
        }
      }
    }
  },
  "id": "checkout_123",
  "status": "incomplete",
  "currency": "USD",
  "line_items": [
    {
      "id": "line_123",
      "item": {
        "id": "sku_123",
        "title": "Bulk screws",
        "price": 1200
      },
      "quantity": 24,
      "totals": [
        {"type": "subtotal", "amount": 28800},
        {"type": "total", "amount": 28800}
      ]
    }
  ],
  "totals": [
    {"type": "subtotal", "amount": 28800},
    {"type": "total", "amount": 28800}
  ],
  "links": [
    {
      "type": "terms_of_service",
      "url": "https://business.example/terms"
    }
  ],
  "discounts": {
    "codes": ["ACME-X7Q9-L2M4"]
  }
}
```

다음 체크아웃 업데이트 요청은 해석된 요청 스키마를 충족하고, `discounts.codes`를
포함하며, 정확히 `["ACME-X7Q9-L2M4"]`를 제공할 때에만 유효하다.

#### 제출된 카드 결제 수단의 청구지 주소 { #billing-address-on-a-submitted-card-instrument }

이 예시에서 비즈니스는 사용 가능한 카드 결제 수단에 `ucp.request_constraints`를
내보내, 다음 요청에 일치하는 카드 결제 수단이 제출되면 `billing_address`를 요구한다.

<!-- ucp:example schema=common/types/available_payment_instrument op=read direction=response -->
```json
{
  "type": "card",
  "ucp": {
    "request_constraints": {
      "path": "$['payment']['instruments'][?@['handler_id'] == 'processor_1' && @['type'] == 'card']",
      "required": ["billing_address"]
    }
  }
}
```

이 조각은 이를 담고 있는 권위 있는 응답의 결제 핸들러 선언이 `id: processor_1`을
가진다고 가정한다. 명시적 `path`는 응답의 `available_instruments[]` 형태에서 제출되는
`payment.instruments[]`로 건너가며, `handler_id`와 `type`으로 결제 수단을 매칭한다.
다음 요청에 일치하는 항목이 있으면, 제약은 일치하는 모든 결제 수단에
`billing_address`를 요구한다. 더 강한 연관 관계는 결제 핸들러나 결제 수단 계약이
정의한다. 이 예시는 카드 브랜드, 자격 증명, 지원 여부, 가용성, 결제 정책을 정의하지
않는다.

#### 제출된 자격 증명에 대한 대안적 검증 요구 사항 { #alternative-verification-requirements-on-a-submitted-credential }

비즈니스가 자격 증명 형태를 둘 이상 받으면서 각각 다른 검증 데이터를 요구하는
경우다. 이 예시에서 PAN에는 `cvc`가 있어야 하고, 네트워크 토큰에는 `cryptogram`과
`eci_value`가 있어야 한다. 각 자격 증명 계열은 별도의 스키마이므로, 모든 분기는
자격 증명 자체의 `type`으로 판별하며 어떤 규칙도 형제 필드에 따라 분기할 필요가 없다.

<!-- ucp:example schema=common/types/available_payment_instrument op=read direction=response -->
```json
{
  "type": "card",
  "ucp": {
    "request_constraints": {
      "path": "$['payment']['instruments'][?@['handler_id'] == 'processor_1' && @['type'] == 'card']",
      "required": ["credential"],
      "properties": {
        "credential": {
          "anyOf": [
            {
              "properties": {"type": {"const": "pan"}},
              "required": ["cvc"]
            },
            {
              "properties": {"type": {"const": "network_token"}},
              "required": ["cryptogram", "eci_value"]
            }
          ]
        }
      }
    }
  }
}
```

경로 하나가 제출된 결제 수단을 선택하고, 객체 제약 하나가 그것을 기술한다. 형제
`required`는 일치하는 모든 결제 수단에 적용되고, 이어서 `anyOf` 분기가 중첩된
`credential` 객체에 적용되며, 이 객체는 적어도 하나의 분기를 충족해야 한다. 각 분기는
`const`로 `type`을 고정하므로 자신이 기술하는 자격 증명 계열에만 매칭된다.
[`payment_credential.json`](site:schemas/common/types/payment_credential.json)이 이미
모든 자격 증명에 `type`을 요구하므로, 어떤 분기도 `required`에 그것을 명시할 필요가
없다. `cvc`가 없는 [PAN 자격 증명](site:schemas/common/types/pan_credential.json)은
실패하고, `eci_value`가 없는
[네트워크 토큰](site:schemas/common/types/network_token_credential.json)도 실패한다.

모든 분기가 판별자를 고정하므로, 분기 집합은 허용되는 자격 증명 계열을 닫는
역할도 한다. 핸들러
[토큰 자격 증명](site:schemas/common/types/token_credential.json)은 이 위치에서
유효한 자격 증명이지만 어느 분기도 충족하지 않으므로, 이 비즈니스는 이 경로에서
그것을 받지 않는다. 나중에 다른 계열을 받으려는 비즈니스는 그 계열에 대한 분기를
추가한다.

이 규칙은 따로따로 대상을 지정한 두 제약으로는 표현할 수 없다. 요청 제약은 논리곱으로
결합되므로, `cvc`를 요구하는 값과 `cryptogram`을 요구하는 값이 있으면 둘 다 요구하게
된다. 대신 경로 필터로 판별하면(한 값에서는 `pan` 자격 증명을, 다른 값에서는
`network_token` 자격 증명을 선택) 조건부 로직이 선택자로 옮겨 가는데, 경로는 그런
로직을 담지 않는다.

## 액션 { #actions }

액션(Action)은 플랫폼이 처리해야 할, 확장이 정의한 미처리 작업 단위다. 액션이
존재한다는 것은 그 액션 유형이 정의하는 효과가 보류(gate)되어 있다는 뜻이다. 액션은
응답에서만, `actions` 맵 아래에 나타난다. 공통 필드는 작업을 식별할 뿐 처리 방법을
정의하지 않는다. 처리 방법은 활성 확장이 정의한다.

이 섹션은 공통 액션 형태와, 이를 채택하는 모든 응답이 공유하는 불변 조건을 정의한다.
이 형태는 재사용할 수 있지만, 기능은 그 명세가 이를 명시적으로 채택하고 부모별 동작을
정의할 때에만 액션을 지원한다. 부모별 동작이란 액션이 어디에 나타나는지, 각 액션
유형이 어떤 효과를 보류하는지, 메시지가 어떻게 적용되는지, 이후 응답이 처리 결과를
어떻게 반영하는지를 말한다. 스키마 조합만으로는 지원이 성립하지 않는다. 장바구니,
체크아웃, 카탈로그가 이 형태를 채택한다. 각각의 부모별 계약은
[장바구니 — 액션](../shopping/cart/index.md#actions),
[체크아웃 — 액션](../shopping/checkout/index.md#actions),
[카탈로그 — 액션](../shopping/catalog/index.md#actions)을 참고한다.

액션과 메시지는 역할이 다르다. 액션은 미처리 작업을 나타내며, 식별자와 확장이 소유하는
처리 설정을 담는다. 메시지는 현재 응답에 대한 설명이나 진단 맥락을 전달하며, RFC 9535
`path`로 정확한 액션 항목을 가리킬 수 있다. 메시지에 `path`가 있으면 비즈니스는 반드시
이를 해당 UCP 응답 객체의 루트를 기준으로 하는 RFC 9535 JSONPath 표현식으로 만들어야
한다(**MUST**).
메시지는 액션의 처리 방법을 정의하거나 결과를 결정하지 않으며, 액션과 메시지 어느 쪽도
다른 쪽을 필요로 하지 않는다.

예를 들어 비즈니스는 미처리 액션 하나를 설명 메시지와 함께 보여 줄 수 있다(예시용
부분 조각).

<!-- ucp:example skip reason="illustrative fragment" -->
```json
{
  "actions": {
    "com.example.identity.student_verification": [
      {
        "id": "verify-student-1",
        "config": {
          "verification_url": "https://business.example.com/verify/abc"
        }
      }
    ]
  },
  "messages": [
    {
      "type": "info",
      "code": "eligibility_accepted",
      "content": "Student discount applied provisionally. Verify your status.",
      "path": "$.actions['com.example.identity.student_verification'][0]"
    }
  ]
}
```

액션은 미처리 작업을 식별하고 `config` 아래에 확장이 소유하는 처리 설정을 담는다.
메시지의 `path`는 자신이 설명하는 정확한 액션 항목을 선택한다.
[체크아웃 자격 확인 예시](../shopping/checkout/index.md#eligibility-verification-at-completion)는
이 패턴을 조합해 완전한 학생 인증 흐름을 보여 준다.

액션을 채택한 기능이 새로 처리한 성공 응답에서, 비즈니스는 반드시 모든 미처리 액션을
포함해야 하며(**MUST**), 미처리 액션이 없으면 반드시 `actions`를 생략해야
한다(**MUST**).

장바구니와 체크아웃은 요청 멱등성을 별도로 정의한다. 중복 요청은 그 기존 규칙을
따르며, `actions`를 포함한 원래의 캐시된 응답을 반환할 수 있다
([메시지 서명 — 재전송 방지](../signatures.md#replay-protection) 참고).

액션의 보류와 오퍼레이션별 결과는 서로 직교한다. 부모 상태도, 메시지의 유형이나
심각도도 액션이 자신이 정의한 효과를 보류하는지를 결정하지 않는다. 메시지는 응답을
설명하거나 특정 요청 효과의 결과를 보고한다. 비즈니스는 오퍼레이션 실패를 보고하지
않으면서 현재 응답을 설명하기 위해, `path`로 미처리 액션을 선택하는 info 또는 warning
메시지를 포함할 수 있다(**MAY**). 상태를 바꾸는 오퍼레이션의 요청 효과가 액션 때문에
적용되지 않았다면, 비즈니스는 대신 현재 리소스와 함께 `path`로 정확한 액션 항목을
선택하는 `recoverable` 오류 메시지를 반드시 반환해야 한다(**MUST**).

오퍼레이션 이후의 상태에 대해서는 비즈니스의 응답이 권위를 가진다. 반환된 리소스가,
해당 기능이 정의하는 부모 수명 주기와 함께 진실의 원천이다. 비즈니스가 처리를 어떻게
관찰하는지는 액션 유형 계약이 정의하며, 플랫폼은 그런 다음 해당 기능의 오퍼레이션
계약을 따른다.

액션 때문에 장바구니나 체크아웃 오퍼레이션이 성공하지 못한 경우, 액션을 처리한다고
해서 그 오퍼레이션이 반복되지는 않는다. 플랫폼이 다시 시도하려면 기존
[재전송 방지](../signatures.md#replay-protection) 규칙에 따라 새 오퍼레이션을
제출한다.

각 액션 키는 역도메인 **액션 유형**이다. 이 이름은 미처리 작업의 유형을 식별하며,
반드시 그것을 선언하는 확장의 이름과 같지는 않다. 활성 확장은 각 액션 유형을 선언하고,
그 `config`, 플랫폼의 처리 방법, 신뢰와 대체 동작, 결과를 정의한다. 하나의 확장이
여러 액션 유형을 선언할 수 있다. 선언하는 각 확장은 `allOf` 조합을 통해 자신의 액션
유형 키를 해당 기능의 스키마에 보탠다([스키마 조합](#schema-composition) 참고). 어떤
확장이 활성화될지는 기능 협상이 선택한다. 확장을 협상하면 그 확장이 선언하는 계약
전체, 즉 그 안의 모든 액션 유형이 활성화된다.

액션 유형 키는 기존 [네임스페이스 거버넌스](#namespace-governance) 규칙을 따른다.
확장은 자신의 스키마 권한 주체가 통제하는 역도메인 네임스페이스 안에서만 유형을
선언할 수 있다. 확장은
[학생 인증 예시](../shopping/checkout/index.md#eligibility-verification-at-completion)처럼
자신의 이름을 단일 액션 유형의 키로 쓸 수도 있고, 서로 다른 키로 여러 액션 유형을
선언할 수도 있다. 각 값은 그 액션 유형 하나의 미처리 인스턴스로 이루어진 비어 있지
않은 배열이다. 키가 유형을 식별하므로 인스턴스에는 별도의 유형 판별자가 없다.
비즈니스는 같은 유형의 미처리 인스턴스가 여럿이면 그 배열에 여러 항목으로 담는다.

`actions` 맵은 액션 유형 간 처리 순서를 정의하지 않는다. 한 유형의 배열 안에서는
JSON이 인스턴스 순서를 보존하며, 그 순서가 처리상 의미를 갖는지는 해당 유형을 선언한
확장이 정의한다. 액션 유형 간 순서가 중요할 때는, 선언하는 확장이 순서와 각 단계에서
어떤 액션 유형이 미처리 상태가 되는지를 정의한다.

예를 들어(설명용일 뿐이다), 협상된 벤더 확장 `com.example.payment.authentication`이
두 액션 유형을 선언한다고 하자. 하나는 보이지 않게 기기·브라우저 데이터를 수집하는
단계인 `com.example.payment.authentication.device_data_collection`이고, 다른 하나는
구매자가 직접 거치는 인증 단계인
`com.example.payment.authentication.three_ds_challenge`다. 수집 단계가 챌린지보다
먼저이므로, 비즈니스는 먼저 `device_data_collection` 유형을 내보내고, 그 인스턴스가
처리되면 이후 응답에서 `three_ds_challenge` 유형을 내보낼 수 있다. 이 예시는 확장
하나가 여러 액션 유형을 선언하고 응답에 걸쳐 순서를 정하는 모습을 보여 줄 뿐, 기기
데이터 수집이나 인증 챌린지를 표준화하지는 않는다. 여기서 그것들은 설명용이다.

모든 인스턴스는 공통 필드 집합을 공유한다.

- `id` — 액션 인스턴스를 위한 비어 있지 않은 식별자.
- `config` — 확장이 소유하는 선택적 설정 객체.

`id`는 모든 인스턴스에 필수이고, `config`는 선택 사항이다. 확장은 플랫폼이 작업을
처리하는 데 필요한 인스턴스별 데이터를 `config` 아래에 정의한다. `config`는 그
데이터를 위한, 확장이 소유하는 채널이다.

액션 인스턴스는 전방 호환성을 위해 추가 최상위 필드에도 열려 있다. 플랫폼은 인식하지
못하는 액션 인스턴스 필드를 반드시 허용하고 무시해야 한다(**MUST**).

비즈니스는 한 응답 안의 각 액션 인스턴스에 서로 다른 `id`를 반드시 사용해야
한다(**MUST**).

연속된 응답이 같은 부모 리소스를 나타낼 때, 같은 작업이 미처리 상태로 남아 있는 동안
비즈니스는 반드시 같은 액션 유형 키와 `id`를 유지해야 한다(**MUST**). 대체 작업은
반드시 새 `id`를 가져야 하며(**MUST**), 비즈니스는 해당 리소스의 수명 동안 `id`를
재사용해서는 안 된다(**MUST NOT**).

그 밖의 경우, 공통 액션 계약은 독립된 응답에 있는 액션 사이의 정체성 관계를 정의하지
않는다. `id` 값이 같다는 것만으로는 같은 작업임을 식별할 수 없다.

비즈니스는 액션 유형을 선언하는 확장이 협상된 교집합에서 해당 기능에 대해 활성화된
경우에만 그 액션 유형을 내보내야 한다(**MUST**). 조합된 JSON Schema로 공통 필드와 선언된
각 유형의 키 및 `config` 형태를 검증할 수는 있지만, 선언하는 확장이 활성화되었는지
확인하려면 협상된 기능 맥락도 필요하다.

### 신뢰와 실행 경계 { #trust-and-execution-boundaries }

확장을 협상하면 런타임 전에 그 확장의 액션 유형 계약 전체를 지원한다는 것이
확인된다. 그렇다고 앞으로의 모든 런타임 값이나 위임 대상이 신뢰받는 것은 아니다. 각
인스턴스는 여전히 조합된 스키마, 액션 유형 계약, 플랫폼 정책의 적용을 받는다.

플랫폼이 어떤 `config` 필드를 처리하고 그것이 무엇을 의미하는지는 활성 액션 유형
계약이 정의한다. 플랫폼은 그 밖의 필드를 콘텐츠 로드, HTML 렌더링, 코드 실행, 셸
명령 실행, 네이티브 API 호출에 대한 지시로 취급해서는 안 된다(**MUST NOT**).

플랫폼은 추가 신뢰 정책이나 런타임 정책을 적용할 수 있으며(**MAY**), 이를 충족하지
않는 인스턴스를 거절할 수 있다(**MAY**). 확장 전체를 지원한다고 해서 플랫폼이 모든
런타임 값을 받아들여야 하는 것은 아니다.

플랫폼은 액션 화면이나 외부 상호작용이 완료되었다는 이유만으로 액션이 보류하던 효과가
성공했다고 가정해서는 안 된다(**MUST NOT**). 그 결과에 대해서는 이후 비즈니스의 응답이,
해당 기능이 정의하는 부모 수명 주기와 함께 여전히 권위를 가진다.

구체적인 신뢰, 실행, 대체 동작 규칙은 선언하는 확장이 정의한다. 공통 액션 계약은
범용 메커니즘을 정의하지 않는다. URL 스킴, 오리진, 위임 정책도, 샌드박스, 권한, 표시
모델도, 타임아웃, 실패, 복구 모델도, 콜백, 결과, 상태, 폴링, 실행기도 정의하지 않는다.
각 구체적인 액션 유형은 자신의 처리에 필요한 메커니즘만 추가한다.

## 정책 { #policies }

정책은 구매 시점에 응답 안의 품목에 적용되는 비즈니스 규칙(반품 조건, 보증, 구독
조건 등)이며, `messages[]`, `links[]`와 나란히 핵심 `policies[]` 배열에 담긴다.

### 정책 유형 { #policy-types }

비즈니스는 잘 알려진 정책과 커스텀 정책을 게시한다. 모든 정책은 열린 역DNS 어휘에서
가져온 `type`을 가진다.

| 잘 알려진 유형 | 설명 |
| :-- | :-- |
| `dev.ucp.shopping.policy.return` | 반품 조건. |
| `dev.ucp.shopping.policy.warranty` | 보증 조건. |
| `dev.ucp.lodging.policy.cancellation` | 숙박 예약의 취소 조건. |

비즈니스는 자신의 도메인에서 커스텀 유형(예: `com.example.policy.price_match`)을
정의할 수 있으며(**MAY**), 해당 `type`을 모델링하는 플랫폼이 구조화된 맥락으로 읽을 수
있는 유형별 필드를 추가할 수 있다(**MAY**). 어휘가 열려 있으므로 플랫폼은 알 수 없는
`type` 값을 반드시 허용해야 하며(**MUST**), 그 정책은 `description`으로 표시한다
([정책 표시](#presenting-policies) 참고).

### 대상 지정 { #targeting }

`applies_to`는 RFC 9535 JSONPath 표현식의 배열이며, **이를 담은 응답의 루트**를
기준으로 평가된다. `messages[].path`와 같은 규칙이다. 루트는 화면마다 다르다.
장바구니, 체크아웃, 주문에서는 `$.line_items[N]`, 카탈로그 검색과 조회에서는
`$.products[N]`, get_product에서는 `$.product`다. 정책은 다음 세 가지 형태 중 하나로
노드를 대상으로 지정한다.

- **단일 쿼리(Singular query)** — 이름 선택자와 인덱스 선택자만 사용해 정확히 한
  노드를 가리키는 표현식([RFC 9535
  §2.3.5.1](https://www.rfc-editor.org/rfc/rfc9535#section-2.3.5.1)). 예:
  `$.line_items[2]`.
- **집합 매칭(Set match)** — 노드 집합과 매칭되는 필터, 와일드카드, 슬라이스. 예:
  `$.products[?@.category=='electronics']`.
- **응답 전체(Response-wide)** — `applies_to`를 생략한 경우로, 정책이 응답 전체에
  적용된다. 가장 흔한 경우다. 사이트 전체에 적용되는 정책은 대상 지정 없는 항목
  하나이며, 품목마다 반복하지 않는다.

대상은 자신이 가리키는 노드 **및 그 아래에 중첩된 모든 것**을 포괄한다.
`$.products[0]`에 대한 정책은 상품과 모든 변형을 포괄하지만,
`$.products[0].variants[3]`에 대한 정책은 그 변형만 포괄한다. 한 변형에 다른 조건을
적용하려면 비즈니스는 그 변형을 직접 지정한다. 둘이 겹치는 곳에서는 더 좁은 대상이
우선한다(아래의 우선순위 참고).

### 우선순위 { #precedence }

`type`이 **다른** 정책은 서로 독립적이다. 각각 따로 적용되므로, 한 노드에 보증 정책과
가격 맞춤 정책이 동시에 걸릴 수 있다.

`type`이 **같은** 정책은 한 노드를 두고 경합할 수 있다. 그런 경우 정확히 하나가 적용되어
나머지를 **대체한다**. 플랫폼은 그 본문들을 병합해서는 안 된다(**MUST NOT**). 병합하려면
정책들이 결합되는지 대체되는지를 추론해야 하는데, 플랫폼은 이를 데이터에서 읽어낼 수
없다. 해석은 구조만으로 적용할 정책 하나를 선택한다. 조건이 실제로 누적된다면
비즈니스가 그 조건들을 가장 구체적인 정책 하나에 합쳐 둔다. 즉 조합은 작성 단계에서
이루어지며 해석 단계에서 이루어지지 않는다. 어느 정책이 적용될지는 현재 응답에 대한
**최장 접두사 매칭**으로 해석한다.

모든 노드에는 정식 정체성이 있다. 바로 루트에서 그 노드까지의 세그먼트 시퀀스인
**Normalized Path**([RFC 9535
§2.7](https://www.rfc-editor.org/rfc/rfc9535#section-2.7))다.
`$.products[0].variants[3]`은 `products`, `0`, `variants`, `3`이고, 루트 `$`는 빈
시퀀스다. 대상이 매칭하는 노드가 어떤 노드 자신이거나 그 조상이면, 대상은 그 노드를
**포괄한다**. 같은 말로, 매칭된 노드의 Normalized Path가 대상 노드의 Normalized Path의
접두사이면 포괄한다. 그 접두사의 길이가 **깊이**(depth)다.

주어진 `type`의 정책 중 어느 것이 노드에 적용되는지는 다음과 같이 해석한다.

1. 대상이 그 노드를 포괄하는, `type`이 같은 정책들을 모은다. 응답 전체 정책
   (`applies_to` 생략)은 루트 `$`를 대상으로 하므로 깊이 0에서 모든 노드를 포괄한다.
   그 노드를 포괄하는 정책이 없으면 그 `type`의 어떤 정책도 적용되지 않는다.
2. 각 정책에 **가장 깊은** 포괄 매칭을 기준으로 `(depth,
   precision)` 쌍의 점수를 매긴다. *depth*는 그 매칭의 접두사 길이다. *precision*은
   대상이 **단일 쿼리**([RFC 9535
   §2.3.5.1](https://www.rfc-editor.org/rfc/rfc9535#section-2.3.5.1) — 이름 선택자와
   인덱스 선택자만 사용하므로 노드 하나를 가리킴)이면 `1`, **집합 매칭**(필터,
   와일드카드, 슬라이스)이면 `0`이다. 가장 큰 깊이를 공유하는 포괄 매칭이 여럿이면
   그중 가장 큰 precision을 택한다.
3. 점수가 가장 큰 정책이 적용된다. **깊이가 먼저이고, 그다음이
   precision이다**.
4. 가장 큰 점수가 공유되면 결과는 **정의되지 않는다**. 동점이 되려면 두 정책이 같은
   깊이와 precision으로 노드를 포괄해야 한다. 노드에는 깊이마다 조상이 하나씩만
   있으므로, 이들은 *같은* 노드를 두고 경합하는 것이다. 겹치는 집합 매칭 두 개나,
   한 `type`의 응답 전체 항목 두 개가 그런 경우다. 이는 경로 대상 지정의 부작용이
   아니라 작성 오류다. 노드를 id로 지정했더라도 똑같이 충돌했을 것이다. 비즈니스는
   그런 충돌을 게시해서는 안 된다(**MUST NOT**). 해석이 정의되지 않으므로 플랫폼은
   조용히 해석하지 말고 모호함을 표시해야 한다(**SHOULD**). 처리 방식은 플랫폼에
   맡긴다.

### 부재와 빈 배열 { #absent-vs-empty }

특정 응답에서 `policies[]`가 없거나 비어 있으면, 플랫폼은 `links[]`의 일반 정책
리소스(예: `refund_policy`, 또는 카탈로그의 변형별 `seller.links`)를 참조해야
한다(**SHOULD**).

### 정책 표시 { #presenting-policies }

정책은 품목에 적용되는 비즈니스 규칙을 기술한다. 플랫폼은 자체 판단(자격 여부, 계산된
반품 기한 등)을 위해 정책을 근거로 추론할 수 있다(**MAY**). 정책의 `type`을 모델링하지
않는 플랫폼도 언제나 정책을 표시할 수 있도록, 비즈니스는 반드시 `description`을
제공해야 한다(**MUST**). 이는 플랫폼이 구매자에게 보여 줄 수 있는(**MAY**), 사람이 읽는
요약이다. 정책 표시는 선택 사항이다.

비즈니스가 정책을 구매자에게 반드시 보여 주도록 **요구하는** 경우(최종 판매 품목,
규제 고지 등), 비즈니스는 다음 조건을 충족하는 `messages[]` warning을 반드시 내보내야
한다(**MUST**).

- `presentation: "disclosure"`를 설정해, 플랫폼이 그 내용을 표시하고 숨기거나 닫을 수
  없게 한다([경고 표시](../shopping/checkout/index.md#warning-presentation) 참고).
- `path`를 고지 대상 품목으로 설정한다.
- `code`를 정책의 `type`으로 설정해 고지를 해당 정책과 연결한다.

이 warning은 유형에 구애받지 않는다. 플랫폼은 그 뒤에 있는 정책을 이해하지 않고도
내용을 표시하므로, 최종 판매 조건부터 규제 고지까지 채널 하나로 모두 처리할 수 있다.

고지(disclosure)는 자신의 `path` 노드에 **적용되는** 정책, 즉 같은 `type`의 정책 여럿이
그 노드를 포괄할 때 [우선순위](#precedence)가 선택하는 정책과 짝을 이룬다. 우선순위는
최대 하나만 산출하므로 짝짓기에 모호함이 없다. 다음 두 규칙이 적용된다.

1. 고지의 내용은 반드시 짝을 이루는 정책과 일치해야 한다(**MUST**). 고지와 정책은 같은
   노드에 대한 두 가지 진술이다.
2. 고지는 적용되는 정책으로 해석되어야 한다(**SHOULD**). `code`가 그 노드를 포괄하는
   정책이 없는 `type`을 가리키면 고지는 여전히 표시되지만, 그 뒤에 구조화된 근거가
   없다.

예를 들어 각인된 라인 아이템은 최종 판매 품목이다. 응답 전체 반품 정책이 장바구니
전체에 적용되지만, 품목 범위의 최종 판매 정책이 2번 라인에 적용되므로, 그 라인의
고지는 최종 판매 정책과 짝을 이룬다. 고지와 정책이 일치한다.

<!-- ucp:example schema=shopping/checkout target=$.policies -->
```json
[
  {
    "type": "dev.ucp.shopping.policy.return",
    "description": { "plain": "Free 30-day returns from delivery." }
  },
  {
    "type": "dev.ucp.shopping.policy.return",
    "description": { "plain": "This engraved item is final sale and cannot be returned." },
    "applies_to": ["$.line_items[2]"],
    "url": "https://example.com/returns#final-sale"
  }
]
```

<!-- ucp:example schema=shopping/checkout target=$.messages -->
```json
[
  {
    "type": "warning",
    "code": "dev.ucp.shopping.policy.return",
    "path": "$.line_items[2]",
    "presentation": "disclosure",
    "content": "This engraved item is final sale and cannot be returned."
  }
]
```

### `links[]`와의 관계 { #relationship-to-links }

`links[]`와 `policies[]`는 상호 보완적이다. `links[]`는 항상 존재하는 대체 수단으로,
응답 전체에 적용되는 레이블이 붙은 URL이며 보통 유형마다 하나다. `policies[]`는 사용
가능할 때의 구조화된 계층으로, 유형이 있고, 선택적으로 품목별 `applies_to` 대상
지정을 하며, 여러 항목(응답 전체 기본값과 그에 대한 재정의)을 가질 수 있다.

### 예시 { #examples_1 }

사이트 전체 보증과 품목별 재정의로, 둘 다 같은 `type`이다. 라인 아이템 2에는 단일
쿼리가 적용되어 응답 전체 기본값을 재정의하고, 나머지 라인 아이템은 모두 기본값을
유지한다.

<!-- ucp:example schema=shopping/checkout target=$.policies -->
```json
[
  {
    "type": "dev.ucp.shopping.policy.warranty",
    "description": { "plain": "1-year limited warranty on all items." }
  },
  {
    "type": "dev.ucp.shopping.policy.warranty",
    "description": { "plain": "3-year extended warranty on this item." },
    "applies_to": ["$.line_items[2]"]
  }
]
```

집합 매칭을 단일 쿼리가 재정의하는 경우로, 역시 같은 `type`이다. 상품 0은 전자 제품이므로
두 항목 모두 여기에 닿는다. 그곳에서는 단일 쿼리가 적용되고, 다른 전자 제품은 2년
조건을 유지한다.

<!-- ucp:example schema=shopping/catalog_search op=search direction=response target=$.policies -->
```json
[
  {
    "type": "dev.ucp.shopping.policy.warranty",
    "description": { "plain": "2-year warranty on electronics." },
    "applies_to": ["$.products[?@.category=='electronics']"]
  },
  {
    "type": "dev.ucp.shopping.policy.warranty",
    "description": { "plain": "5-year manufacturer-certified warranty on this item." },
    "applies_to": ["$.products[0]"]
  }
]
```

플랫폼이 모델링하지 않는 커스텀 `type`도 `description`으로 여전히 표시할 수 있다.

<!-- ucp:example schema=shopping/catalog_search op=search direction=response target=$.policies -->
```json
[
  {
    "type": "com.example.policy.price_match",
    "description": { "plain": "We match a competitor's lower price for 14 days after purchase." },
    "applies_to": ["$.products[0]"]
  }
]
```

## 보안 { #security }

### 전송 보안 { #transport-security }

모든 UCP 통신은 반드시 **HTTPS**로 이루어져야 한다(**MUST**).

### 데이터 프라이버시 { #data-privacy }

민감한 데이터(결제 자격 증명이나 개인 식별 정보(PII) 등)는 반드시 PCI-DSS와 GDPR
지침에 따라 다루어야 한다(**MUST**). UCP는 비즈니스와 플랫폼의 책임을 최소화하기 위해
토큰화된 결제 데이터 사용을 권장한다.

### 시그널 { #signals }

비즈니스는 승인, 요청 속도 제한, 남용 방지를 위해 환경 데이터가 필요하다. 시그널 값은
구매자가 스스로 주장한 내용이어서는 안 된다(**MUST NOT**). 플랫폼은 직접 관찰한
결과(예: 연결 IP, 사용자 에이전트)를 바탕으로, 또는 독립적으로 검증 가능한 서드파티
증명을 중계하는 방식으로 시그널을 제공한다. 서드파티 증명의 예로는 비즈니스가 제공자의
게시된 키 세트로 검증할 수 있는, 외부 검증자의 암호학적으로 서명된 결과가 있다.

모든 시그널 키는 출처를 보장하고 여러 확장이 공유 네임스페이스에 기여할 때 충돌을 막기
위해 반드시 역도메인 명명 방식을 사용해야 한다(**MUST**). 잘 알려진 시그널은 `dev.ucp`
네임스페이스를 사용하고(예: `dev.ucp.buyer_ip`), 확장 시그널은 자체 네임스페이스를
사용한다(예: `com.example.device_id`).

<!-- ucp:example schema=shopping/checkout op=create direction=request target=$.signals -->
```json
{
  "dev.ucp.buyer_ip": "203.0.113.42",
  "dev.ucp.user_agent": "Mozilla/5.0 ...",
  "com.example.attestation": {
    "provider_jwks": "https://example.com/.well-known/jwks.json",
    "kid": "example-key-2026-01",
    "payload": { "id": "att-7c3e9f", "pass": true, "...": "..." },
    "sig": "base64url..."
  }
}
```

시그널 필드에는 개인 식별 정보(PII)가 담길 수 있다. 플랫폼은 현재 트랜잭션과 관련된
시그널만 포함해야 한다(**SHOULD**). 비즈니스는 트랜잭션의 운영상 필요(예: 주문 확정,
사기 검토)를 넘어 시그널 데이터를 보관하지 않아야 한다(**SHOULD NOT**).

비즈니스는 코드가 `signal`인 메시지를 사용해 추가 데이터를 요청할 수 있다(**MAY**).
`path` 필드는 요청하는 시그널을 식별하고, 메시지 `type`이 강제 수준을 결정한다.
`error`는 시그널이 제공될 때까지 상태 진행을 막고, `info`는 권고일 뿐 진행을 막지
않는다.

<!-- ucp:example schema=shopping/checkout target=$.messages -->
```json
[
  {
    "type": "error",
    "code": "signal",
    "path": "$.signals['dev.ucp.buyer_ip']",
    "content": "Buyer IP is required to proceed.",
    "severity": "recoverable"
  },
  {
    "type": "info",
    "code": "signal",
    "path": "$.signals['dev.ucp.user_agent']",
    "content": "Providing user agent may improve checkout outcomes."
  }
]
```

### 어트리뷰션 { #attribution }

플랫폼은 유료 광고, 자연 추천, 인플루언서 링크, AI 에이전트 등 다양한 채널을 통해
사용자를 비즈니스로 유입시킨다. 브라우저 기반 흐름에서는 유입 맥락(캠페인, 클릭
식별자, 소스/매체 표시)이 URL 쿼리 파라미터로 전달된다. `attribution` 필드는 플랫폼이
같은 파라미터를 비즈니스에 전달할 수 있게 한다.

UCP는 어트리뷰션 모델, 기간, 할당 로직을 규정하지 **않는다**(**NOT**). 플랫폼은 기존
관례(GA4 캠페인 파라미터, `gclid` / `fbclid` / `ttclid` 같은 클릭 식별자 등)를 사용하고,
비즈니스는 자체 분석 요구에 따라 이를 받아 처리한다.

<!-- ucp:example schema=shopping/checkout op=create direction=request target=$.attribution -->
```json
{
  "campaign_id": "18234567890",
  "campaign_source": "google",
  "campaign_medium": "cpc",
  "campaign_name": "spring_2026",
  "gclid": "EAIaIQobChMI..."
}
```

어트리뷰션은 정보 제공용이며, 플랫폼이 선택적으로 제공한다.
비즈니스는 지원 여부를 협상하거나 광고하지 않는다. 이 필드의 존재 여부가 응답이나
협상에 영향을 주어서는 안 된다(MUST NOT).

이 데이터에는 관련 데이터 보호법상 개인 데이터로 취급되는 가명 식별자(클릭 ID, 세션
키)가 담길 수 있다. 플랫폼과 비즈니스는 각자의 관할권에서 규정 준수에 대한 책임을
진다. 플랫폼은 무엇을 내보내고 고지할지 결정하고, 비즈니스는 자체 데이터 처리, 보관,
동의 정책을 적용한다. `buyer_consent` 확장은 구매자가 동의 상태를 전달할 수 있는
구조화된 채널을 제공한다.

어트리뷰션은 장바구니, 체크아웃, 카탈로그 요청에서는 플랫폼이 제공하는 어트리뷰션
맥락으로 나타나고, 주문에서는 원래 체크아웃의 어트리뷰션을 비즈니스가 스냅숏으로 내보낸
형태로 나타난다.

### 트랜잭션 무결성과 부인 방지 { #transaction-integrity-and-non-repudiation }

승인에 대한 암호학적 증명이 필요한 시나리오(예: 자율 에이전트, 고액 트랜잭션)를 위해
UCP는 **AP2 위임 확장**(`dev.ucp.common.payment.ap2_mandate`)을 지원한다. 이 선택적
확장이 협상되면:

- 비즈니스는 체크아웃 조건에 대한 암호학적 서명을 제공한다
- 플랫폼은 사용자 승인을 증명하는 암호학적 위임을 제공한다

이 메커니즘은 트랜잭션 세부 사항과 참여자 동의에 대해 강력한 종단 간 암호학적 보증을
제공하므로, 변조와 분쟁의 위험을 크게 줄인다.

전체 명세, 구현 가이드, 예시는 [AP2 위임 확장](../payment/extensions/ap2-mandates.md)을
참고한다.

## 버전 관리 { #versioning }

### 버전 형식 { #version-format }

UCP는 `YYYY-MM-DD` 형식의 날짜 기반 버전 관리를 사용한다. 이 방식은 시간 순서가
명확하고 버전 비교에 모호함이 없다.

### 버전별 프로필 { #versioned-profiles }

UCP는 강력한 하위 호환성을 우선한다. 각 프로필 문서는 `ucp.version`에 코어 릴리스 버전
하나를 선언하며, 그 서비스, 기능, 확장, 결제 핸들러도 명시적인 버전을 가진다.
비즈니스는 현재 프로필을 광고하고, `supported_versions`를 통해 지원하는 이전 릴리스의
프로필로 연결되는 링크를 제공한다. [프로토콜 버전](#protocol-version)을 참고한다.

#### 예시 { #example }

=== "비즈니스 프로필"

    <!-- ucp:example schema=profile def=business_schema -->
    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "services": { ... },
        "capabilities": { ... },
        "payment_handlers": { ... }
      }
    }
    ```

=== "플랫폼 프로필"

    <!-- ucp:example schema=profile def=platform_schema -->
    ```json
    {
      "ucp": {
        "version": "{{ ucp_version }}",
        "services": { ... },
        "capabilities": { ... },
        "payment_handlers": { ... }
      }
    }
    ```

### 버전 선택 { #version-selection }

![상위 수준 해석 흐름 시퀀스 다이어그램](site:specification/images/ucp-discovery-negotiation.png)

프로토콜 버전 선택은 먼저 완전한 비즈니스 프로필 하나를 고른다. 그런 다음 기능 협상이
그 프로필 안에서 정확한 버전 교집합으로 활성 기능과 확장을 선택한다. 아래 섹션에서
각 단계를 정의한다. 릴리스 정렬과 버전 소유권은
[컴포넌트 버전 관리와 릴리스 스냅숏](#component-versioning-and-release-snapshots)을
참고한다.

#### 프로토콜 버전 { #protocol-version }

`version` 필드는 비즈니스의 현재 프로토콜 버전을 선언한다. `/.well-known/ucp`의
프로필은 그 버전에서 사용할 수 있는 기능, 서비스, 결제 핸들러를 기술한다.

이전 프로토콜 버전을 지원하는 비즈니스는 각 이전 버전을 프로필 URI에 매핑하는
`supported_versions` 객체를 선언해야 한다(**SHOULD**). 각 URI는 해당 버전에 대한
완전하고 자기 완결적인 프로필을 가리킨다. 그 프로필의 `ucp.version`은 맵 키와 같으며,
자체 기능, 서비스, 결제 핸들러, 서명 키를 포함한다. `supported_versions`를 생략하면
`version`만 지원한다.

<!-- ucp:example schema=profile def=business_schema -->
```json
{
  "ucp": {
    "version": "2026-01-23",
    "supported_versions": {
      "2026-01-11": "https://business.example.com/.well-known/ucp/2026-01-11"
    },
    "services": {},
    "payment_handlers": {}
  }
}
```

##### 초기 서비스 및 기능 디스커버리 { #initial-service-and-capability-discovery }

플랫폼은 다음 흐름으로 비즈니스의 기능을 디스커버리한다.

1. 플랫폼이 `/.well-known/ucp`를 가져온다. 이것이 현재 버전의 프로필이다.
2. 플랫폼은 비즈니스의 `version`과 `supported_versions` 키 중에서 양측이 지원하는
    프로토콜 버전을 선택한다. 플랫폼은 가장 최신 버전을 우선해야 한다(**SHOULD**).
    선택한 버전이 `version`과 일치하면 플랫폼은 이 프로필을 그대로 사용해 기능 협상으로
    진행한다.
3. 선택한 버전이 `supported_versions`의 키라면, 매핑된 URI에서 프로필을 가져온다.
    플랫폼은 가져온 프로필의 `ucp.version`이 자신이 선택한 `supported_versions` 키와
    같은지 반드시 검증해야 하며(**MUST**), 일치하지 않으면 그 프로필을 사용해서는 안
    된다(**MUST NOT**). 일치하면 이 프로필이 해당 프로토콜 버전에서 사용할 수 있는
    기능을 기술하므로, 기능 협상으로 진행한다.
4. 양측이 지원하는 버전이 없으면 비즈니스가 플랫폼의 프로토콜 버전을 지원하지 않는
    것이다. 플랫폼은 호환되지 않는 버전으로 요청을 보내지 않아야 하며(**SHOULD NOT**),
    비즈니스는 반드시 `version_unsupported` 오류로 응답해야 한다(**MUST**).

버전별 프로필은 말단(leaf) 문서다. 정확히 하나의 프로토콜 버전만 기술하며,
`supported_versions` 필드를 포함해서는 안 된다(**MUST NOT**).

선택된 프로필은 선언된 버전에서 선언된 기능들 사이의 호환성을 보장한다. 버전이 다르면
리소스 표현이 다를 수 있다. 플랫폼은 같은 비즈니스와 서로 다른 지원 버전으로 별도의
협상을 진행할 수 있으며(**MAY**), 한 협상 안에서 여러 프로필의 기능을 합치거나 교차시키지
않아야 한다(**SHOULD NOT**).

##### 요청 시점 검증 { #request-time-validation }

비즈니스는 모든 요청에서 플랫폼의 프로토콜 버전을 반드시 검증해야 한다(**MUST**).

1. 플랫폼은 요청에서 참조하는 프로필의 `version` 필드로 자신이 사용하는 프로토콜
    버전을 선언한다.
2. 비즈니스는 다음과 같이 검증한다.
    - 플랫폼의 `version`이 비즈니스의 `version`과 일치하거나 `supported_versions`의
        키라면, 요청은 비즈니스 프로필의 해당 버전을 사용해 기능 협상으로 진행할 수
        있다(**MAY**).
    - 그렇지 않으면 비즈니스는 반드시 `version_unsupported` 오류를 반환해야
        한다(**MUST**).
3. 요청된 오퍼레이션에 필요한 기능에 대해 기능 협상 결과 양측이 지원하는 버전이 없으면,
    비즈니스는 반드시 `capabilities_incompatible` 오류를 반환해야 한다(**MUST**)
    ([오류 처리](#error-handling) 참고).
4. 비즈니스는 모든 응답에 협상된 프로토콜 버전을 반드시 포함해야 한다(**MUST**).

버전 확인이 담긴 응답:

<!-- ucp:example schema=shopping/checkout extract=$.ucp target=$.ucp -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": { ... },
    "payment_handlers": { ... }
  },
  "id": "checkout_123",
  "status": "incomplete"
  // ... other checkout fields
}
```

지원하지 않는 버전 오류 — 리소스가 생성되지 않는다:

<!-- ucp:example schema=common/types/error_response op=read -->
```json
{
  "ucp": { "version": "{{ ucp_version }}", "status": "error" },
  "messages": [{
    "type": "error",
    "code": "version_unsupported",
    "content": "Version 2026-01-12 is not supported. This business implements version {{ ucp_version }}.",
    "severity": "unrecoverable"
  }],
  "continue_url": "https://merchant.com/"
}
```

##### 사전 릴리스 버전 { #pre-release-versions }

프로토콜 버전은 반드시 `YYYY-MM-DD` 형식의 날짜가 붙은 릴리스여야 한다(**MUST**).
비즈니스는 프로필의 `version` 필드나 `supported_versions`에 날짜가 아닌 버전
문자열(예: `"draft"`)을 광고해서는 안 된다(**MUST NOT**). 사전 릴리스 구현은 안정적이지
않으므로 공개 디스커버리로 노출해서는 안 된다(MUST NOT). 그렇게 하면 생태계 전반이
정의되지 않은 동작과 예고 없는 비호환 변경에 노출된다.

플랫폼과 비즈니스는 공개 디스커버리 밖에서 사전 릴리스 구현을 조율할 수
있다(**MAY**). 이런 사용에는 안정성이나 호환성 보장이 전혀 없다. 기반 동작은 언제든
예고 없이 바뀔 수 있다.

#### 기능 버전 { #capability-versions }

기능 호환성은 날짜 순서로 호환성을 추론하는 것이 아니라, 오직 정확한 버전 일치로만
확립된다. 프로필의 각 기능은 배열이다. 해당 게시 정책이 허용하는 경우, `version` 값이
서로 다른 여러 항목으로 여러 버전 지원을 광고할 수 있다. 정확히 공유되는 집합에 버전이
둘 이상 있으면, 교집합 알고리즘은 공유 날짜들을 정렬해 가장 최신 버전을 선택한다.
지원되는 서드파티 확장 버전은 `ucp.version`과 독립적으로 올라간다. UCP가 작성한
`dev.ucp.*` 항목은 릴리스 `D`에서 버전 `D`를 선언한다. 즉 `ucp.version = D`인 프로필은
이 항목들에 대해 버전 `D`를 광고한다. 이전 `dev.ucp.*` 버전은 `supported_versions` 말단
프로필을 통해서만 광고하며, 더 새로운 프로필에 추가 항목으로 넣지 않는다. 레지스트리
메커니즘은 다중 버전 배열을 지원하지만, 코어 게시 계약은 `dev.ucp.*` 항목에 이를 사용하지
않는다. 따라서 다중 버전 배열은 작성자가 여러 버전을 동시에 지원하는 서드파티 확장에서
생긴다. 기능 교집합 알고리즘은 양측이 모두 지원하는 기능 버전만 고려한다.

비즈니스는 응답에 협상된 프로토콜 버전과 호환되는 기능만 반드시 포함해야
한다(**MUST**). 더 새로운 프로토콜 버전에서 도입된 기능에 의존하는 기능은 이전 프로토콜
버전으로 처리할 때 포함해서는 안 된다(**MUST NOT**).

### 하위 호환성 { #backwards-compatibility }

UCP는 변경 사항을 하위 호환 변경과 호환성을 깨는 변경으로 분류하고, 이 분류를 자체
컴포넌트에 적용해 UCP 릴리스가 어떻게 진행될지 관리한다. 두 분류에 대한 게시 및 백포트
정책은 [컴포넌트 버전 관리와 릴리스 스냅숏](#component-versioning-and-release-snapshots)에
정의되어 있다. 아래 목록은 어떤 변경이 규격을 준수하는 연동을 유지하고 어떤 변경이
깨뜨리는지 설명한다. 서드파티 확장과 결제 핸들러의 작성자는 자체 버전 정책을 관리하며,
같은 구분을 자신의 주기에 맞춰 적용할 수 있다.

#### 하위 호환 변경 { #backwards-compatible-changes }

다음 변경은 **하위 호환된다**. 즉 규격을 준수하는 연동을 깨뜨리지 않는다.

- 응답에 필수가 아닌 새 필드 추가
- 요청에 필수가 아닌 새 파라미터 추가
- 전송 방식에 새 엔드포인트, 메서드, 오퍼레이션 추가
- 기존 오류 구조를 사용하는 새 오류 코드 추가
- enum에 새 값 추가(전체 목록이라고 명시적으로 문서화된 경우 제외)
- 응답의 필드 순서 변경
- 불투명 문자열(ID, 토큰)의 길이나 형식 변경

#### 호환성을 깨는 변경 { #breaking-changes }

다음 변경은 **호환성을 깨는 변경**이다. 규격을 준수하는 연동을 깨뜨리므로 새 컴포넌트
버전이 필요하다.

- 기존 필드 제거 또는 이름 변경
- 필드 타입이나 의미 변경
- 필수가 아닌 필드를 필수로 변경
- 오퍼레이션, 메서드, 엔드포인트 제거
- 인증 또는 권한 부여 요구 사항 변경
- 기존 프로토콜 흐름이나 상태 머신 수정
- 기존 오류 코드의 의미 변경

### 컴포넌트 버전 관리와 릴리스 스냅숏 { #component-versioning-and-release-snapshots }

UCP 릴리스 `D`는 코어 프로토콜, 즉 서비스와 전송 바인딩, 기능, 확장, 공유 스키마의
스냅숏이며, 내부적으로 호환되는 묶음으로 함께 게시되고 인증된다. `ucp.version`으로
`D`를 선택하면 그 스냅숏을 선택하는 것이다. 릴리스 `D`에 대해:

1. `ucp.version`은 `D`다.
2. UCP가 정의한 모든 서비스는 `version` `D`를 선언한다. 각 서비스 항목은 서비스를 하나의
    전송 바인딩과 짝짓므로, 한 서비스 아래의 모든 항목이 그 서비스 `version`을 반복한다.
    전송 바인딩에는 별도의 버전이 없다. 참조되는 OpenAPI/OpenRPC 아티팩트는 릴리스
    메타데이터로서 `D` 아래에 게시된다.
3. UCP가 정의한 모든 기능과 확장은, 자체 스키마가 직접 바뀌지 않았더라도 `version` `D`를
    선언한다.
4. 공유 스키마는 같은 스냅숏의 일부로 게시된다.
5. UCP는 스냅숏(컴포넌트, 바인딩, 지원되는 조합)을 게시하기 전에 함께 인증한다.

UCP는 승인된 하위 호환 변경(기능, 또는 보안·정확성·상호 운용성 수정)을 지원 중인
릴리스와 그로부터 생성된 `D` 아티팩트에 백포트할 수 있다(**MAY**). 호환성을 깨는 변경은
다음 릴리스에 들어가며 기본적으로 백포트하지 않는다. 예외적인 경우 거버넌스 위원회는
호환성을 깨는 변경을 지원 중인 릴리스에 백포트하는 것을 승인할 수 있다(**MAY**). 단,
게시된 릴리스의 보안, 정확성, 상호 운용성을 해치는 결함에 한하며, 호환성을 깨는 변경에
대한 고지 절차를 따라야 한다. UCP는 업데이트된 아티팩트를 게시하기 전에 반드시 스냅숏을
재인증해야 한다(**MUST**).

날짜가 붙은 릴리스는 장기 지원(LTS) 채널처럼 동작한다. 날짜는 하나의 호환성 계열을
가리키며, 백포트된 변경은 게시된 `D` 스냅숏을 그 자리에서 수정한다. `D`는 그대로인 채
내용이 바뀌므로, 서로 다른 시점에 가져온 당사자들은 같은 `D`의 서로 다른 사본을 가질 수
있다. 이런 차이는 구조상 안전하다. 백포트는 기본적으로 하위 호환 분류에 속하므로, 이전
사본은 이후 추가 사항이 빠져 있을 뿐 그것과 충돌하지 않는다. 런타임에 아티팩트를
가져오는 당사자는 표준 HTTP 캐싱 의미론을 따라, `max-age`가 만료되면 캐시된 사본을
재검증해야 한다(**SHOULD**)([호스팅](#hosting) 참고). 따라서 갱신 빈도는 게시자가
통제한다. 빌드 시점에 아티팩트를 사용하는 당사자는 자신의 릴리스 주기에 맞춰 수정 사항을
반영한다. 예외적으로 호환성을 깨는 백포트는 호환성을 깨는 변경에 대한 고지 절차로
공지되며, 요청 시점에 수정된 계약으로 강제된다. 이 백포트가 이전 사본을 무효화하는
이유는 오직 게시된 릴리스에 결함이 있었기 때문이다.

`ucp.version`으로 `D`를 선택하는 비즈니스나 플랫폼은 프로필의 모든 `dev.ucp.*` 서비스,
기능, 확장 항목에 반드시 `version` `D`를 선언해야 한다(**MUST**). `version`이 프로필의
`ucp.version`과 다른 `dev.ucp.*` 항목을 만난 플랫폼은 반드시 그 항목을 거부해야
하며(**MUST**)(존재하지 않는 것으로 취급하고 절대 활성화하지 않는다), 나머지 항목으로
계속 진행할 수 있다(**MAY**). 이전 릴리스는 자체 `ucp.version`이 그 이전 릴리스 날짜인
별도의 `supported_versions` 말단 프로필을 통해서만 선택된다
([프로토콜 버전](#protocol-version) 참고). 결제 핸들러 버전은 작성자가 관리한다. UCP는
공유 선언 구조를 정의하지만 현재 구체적인 핸들러는 정의하지 않으므로, 어떤 핸들러 버전도
`D`로 제약되지 않는다. `version` `D`를 선언한다고 협상 방식이 바뀌지는 않는다.
`dev.ucp.*` 기능과 확장은 여전히 정확한 버전 교집합으로 선택된다
([기능 버전](#capability-versions) 참고).

#### Third-party extensions (`com.{vendor}.*`, `org.{org}.*`)

서드파티 기능은 `ucp.version`과 독립적으로 버전이 매겨진다. 버전과 릴리스 주기 모두
작성자가 관리한다. 일반적인 형태는 UCP가 정의한 하나 이상의 루트 기능 위에 얹는
**확장**이다. 서드파티 확장은:

- UCP가 정의한 하나 이상의 루트 기능에 대해 `extends`를 선언하며, `allOf`로 조합된다
    ([확장 스키마 패턴](#extension-schema-pattern) 참고).
- `ucp.version`이 선택한, UCP가 정의한 서비스와 전송 바인딩을 사용한다.
- 비즈니스 프로필과 플랫폼 프로필에 정확한 확장 버전을 광고하며, 다른 모든 기능과
    마찬가지로 정확한 버전 교집합으로 협상된다.

서드파티 확장의 버전은 절대 `ucp.version`에 묶이지 않는다. 작성자는 확장에 필요한 UCP
릴리스와 기능 버전을 `requires`로 선언한다([버전 요구 사항](#version-requirements)
참고). 이 범위는 프로필이 선택한 버전과의 호환성을 검증할 뿐, 버전을 선택하지는 않는다.

## 용어집 { #glossary }

UCP 명세 전반에서 사용하는 약어와 용어의 정의는 [용어집](../glossary.md)을 참고한다.
