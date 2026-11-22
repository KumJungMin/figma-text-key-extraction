# Figma Text Key Extraction

Figma의 TextNode를 i18n 키 규칙에 따라 수집하고, 화면별 문구와 공통 문구를 하나의 Nested JSON으로 추출하는 **읽기 전용 Figma 플러그인**입니다.

> 상태: 설계 문서 초안(v0.1). 플러그인 구현 및 배포는 아직 시작되지 않았습니다.

## 핵심 사용 사례

- 화면 Frame `ez.join.idCardCamera` 안의 `*title` → `ez.join.idCardCamera.title`
- 어느 화면에서나 `*common.confirm` → `common.confirm`
- `common` Group/Frame 안의 `*cancel` → `common.cancel`
- 동일한 키의 다른 문자열, JSON 경로 충돌을 사전에 검증한 뒤 `ko.json`과 같은 단일 파일 내보내기

~~~json
{
  "ez": { "join": { "idCardCamera": { "title": "신분증 촬영" } } },
  "common": { "confirm": "확인", "cancel": "취소" }
}
~~~

## 문서

| 문서 | 역할 |
| --- | --- |
| [PRD](docs/PRD.md) | 배경, 대상 사용자, 사용자 흐름, MVP 및 성공 기준 |
| [REQUIREMENTS](docs/REQUIREMENTS.md) | 기능/비기능 요구사항과 인수 조건 |
| [KEY_RESOLUTION](docs/KEY_RESOLUTION.md) | 레이어 명명, 키 해석 우선순위 및 예외 |
| [EXPORT_FORMAT](docs/EXPORT_FORMAT.md) | 출력 JSON, 중복/충돌, 진단 결과 계약 |
| [ARCHITECTURE](docs/ARCHITECTURE.md) | 구현 모듈, 데이터 흐름, 경계와 기술 제안 |
| [TEST_PLAN](docs/TEST_PLAN.md) | 요구사항 추적 가능한 테스트 케이스 |

## 설계 상태

**확정:** 화면 Frame 기반 네임스페이스, 명시적 공통 키 + `common` Group/Frame, 언어별 통합 Nested JSON.

**제안(검토 가능):** 추출 범위 3종, 기본 숨김 노드 포함, 공백 값 제외, 충돌 시 다운로드 차단, TypeScript + Vitest.

**MVP 제외:** 자동 번역, Google Sheets 동기화, 양방향 동기화, 기존 JSON과의 Diff.

## 연관 프로젝트

키 네이밍과 조상 키 상속에 대한 초기 참고 구현: [sheets-i18n-figma-sync (`feature/ko`)](https://github.com/KumJungMin/sheets-i18n-figma-sync/tree/feature/ko).

기존 Import 코드는 `common.*`를 화면 절대 키로 취급하지 않으므로 새 플러그인과 **완전한 왕복 호환성은 아직 보장되지 않습니다**. [키 규칙](docs/KEY_RESOLUTION.md) 참조.
