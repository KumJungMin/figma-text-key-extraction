# Architecture & Implementation Plan

- 상태: Draft · v0.1
- 구현 기술은 **제안**이며, 제품 확정 사항과 구분한다.

## 1. 원칙

1. Figma API/페이지 탐색은 Adapter에서, 키 해석·검증·JSON 생성은 순수 TypeScript 함수에서 처리한다.
2. 읽기 전용: Figma 노드의 `name`, `characters`, 스타일 등을 변경하지 않는다.
3. 키 규칙의 단일 출처는 [KEY_RESOLUTION](KEY_RESOLUTION.md), 파일 규격은 [EXPORT_FORMAT](EXPORT_FORMAT.md).
4. 수집한 모든 원본은 `nodeId`/`pageId`로 추적 가능해야 한다.
5. 유효하지 않은 데이터가 있어도 오류가 숨겨지지 않게 Report로 전달한다.

## 2. 권장 모듈 배치

~~~text
src/
├── plugin/
│   ├── main.ts               # Figma plugin 진입점/메시지 수신
│   ├── figma-scanner.ts      # Page·선택 영역 탐색 및 snapshot 추출
│   └── node-adapter.ts       # Figma 노드를 코어 입력으로 변환
├── core/
│   ├── key-resolver.ts       # 직접·조상·common·화면 네임스페이스
│   ├── entry-collector.ts    # TEXT → ExportEntry
│   ├── export-validator.ts   # 중복·경로 충돌
│   ├── json-builder.ts       # flat → nested JSON
│   └── types.ts              # 순수 코어 계약
├── ui/
│   ├── index.html
│   └── main.ts               # 범위, 언어, 미리보기, 이슈, 다운로드
└── tests/
    ├── fixtures/
    ├── key-resolver.test.ts
    ├── export-validator.test.ts
    └── json-builder.test.ts
manifest.json
package.json
vite.config.ts
~~~

예상 툴체인: TypeScript, Vite, Vitest. UI 프레임워크는 초기 MVP에서 선택 사항이며 필요 없으면 vanilla TS도 충분하다.

## 3. 데이터 파이프라인

~~~text
UI: options(scope, language)
   ↓ postMessage(scan)
Plugin: FigmaScanner (read-only)
   ↓ NodeSnapshot[]
Core: KeyResolver → EntryCollector
   ↓ ExportEntry[]
Core: ExportValidator (same-key, value, prefix/path collisions)
   ↓ ExportReport (issues + optional flat valid map)
Core: JsonBuilder (stable key sorting)
   ↓ JSON string + report
Plugin → UI: postMessage(scan-result)
   ↓ Preview, diagnostics, download
~~~

`NodeSnapshot`은 키 결정에 필요한 이름/타입/부모관계/ID/텍스트만 담는다. Figma의 실제 Node 객체는 UI로 전송하지 않는다.

## 4. 스캔 옵션과 탐색

~~~ts
type ScanScope = "current-page" | "selection" | "entire-file";

interface ScanOptions {
  scope: ScanScope;
  language: string; // e.g. ko, en, lo
}
~~~

- `current-page`: 현재 Figma Page의 `TEXT`를 탐색한다.
- `selection`: 선택한 노드 및 하위 TEXT만 처리하고 키 해석에는 선택 밖 부모도 참조한다. 중복 선택된 조상/자식은 한 번만 처리한다.
- `entire-file`: 모든 Page를 순차 탐색해 한꺼번에 중복/충돌을 검증한다. Figma의 페이지 로드 정책에 따라 `loadAsync()` 등 적합한 API를 구현 시 검증한다.
- 스캔 중 UI 응답성 보장을 위해 페이지/노드 단위 비동기 처리 또는 chunking을 권장한다.
- 실제 성능/메모리 목표는 대형 Figma 파일 측정 뒤 결정한다.

## 5. 권장 순수 함수 경계

~~~ts
resolveKey(node: NodeSnapshot, context: ResolutionContext): KeyResolution
collectEntries(nodes: NodeSnapshot[], context: ResolutionContext): ExportEntry[]
validateEntries(entries: ExportEntry[]): ExportIssue[]
buildNestedJson(entries: ExportEntry[]): Record<string, unknown>
serializeJson(json: Record<string, unknown>): string
~~~

이 함수 이름은 구현 제안이며 API 계약으로 고정된 것은 아니다.

## 6. 단계별 구현

- **Phase 1 — Core:** 레이어 스냅샷, KeyResolver, 검사기, JSON Builder와 단위 테스트
- **Phase 2 — Plugin Adapter:** 현재 Page/선택 영역 스캔, Node ID 추적, read-only 보장
- **Phase 3 — UI:** 범위/언어 입력, 요약/이슈/미리보기, JSON 다운로드
- **Phase 4 — 확대:** 전체 파일 스캔, 중대형 문서 성능 검증, 오류 UX 보완

## 7. 기존 프로젝트와의 관계

`sheets-i18n-figma-sync`의 함수/규칙을 **참고**하되 이 저장소는 독립 프로젝트다. 기존 Import에는 `common.*` 전역 키 우선 규칙이 없으므로 별도 리팩터 없이 완전한 왕복 호환을 약속하지 않는다.

## 8. 기술 검증 포인트

- 사용 중인 Figma Plugin API에서 Page 탐색/동적 로드 지원 방식
- Component/Instance 내부 TEXT와 숨김 노드 조회 동작
- Sandbox ↔ UI 메시지 크기와 대형 JSON 다운로드 방식
- 정규식/키 문법이 기존 운영 키 집합과 호환되는지
- 원본 노드 쓰기 API 호출이 전혀 없는지
