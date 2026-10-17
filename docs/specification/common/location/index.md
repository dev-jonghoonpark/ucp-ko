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

# 위치 기능 { #location-capability }

위치(Location) 기능을 사용하면 플랫폼이 비즈니스의 물리적 위치(소매 매장, 음식점,
브랜드 보관함 등)를 찾고, 검색하고, 조회할 수 있다.

이 기능은 특정 버티컬에 종속되지 않으며 다음과 같은 핵심 커머스 흐름을 가능하게 한다.

* **근처 픽업 장소 탐색**: 구매자 픽업을 지원하는 소매 매장이나 음식점 지점 등
    가까운 위치를 찾고, 영업시간을 확인하고, 상품을 고르기 전에 선택한 상품을
    현재 어디에서 구할 수 있는지 파악한다.
* **풀필먼트 지역 확인**: 특정 위치(예: 유틸리티 거점, 음식점, 지역 서비스 제공자)가
    구매자의 주소에 서비스할 수 있는지 확인한다(`serves` 관계).

## 기능 { #capabilities }

| 기능 | 설명 |
| :--- | :--- |
| [`dev.ucp.common.location.search`](search.md) | 자유 텍스트 질의, 명시적 공간 관계(`distance`, `serves`), 필터(`hours`, `amenities`, `items`)를 사용해 위치를 검색한다. |
| [`dev.ucp.common.location.lookup`](lookup.md) | 식별자로 하나 이상의 위치에 대한 전체 정보를 조회한다. 같은 관계와 필터로 결과를 좁힐 수도 있다. |

## 핵심 개념 { #key-concepts }

* **위치(Location)**: 지도에서 찾을 수 있는 물리적 개체. 표시 이름, 주소, 영업시간,
    **지리적 맥락**(지리 좌표)으로 정의된다.
* **편의시설(Amenities)**: 위치가 갖춘 고정적인 특징, 서비스, 역량.
    응답은 역방향 DNS 형식의 편의시설 식별자를 키로 하는 맵을 사용한다. 이 방식은
    충돌에 강한 이름, 식별자당 하나의 항목, 그리고 사용자 정의 편의시설도 화면에
    표시할 수 있게 해 주는 구매자용 설명을 함께 제공한다.
    [편의시설 어휘](search.md#amenity-vocabulary)를 참고한다.
* **상품 재고 여부(Item Availability)**: `items` 필터는 후보 위치에서 비즈니스가
    참조된 모든 상품을 현재 제공할 수 있는지를 묻는다. 위치가 반환되었다는 것은
    대략적이고 잠정적인 확인일 뿐, 상품 단위의 세부 정보나 예약을 의미하지 않는다.
    [상품 재고 필터](search.md#item-availability-filter)를 참고한다.
* **근접성과 서비스 가능 여부**: 서로 구별되는 두 가지 명시적 공간 관계가 있다.
    * **`distance`**: 위치의 좌표를 플랫폼이 제공한 중심점 및 경계를 포함하는 반경과 비교한다.
    * **`serves`**: 플랫폼이 제공한 하나의 명시적 대상에 위치가 잠정적으로 서비스할 수
      있는지를 묻는다. 이 답의 결정 권한은 비즈니스에 있으며, UCP는 그 바탕이 되는 서비스
      범위의 기하 정보를 모델링하거나 노출하지 않는다. 위치가 자신의 좌표 주변에 암묵적인
      기하학적 경계를 갖는 것도 아니다. [공간 관계](search.md#spatial-relations)를 참고한다.
* **영업시간**: 정기 주간 일정(`hours`)과 특정 날짜의 예외(`exception_hours`)로 이루어지며,
    위치의 `timezone`을 기준으로 해석한다. [영업시간](#operating-hours)을 참고한다.

### 다른 기능과의 관계 { #relationship-to-other-capabilities }

위치 기능은 다른 기능(쇼핑의 카탈로그, 장바구니, 체크아웃 등)과 긴밀하게
통합되어 지역 기반 커머스의 토대가 된다.

1. **안정적인 식별자**: 위치 검색과 조회는 물리적 개체를 나타내는, 비즈니스 범위의
    안정적인 `Location.id` 값을 반환한다. 위치 ID는 물리적 개체만 선택할 뿐,
    상호작용 방식을 결정하거나 재고를 보장하거나 운영 조건을 확정하지 않는다.
    모든 조건은 거래 처리 과정에서 후속 기능이 권한을 갖고 협상하고 다시 검증한다
    (예: 반환된 ID를 `selected_destination_id`로 제출하는 경우.
    [선택과 위치 식별](../../shopping/extensions/fulfillment.md#selection-and-location-identity) 참고).
2. **탐색 관심사의 분리**: 각 기능은 하나의 좁은 질문에만 답하고, 이후 단계에서
    앞선 신호를 다시 검증한다. 위치 기능은 참조된 상품 집합을 현재 어느 장소에서
    제공할 수 있는지에 답한다([`items` 필터](search.md#item-availability-filter)).
3. **잠정적 경계와 권한 있는 경계**:
    * *탐색 단계(잠정적)*: 영업시간, 현재 상품 재고, 편의시설을 바탕으로 한 위치 응답은
        질의 시점의 비즈니스 *현재 조건*을 나타낸다. 이는 **잠정적 신호**이며
        구속력 있는 약속이 아니다.
    * *체크아웃 단계(권한 있음)*: 위치에 따라 달라지는 최종 거래 조건(예: 픽업)은
        반드시 권한 있는 방식으로 협상하고 확정해야 한다(**MUST**). 탐색 신호는 재검증 없이
        캐시하거나 세션 간에 재사용하지 않아야 한다(**SHOULD NOT**).

## 영업시간 { #operating-hours }

### 표현 방식 { #representation }

hours 필터의 [`open_at`](search.md#hours-based-filter) 값은 정확한 시각(instant)이다.
영업시간은 위치의 현지 날짜와 시계 시각을 사용하며, 위치의 IANA 시간대로 해석한다.

* `hours`는 정기 주간 구간의 목록이다. 각 항목은 `day`, `opens`, `closes`를
    포함한다. `day`는 구간이 시작되는 요일을 나타내는 안정적인 UCP 요일 식별자이며,
    현지화된 표시 텍스트가 아니다. 플랫폼은 화면에 표시할 때 이를 현지화할 수
    있다(**MAY**). 시각은 24시간제 `HH:MM` 형식을 사용한다.
* `exception_hours`는 특정 날짜의 시간 구간 또는 전일 휴무의 목록이다. 각 항목은
    `YYYY-MM-DD` 형식으로 경계를 포함하는 현지 날짜 범위 `valid_from`과
    `valid_through`를 가진다. 두 경계가 같으면 하루만 선택한다. `opens`와 `closes`를
    모두 지정하면 시간 구간을, 둘 다 생략하면 전일 휴무를 정의한다. 선택 항목인
    `title`은 사람이 읽을 수 있는 짧은 제목이다. 표시용 메타데이터일 뿐 일정 평가에는
    영향을 주지 않는다.
* `timezone`은 두 일정 모두에 적용되는, 비즈니스가 소유하는 정규 현지 상용시 기준을
    식별한다.

### 평가 { #evaluation }

시간 구간은 `opens`에 열리고 `closes`에 닫힌다. 예를 들어 `10:00`–`17:00`은
`17:00` 직전까지 열려 있고 `17:00`에 닫힌다. `closes`가 `opens`보다 이르면 구간은
다음 현지 날짜로 이어진다. 이 마감 경계 규칙의 예약된 예외로, 정확히 `00:00`–`23:59`인
쌍은 일광 절약 시간 전환을 포함해 현지 상용 날짜 전체를 나타낸다. `opens`와 `closes`가
같은 쌍은 `00:00`–`00:00`을 포함해 모두 유효하지 않다.

일정 평가는 각 시각에 대해 결정적이다. `timezone`을 사용해 시각을 위치의 현지 날짜,
요일, 시각으로 변환한 뒤, 그 현지 값에 유효한 일정을 적용한다. 일광 절약 시간(DST)이
앞으로 전환될 때, 건너뛴 구간의 현지 시계 표기는 어떤 시각에도 대응하지 않으며 다른
시각으로 밀리지 않는다. DST가 뒤로 전환될 때, 반복되는 같은 현지 시각에 대응하는 겹친
구간의 두 시각은 같은 일정 결과를 받는다. 현재의 일정 구조로는 겹친 구간의 두 시각을
구별할 수 없다.

같은 `day`에 대한 `hours` 항목이 여러 개면 분할 근무로 합쳐진다. 어떤 요일이 생략되었다면
그날 시작하는 정기 구간이 없다는 뜻이지만, 전날의 구간이 그날로 이어질 수는 있다.
`hours`가 없으면 정기 일정을 알 수 없다는 뜻이지 휴무라는 뜻이 아니다. 일시적인 전일
휴무를 나타내려면 비즈니스는 정기 `hours`를 그대로 두고, 해당 날짜에 대해 `opens`와
`closes`가 없는 `exception_hours` 항목을 추가한다.

예외 일정은 그것이 적용되는 모든 현지 날짜에서 정기 일정을 대체한다. 어떤 구간이 예외의
적용을 받는 현지 날짜로 이어지면, 현지 자정부터는 그 예외가 우선한다. 동일하지 않은 예외
범위가 서로 겹치면 유효하지 않으며, 한 범위가 다른 범위를 포함하는 경우도 마찬가지다.
경계가 동일한 시간 구간 항목들은 분할 근무로 공존할 수 있지만, 전일 휴무는 해당 경계에서
단독으로만 존재한다. 배열 순서는 우선순위를 정하지 않는다.

#### 예외 영업시간 예시 { #exception-hours-example }

<!-- ucp:example schema=common/types/location op=read direction=response -->
```json
{
  "id": "loc_downtown",
  "name": "Downtown Store",
  "timezone": "America/New_York",
  "exception_hours": [
    {
      "title": "Holiday hours",
      "valid_from": "2026-12-24",
      "valid_through": "2026-12-26",
      "opens": "10:00",
      "closes": "14:00"
    },
    {
      "title": "Holiday hours",
      "valid_from": "2026-12-24",
      "valid_through": "2026-12-26",
      "opens": "16:00",
      "closes": "18:00"
    }
  ]
}
```

두 항목은 날짜 경계가 같으므로, 경계를 포함하는 범위의 각 날짜에 독립적으로 적용되는
분할 근무를 정의한다. 전일 휴무라면 `opens`와 `closes`를 모두 생략한 항목 하나를 사용한다.

### 지침 { #guidelines }

#### 비즈니스 { #business }

각 위치의 정규 일정 기준은 비즈니스가 소유한다. 비즈니스가 `hours`나 `exception_hours`를
내보낼 때는 반드시 모든 `day`, `opens`, `closes`, `valid_from`, `valid_through` 값을 그
기준으로 표현해야 하며(**MUST**), 반드시 `timezone`을 유효한
[IANA(Internet Assigned Numbers Authority) 시간대 데이터베이스](https://www.iana.org/time-zones)
식별자로 포함해야 한다(**MUST**). 비즈니스는 요청한 플랫폼이나 구매자의 시간대에 따라
정규 일정 기준을 바꿔서는 안 된다(**MUST NOT**). JSON Schema가 모든 의미적 제약을
강제하지는 않으므로, 비즈니스는 반드시 위 규칙을 따르는 일정만 내보내야 한다(**MUST**).
여기에는 `opens`와 `closes`가 서로 다를 것, `valid_from` 값이 `valid_through`보다 늦지
않을 것, 예외 범위의 교차가 유효할 것이 포함된다. 비즈니스는 현재 또는 미래의 어떤
시각에도 더 이상 일정에 영향을 줄 수 없게 된 `exception_hours` 항목을 생략해야
한다(**SHOULD**). 또한 자신의 일정이 권한을 갖는 계획 기간까지 알려진 미래 예외를
게시해야 한다(**SHOULD**).

#### 플랫폼 { #platform }

반환된 일정을 평가할 때 플랫폼은 반드시 반환된 위치의 `timezone`을 사용해야
한다(**MUST**). 플랫폼은 표시를 위해 날짜가 정해진 구체적인 발생 시점을 다른 시간대로
변환할 수 있지만(**MAY**), 정규 일정 값을 다른 시간대로 재해석해서는 안 된다(**MUST NOT**).
플랫폼은 일정 데이터가 없거나 유효하지 않거나 달리 사용할 수 없는 위치를 영업 중이라고
추론해서는 안 된다(**MUST NOT**). 플랫폼은 자체 표시 정책에 따라 `title`을 표시할 수
있으며(**MAY**), 이를 위치의 영업 여부를 판단하는 데 사용해서는 안 된다(**MUST NOT**).

## 공유 엔티티 { #shared-entities }

### 편의시설 {: #amenity }

위치 응답은 편의시설을 편의시설 식별자를 키로 하는 맵으로 표현한다.
비즈니스는 반드시 모든 항목에 비어 있지 않은 일반 텍스트 형식의 구매자용 `description`을
제공해야 한다(**MUST**). 비즈니스는 이를 간결한 목록에 그대로 쓸 수 있는 짧고 자기 완결적인
레이블이나 구문으로 만들어야 하며(**SHOULD**), 가능하면 요청에 맞게 현지화해야
한다(**SHOULD**). 이 설명은 편의시설의 식별이나 필터링에 관여하지 않는다. 설명이 없는
항목은 스키마상 유효하지 않으며, 플랫폼은 키에서 표시 텍스트를 추론하는 대신 반드시
유효하지 않은 응답을 거부해야 한다(**MUST**).

편의시설 표시는 식별자와 무관하다. 비즈니스가 제공한 `description`만으로 어떤
편의시설이든 표시할 수 있다. 플랫폼은 자체 표시 정책을 관리하며 편의시설을 표시할지,
어디에 표시할지 결정할 수 있다(**MAY**)(예: 간결한 요약 카드에서는 생략). 편의시설을
표시할 때는 표시하는 모든 항목에 제공된 `description`을 사용해야 하며(**SHOULD**),
식별자를 인식하지 못한다는 이유만으로 항목을 숨겨서는 안 된다(**MUST NOT**). 플랫폼은
인식한 편의시설에 아이콘, 그룹화 등 구조화된 표시를 더할 수 있지만(**MAY**), 인식하지
못한 식별자에서 의미를 추론하거나 이를 다른 편의시설과 동등하게 취급해서는
안 된다(**MUST NOT**).

{{ extension_schema_fields('types/location.json#/$defs/amenity', 'common/location') }}

### 컨텍스트 { #context }

작업에 쓰이는 구매자 위치 및 시장 컨텍스트다. 모든 필드는 관련성과 현지화를 위한 선택적
힌트다. 플랫폼은 요청 헤더에서 컨텍스트를 지리적으로 감지할 수 있다(**MAY**).

컨텍스트 신호는 잠정적인 것이며 권한 있는 데이터가 아니다. 비즈니스는 이를 순위, 현지화,
또는 범위가 제한된 기본 탐색 페이지 선택에 반영할 수 있으며(**MAY**), 더 신뢰도 높은
신호(인증된 계정, 위험 탐지)와 맞지 않으면 무시하거나 순위를 낮출 수 있다(**MAY**).
비즈니스는 이를 명시적인 `distance`나 `serves` 피연산자, 또는 `filters.items`의 상품
식별자 대신 사용해서는 안 된다(**MUST NOT**). 컨텍스트 신호는 근접성, 서비스 가능 여부,
상품 재고 중 어느 것도 증명하지 않는다([요청 문법](search.md#request-grammar) 참고).

{{ schema_fields('types/context', 'common/location') }}

### 신호 { #signals }

인가와 악용 방지를 돕기 위해 플랫폼이 제공하는 환경 데이터다. 신호 값은 구매자가
스스로 주장한 내용이어서는 안 된다(**MUST NOT**). 자세한 내용과 개인정보 보호 요구 사항은
[신호](../../overview/index.md#signals)를 참고한다.

{{ schema_fields('types/signals', 'common/location') }}

## 메시지와 오류 처리 { #messages-and-error-handling }

모든 위치 응답에는 선택적 `messages` 배열이 포함되며, 비즈니스는 이를 통해 오류, 경고,
안내 정보에 대한 맥락을 제공할 수 있다.

### 메시지 유형 { #message-types }

메시지는 비즈니스 처리 결과를 전달하고 맥락을 제공한다.

| 유형 | 사용 시점 | 예시 코드 |
| :--- | :--- | :--- |
| `error` | 비즈니스 수준 오류 | 비즈니스 정의 코드(자유 형식 코드 허용) |
| `warning` | 구매에 영향을 주는 중요한 상황 | `permanently_closed`, `temporary_closure` |
| `info` | 작업을 실패시키지 않는 진단 정보나 추가 맥락 | `not_found`, `holiday_hours_active` |

#### 메시지(오류) { #message-error }

{{ schema_fields('types/message_error', 'common/location') }}

#### 메시지(경고) { #message-warning }

{{ schema_fields('types/message_warning', 'common/location') }}

#### 메시지(정보) { #message-info }

{{ schema_fields('types/message_info', 'common/location') }}

### 일반적인 시나리오 { #common-scenarios }

#### 빈 검색 결과 { #empty-search }

유효한 검색에서 일치하는 결과가 없으면 비즈니스는 반드시 빈 `locations` 배열을 반환해야
한다(**MUST**). 일반적인 불일치에는 메시지가 필요 없으며, `filters.items` 식별자를 알 수
없거나, 재고가 없거나, 평가할 수 없는 경우도 마찬가지다.

<!-- ucp:example schema=common/location_search op=search -->
```json
{
  "ucp": {...},
  "locations": []
}
```

일반적인 빈 결과는 오류가 아니다. 질의는 유효했지만 일치하는 위치가 없었을 뿐이다.

## 전송 바인딩 { #transport-bindings }

위 기능들은 특정 전송 프로토콜에 바인딩된다.

* [REST 바인딩](rest.md): RESTful API 매핑.
* [MCP 바인딩](mcp.md): JSON-RPC를 통한 Model Context Protocol 매핑.

## 보안 및 개인정보 보호 고려 사항 { #security-privacy-considerations }

1. **기본은 대략적으로**: 플랫폼은 탐색 단계에서 기본적으로 대략적인 위치 힌트(예: 우편번호나 반올림한 좌표)를 보내야 한다(**SHOULD**).
  정확한 위치나 좌표는 구매자가 명시적으로 동의하거나 특정 위치를 선택한 경우에만 공유해야 한다(**SHOULD**).
2. **재고 탐색 공격 완화**: 비즈니스는 상품 재고 탐색과 디렉터리의 공격적인 열거를
  줄이기 위해 `filters.items`를 포함한 요청에 속도 제한을 적용해야 한다(**SHOULD**).
3. **비공개/다크 위치**: 비즈니스는 내부 전용이거나 구매자가 접근할 수 없는 위치(예: 다크 키친, 풀필먼트 전용 허브)를
  반드시 검색 결과에서 제외해야 한다(**MUST**).
4. **물리적 주소 위조(무결성)**: 위치 탐색은 읽기 전용이지만, 응답의 물리적 주소가 변조되면(예: MITM 공격)
  물리적 안전 및 사기 위험이 생긴다. 플랫폼은 위치 페이로드를 구매자에게 보여주기 전에 서명을 검증해야 한다(**SHOULD**).
5. **데이터 보존 및 로그 정제**: 비즈니스는 구매자의 명시적 동의를 받지 않는 한 정확한 위치 입력을 요청 수명 주기 이후까지
  저장해서는 안 된다(**MUST NOT**). 서버 로그는 정확한 구매자 이력이 실수로 저장되지 않도록 좌표 입력의 소수점 자릿수를
  줄여(예: 소수점 둘째 자리, 약 1km 정확도) 정제해야 한다.
