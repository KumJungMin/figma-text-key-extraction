# Functional & Non-functional Requirements

- 상태: Draft · v0.1
- 우선순위: **P0 필수**, **P1 개선**
- `확정`은 사용자 결정, `제안`은 구현 기본값이며 리뷰로 변경 가능.
- 규범적인 키 상세 규칙은 [KEY_RESOLUTION](KEY_RESOLUTION.md), 출력 계약은 [EXPORT_FORMAT](EXPORT_FORMAT.md)을 단일 출처로 삼는다.

## 기능 요구사항

| ID | 우선 | 요구사항 | 인수 기준 | 상태 |
| --- | --- | --- | --- | --- |
| FR-001 | P0 | Figma `TEXT` 노드의 `name`, `characters` 읽기 | 원본을 수정하지 않음 | 확정 목표 |
| FR-002 | P0 | 화면 `FRAME`의 이름을 화면 네임스페이스로 사용 | `ez.join.login` + `*title` → `ez.join.login.title` | 확정 |
| FR-003 | P0 | 명시적 `*common.xxx` 지원 | 화면 안에서도 `common.xxx` | 확정 |
| FR-004 | P0 | `common` Group/Frame 컨텍스트 지원 | 내부 `*confirm` → `common.confirm` | 확정 |
| FR-005 | P0 | TextNode 직접 키 우선 | 직접 키가 있으면 조상 `*key`보다 우선 | 제안 |
| FR-006 | P0 | 키가 있는 조상/하위 TEXT 하나인 경우 상속 | `*title` Group 아래 TEXT 하나면 상속 | 기존 Import 호환 |
| FR-007 | P0 | 일반 TextNode 무시 및 미해석 키 경고 | 유효하지 않거나 Frame이 없는 상대 키가 출력되지 않음 | 제안 |
| FR-008 | P0 | 현재 Page 범위 추출 | 현재 페이지 밖은 포함하지 않음 | 제안 |
| FR-009 | P0 | 선택 영역 범위 추출 | 선택한 노드 및 하위 TEXT만, 키 탐색은 외부 조상까지 | 제안 |
| FR-010 | P0 | 전체 파일 범위 추출 | 모든 Page의 키를 같은 출력 테이블에서 검증 | 제안 |
| FR-011 | P0 | 동일 키의 중복 처리 | 같은 값은 병합, 다른 값은 오류 | 제안 |
| FR-012 | P0 | Nested JSON 경로 충돌 검출 | `a`와 `a.b` 공존 시 오류 | 제안 |
| FR-013 | P0 | 언어별 단일 JSON 생성 | 선택 언어에 대해 `ko.json` 등 하나를 생성 | 확정 |
| FR-014 | P0 | 원본 문자열 유지 | trim/번역 없이 개행, 중괄호, 유니코드 보존 | 제안 |
| FR-015 | P0 | 결과 요약/미리보기/진단 | 건수, 키, 원본 Node 위치를 확인 가능 | 제안 |
| FR-016 | P0 | 오류 시 안전한 내보내기 | 오류 또는 출력 0건이면 다운로드 차단 | 제안 |
| FR-017 | P0 | Figma DOM read-only | `.characters`, `.name` 등 어떠한 필드도 쓰지 않음 | 확정 목표 |
| FR-018 | P1 | 레이어로 이동 기능 | 진단 항목에서 원본 노드를 찾기 쉽게 지원 | 제안 |

## 추출·UI 상태

| 상태 | 설명 | 허용 작업 |
| --- | --- | --- |
| `idle` | 스캔 전 | 범위/언어 변경, 스캔 |
| `scanning` | 조회 및 검증 중 | 진행 표시, 중복 실행 방지 |
| `ready` | 정상 키 >= 1건, 차단 오류 없음 | JSON 미리보기 및 다운로드 |
| `blocked` | 키 충돌/경로 충돌/기타 차단 오류 | 진단 확인, 재스캔 |
| `empty` | 유효 키 0건 | 범위/레이어 규칙 점검 |
| `failed` | Figma API 또는 직렬화 실패 | 오류 표시, 재시도 |

값 또는 레이어가 변경되었다면 사용자가 다시 스캔한다. 지속적인 자동 감시는 MVP 제외.

## 비기능 요구사항

| ID | 요구사항 | 검증 |
| --- | --- | --- |
| NFR-001 | 결정적인 출력 | 동일 입력·옵션이면 동일 키 순서/JSON |
| NFR-002 | 안전성 | 네임스페이스/객체 생성에서 Prototype Pollution 차단 |
| NFR-003 | 무손실 값 추출 | `characters`를 원문 그대로 보존 |
| NFR-004 | 읽기 전용 | 어떤 경로에서도 Figma 노드 쓰기 API 호출 없음 |
| NFR-005 | 대형 파일 안정성 | Page 순차 로드, 처리 중 UI 응답성 보존(실측 목표 추후 결정) |
| NFR-006 | 진단 가능성 | 오류에 출처 Node/Page ID와 오류 코드를 포함 |
| NFR-007 | 개인정보 최소화 | 외부 서버 전송·불필요한 플러그인 영구 저장 없음 |
| NFR-008 | 테스트 가능성 | Figma 런타임 없이 순수 키 해석/검증 로직 테스트 가능 |

## 최소 UI 구성 (제안)

- 추출 범위: `current-page` / `selection` / `entire-file`
- 언어 코드: `ko` 기본값(사용자가 실제 텍스트 언어 확인)
- `Scan`, 결과 카운트, 키별 미리보기, 오류 목록, `Export JSON`
- 오류 목록에는 사유와 Node 식별자를, 충돌에는 모든 관련 TextNode를 표시
- 출력은 명시적 재스캔 결과로만 만들어지며, `Export` 직전 최신 스캔 유무를 표시

## 참조

- [PRD](PRD.md)
- [키 규칙](KEY_RESOLUTION.md)
- [출력/검증 계약](EXPORT_FORMAT.md)
- [테스트 계획](TEST_PLAN.md)
