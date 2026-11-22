# Key Resolution Specification

- 상태: Draft · v0.1
- 목적: Figma 레이어 트리에서 `TEXT` 하나에 대해 최종 i18n 키를 결정하는 순수 규칙.
- **확정 사항:** 화면 Frame 네임스페이스, 명시적/그룹형 common, 통합 Nested JSON.
- 다음의 세부 우선순위/문법/오류 정책은 검토 가능한 **제안 기본값**이다.

## 1. 용어와 명명

- **화면 Frame**: `type === "FRAME"`이며 이름이 점(`.`)으로 구분된 유효한 네임스페이스(예: `ez.join.login`)인 조상. Figma `PAGE` 이름과 구분한다.
- **공통 컨테이너**: 이름이 정확히 `common`이고 타입이 `FRAME` 또는 `GROUP`인 조상. 일반 UI 그룹명 `commonButtons`는 포함하지 않는다.
- **직접 키**: TEXT 레이어 `name`의 첫 글자가 `*`이고 뒤에 유효한 키가 존재하는 경우.
- **조상 키**: TEXT의 조상 `GROUP` 또는 `FRAME` 이름이 `*key`인 경우. 키가 없는 TEXT가 물려받는다.
- **상대 키**: 컨텍스트 이름을 앞에 붙여 완성해야 하는 키, 예: `*title`, `*header.title`.
- **절대 키**: `*common.confirm` 또는 현재 화면 네임스페이스로 시작하는 완성 키, 예: `*ez.join.login.title`(현재 Frame `ez.join.login`).

### 문법 (제안)

- 키는 하나 이상의 ASCII 식별자 세그먼트와 점(`.`)으로 구성.
- 식별자: `[A-Za-z_][A-Za-z0-9_]*`. 점이 연속하거나 앞뒤에 오면 오류.
- Key 추출은 기존 Import처럼 이름 **첫 글자**가 `*`일 때만 실행. `*` 뒤의 선행/후행 공백은 제거. ` *title`은 직접 키가 아니다.
- 대소문자를 구분. 정규화나 자동 수정하지 않는다.
- 위험 세그먼트 `__proto__`, `prototype`, `constructor`는 거부한다.
- 화면 Frame 후보는 `FRAME` 타입, 최소 두 세그먼트, `common`과 같지 않은 이름으로 제한한다. `*title` 같은 keyed 컨테이너는 화면 Frame 후보에서 제외한다.
- 이름 `common`은 예약어로 사용한다.

## 2. 해석 우선순위

1. TEXT 자신의 직접 키가 유효하면 먼저 사용한다. 직접 키가 잘못되었다면 조상으로 폴백하지 않고 **키 오류**를 보고한다.
2. 직접 키가 없으면 가장 가까운 `*key` 조상을 찾는다. 그 조상의 **전체 TEXT 자손 수**가 정확히 1일 때만 상속한다. 2개 이상이면 `AMBIGUOUS_ANCESTOR`로 제외한다.
3. 해석할 키가 `common.`으로 시작하면 위치와 무관하게 절대 공통 키로 사용한다.
4. 아니면 TEXT에서 조상으로 올라가 **가장 가까운 유효한 네임스페이스 경계**를 찾는다. 경계는 `common` Group/Frame 또는 화면 Frame이다.
5. 공통 경계라면 `common.`을 앞에 붙인다(이미 `common.`이면 그대로). 화면 Frame이라면 프레임 이름으로 시작하는 키는 그대로, 그 외는 `frame.name + "." + key`로 만든다.
6. 경계가 없고 공통 절대 키도 아니면 `MISSING_NAMESPACE`로 제외한다.

이 정책은 TextNode 바로 위의 `common` 컨테이너가 화면 Frame보다 가까우면 공통 경계를 택한다. 반대로 더 가까운 화면 Frame이 존재하면 그 화면 Frame을 택한다.

## 3. 해석 예제

| Figma 구조 | 원본 키 | 최종 키 / 처리 |
| --- | --- | --- |
| `FRAME ez.join.login` > TEXT | `*title` | `ez.join.login.title` |
| `FRAME ez.join.login` > TEXT | `*header.title` | `ez.join.login.header.title` |
| `FRAME ez.join.login` > TEXT | `*ez.join.login.title` | `ez.join.login.title` |
| `FRAME ez.join.login` > TEXT | `*common.confirm` | `common.confirm` |
| `GROUP common` > TEXT | `*cancel` | `common.cancel` |
| `FRAME ez.join.login` > `GROUP common` > TEXT | `*cancel` | `common.cancel` |
| `FRAME ez.join.login` > `GROUP Content` > TEXT | `*title` | `ez.join.login.title` |
| `GROUP *title` > TEXT 이름 `Label` (텍스트 하나) | 없음 | 조상 키 적용(유효 경계 필요) |
| `GROUP *title` > TEXT 2개 | 없음 | `AMBIGUOUS_ANCESTOR` |
| `GROUP *header` > TEXT `*title` | `*title` | 직접 키 우선, `*header`는 prefix 아님 |
| 화면 Frame 없음 > TEXT | `*title` | `MISSING_NAMESPACE` |
| 화면 Frame 없음 > TEXT | `*common.confirm` | `common.confirm` |
| `GROUP commonButtons` > TEXT | `*confirm` | 공통 경계 아님(다른 화면 경계 없으면 제외) |

### 비명시적 절대 키 주의

`ez.join.login` 아래 `*login.title`은 `ez.join.login.login.title`로 해석한다. 다른 화면의 완전 키를 자동 추측해 절대 키로 처리하지 않는다. 서로 다른 화면에 걸친 절대 키 참조는 MVP에서 `common.*`로만 공식 지원한다.

## 4. 조상 키 예외

~~~text
FRAME ez.join.login
└── GROUP *title
    └── TEXT Label       "로그인"
~~~

TEXT의 직접 키가 없고 `*title`의 TEXT 자손이 1개라면 `ez.join.login.title`이다.

~~~text
FRAME ez.join.login
└── GROUP *title
    ├── TEXT Label A     "로그인"
    └── TEXT Label B     "시작"
~~~

자손이 2개이므로 **두 TextNode 모두** 조상 상속 실패. `*title`에서 임의로 하나를 고르지 않는다.

TEXT에 직접 `*subtitle`을 부여하면 그 TEXT는 직접 키로 해석된다. 다른 자손이 키 없으면 여전히 모호한 조상 상속으로 제외된다. 자손 수는 선택 영역 범위에 의해 달라지지 않으며 전체 실제 자손을 기준으로 센다.

## 5. 오류와 특수 케이스

| 오류 코드 | 조건 | 처리 |
| --- | --- | --- |
| `INVALID_KEY` | `*`, `*a..b`, `*a b` 등 | 오류 표시, 해당 TEXT 제외 |
| `MISSING_NAMESPACE` | 상대 키에 유효한 화면/공통 경계 없음 | 경고 표시, 해당 TEXT 제외 |
| `AMBIGUOUS_ANCESTOR` | 조상 `*key`의 TEXT 자손 2개 이상 | 경고 표시, 해당 TEXT 제외 |
| `INVALID_NAMESPACE` | 조상 후보가 규칙을 만족하지 않음 | 다른 상위 유효 경계 탐색; 없으면 `MISSING_NAMESPACE` |
| `EMPTY_VALUE` | TEXT `characters`가 빈 문자열 또는 공백만 | 경고 표시, 해당 TEXT 제외 |

숨김 노드, Component, Instance 안 TEXT는 기본적으로 포함하는 안을 제안한다. 인스턴스 레이어에서 실제로 노출된 이름과 문자열을 기준으로 한다.

## 6. 기존 Import와 다른 점

[기존 Import의 `feature/ko/code.js`](https://github.com/KumJungMin/sheets-i18n-figma-sync/blob/feature/ko/code.js)는 `keyFromNodeName`, `resolveFrameNamespace`, `resolveKeyForTextNode`를 구현하지만, dot이 있는 Group 조상도 화면 후보로 인정하고 `common.*`를 전역 절대 키로 우선 취급하지 않는다.

따라서 **기존 Import와 완전 호환이 아니다**. 이번 문서를 새로운 규범으로 두되, Import의 공통 로직을 이 규칙으로 마이그레이션해야 완전한 Export↔Import 왕복을 주장할 수 있다.
