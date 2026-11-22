# Test Plan & Acceptance Criteria

- 상태: Draft · v0.1
- 테스트 파일은 가능한 한 순수 TypeScript 함수 대상으로 만들고, Figma API는 어댑터 테스트에서 mock한다.
- P0 케이스는 MVP 릴리스 차단 조건이다.

## 1. 키 해석 (P0)

| TC | 관련 요구사항 | Fixture | 기대 |
| --- | --- | --- | --- |
| KR-01 | FR-002 | Frame `ez.join.login` > TEXT `*title` | `ez.join.login.title` |
| KR-02 | FR-002 | 위 Frame > TEXT `*header.title` | `ez.join.login.header.title` |
| KR-03 | FR-002 | 위 Frame > TEXT `*ez.join.login.title` | 절대 키 중복 prefix 없음 |
| KR-04 | FR-003 | 위 Frame > TEXT `*common.confirm` | `common.confirm` |
| KR-05 | FR-004 | Group `common` > TEXT `*cancel` | `common.cancel` |
| KR-06 | FR-004 | Frame `ez.join.login` > Group `common` > TEXT `*cancel` | `common.cancel` |
| KR-07 | FR-006 | Frame `ez.join.login` > Group `*title` > TEXT `Label` | `ez.join.login.title` |
| KR-08 | FR-006 | Group `*title` > 키 없는 TEXT 2개 | 양쪽 모두 `AMBIGUOUS_ANCESTOR` |
| KR-09 | FR-005 | Group `*title` > TEXT `*subtitle` | 직접 키 `subtitle` 우선 |
| KR-10 | FR-007 | 프레임 밖 TEXT `*title` | `MISSING_NAMESPACE` |
| KR-11 | FR-003 | 프레임 밖 TEXT `*common.confirm` | `common.confirm` |
| KR-12 | FR-007 | TEXT `*title..a` | `INVALID_KEY` |
| KR-13 | FR-007 | TEXT `*__proto__.x` | `INVALID_KEY` |
| KR-14 | FR-007 | TEXT 이름 ` *title` | 직접 키로 인식 안 함 |
| KR-15 | FR-009 | 조상 Group에서 TEXT 하나만 선택(실제 TEXT 자손 2개) | 상속 모호성을 유지 |
| KR-16 | FR-002 | 일반 중간 Group/Frame + 유효 화면 Frame | 화면 Frame 이름 적용 |

## 2. 수집/검증 (P0)

| TC | 관련 요구사항 | 입력 | 기대 |
| --- | --- | --- | --- |
| VL-01 | FR-011 | `common.confirm` = `"확인"` 3개 | 동일 중복 2개, 출력 1개 |
| VL-02 | FR-011 | `common.confirm` = `"확인"` / `"완료"` | `DUPLICATE_CONFLICT`, 다운로드 차단 |
| VL-03 | FR-011 | `common.confirm` = `"확인"` / `"확인 "` | 다른 값으로 충돌 |
| VL-04 | FR-012 | `a` = `"x"`, `a.b` = `"y"` | `PATH_COLLISION` |
| VL-05 | FR-014 | 개행, `{name}`, 유니코드 포함 | JSON 왕복 후 원본 문자열과 동일 |
| VL-06 | FR-013 | 화면 + 공통 키 동시 존재 | 하나의 Nested JSON |
| VL-07 | FR-016 | 유효 키 0개 | `NO_VALID_ENTRIES`, 다운로드 차단 |
| VL-08 | FR-013, NFR-001 | 노드 탐색 순서 뒤섞기 | 같은 JSON 문자열 |
| VL-09 | FR-014 | `characters`가 공백만 포함 | `EMPTY_VALUE`, 출력 제외 |
| VL-10 | FR-012, NFR-002 | 위험 키 세그먼트 | 거부 및 prototype 오염 없음 |

## 3. 스코프 및 플러그인 통합 (P0)

| TC | 관련 요구사항 | 기대 |
| --- | --- | --- |
| IN-01 | FR-008 | 현재 Page 밖 TEXT가 결과에 들어가지 않음 |
| IN-02 | FR-009 | 선택 영역 중첩으로 같은 TEXT를 두 번 조회하지 않음 |
| IN-03 | FR-009 | 선택된 TEXT의 선택 밖 조상 Frame을 키 해석에 사용 |
| IN-04 | FR-010 | 다른 Page에 같은 키/다른 값이 있으면 충돌 검출 |
| IN-05 | FR-017, NFR-004 | 모든 스캔 경로에서 Figma 노드 쓰기 API 미호출 |
| IN-06 | FR-015 | 오류 목록에 관련 Node ID와 키, 이유가 존재 |
| IN-07 | FR-016 | 충돌 상태에서 다운로드 액션이 실행되지 않음 |
| IN-08 | FR-001 | 숨김 TEXT/Instance TEXT 처리 정책 검증(제안 정책 기준) |

## 4. QA 수동 점검

1. Figma에 화면 Frame, `common` Group, TEXT 노드를 복합 구성한다.
2. 현재 Page/선택/전체 파일 범위를 각각 실행해 포함 범위를 확인한다.
3. 오류 발생 시 오류 목록에서 실제 Layer를 찾을 수 있는지 확인한다.
4. `ko.json` 다운로드 후 `JSON.parse`가 성공하고 값·키를 원본과 비교한다.
5. 스캔 전후 동일 노드의 `name`, `characters` 및 주요 속성이 변하지 않았는지 확인한다.
6. 충분히 큰 Figma 파일에서 UI 응답성과 처리 누락을 측정한다.

## 5. MVP 인수 기준

- [ ] P0 `KR`, `VL`, `IN` 테스트 통과
- [ ] 차단 오류 발생 시 손상되거나 불완전한 JSON을 다운받을 수 없음
- [ ] 두 번 연속 같은 옵션으로 스캔 시 같은 결과
- [ ] Figma 원본 미수정
- [ ] 문서의 확정 결정(PD-01~03) 위반 없음

## 6. 나중에 추가할 테스트(P1)

- 아주 큰 데이터의 처리 제한/취소
- 오류 항목 클릭 시 Node 포커스/선택
- 언어별 텍스트 비교
- 기존 Import와 Export round-trip 검사(Import 개선 이후)
