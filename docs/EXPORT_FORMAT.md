# Export Format & Validation Contract

- 상태: Draft · v0.1
- **확정:** 화면/공통 키를 합친 언어별 **단일 Nested JSON**.
- 정렬·진단 심각도·오류 차단 정책은 검토 가능한 제안 기본값이다.

## 1. 산출물

- 입력: 유효한 `(finalKey, characters, source)` 항목의 집합, 사용자 선택 언어 코드.
- 출력: `<language>.json` (예: `ko.json`). 언어 선택은 라벨일 뿐 자동 감지/번역은 하지 않는다.
- 내용: 최상위 JSON object, dot-path를 중첩 object로 변환한 리소스만 포함. `_meta`, 원본 Node ID, 통계는 JSON에 포함하지 않는다.
- 들여쓰기: 2칸, 마지막 줄바꿈 추가. 키는 알파벳순의 결정적 정렬을 사용한다.

~~~json
{
  "common": {
    "cancel": "취소",
    "confirm": "확인"
  },
  "ez": {
    "join": {
      "idCardCamera": {
        "description": "신분증을 준비해 주세요",
        "title": "신분증 촬영"
      }
    }
  }
}
~~~

## 2. 문자열 보존

- `TEXT.characters`를 가공하지 않는다. 앞뒤 공백·중간 공백·개행·이모지·한국어·라오어·`{name}` 등 그대로 보존.
- 단, `value.trim().length === 0`이면 **비어 있는 값으로 분류**하고 제외하는 정책 제안.
- 직렬화는 안전한 JSON serializer를 사용하여 `\n`, `\"`, 역슬래시를 적절히 이스케이프한다.
- 리치 텍스트의 폰트, 색상 등 스타일 속성은 추출하지 않는다.
- 값은 단순 `string`. 키 또는 값에 자동 번역·문자열 치환을 적용하지 않는다.

## 3. 동일 키 병합 및 차단

먼저 모든 엔트리를 finalKey로 그룹화해 검사한 뒤 Nested JSON을 생성한다.

| 조건 | 분류 | 동작 |
| --- | --- | --- |
| 하나의 키/하나의 값 | 정상 | 값 출력 |
| 같은 키, 완전히 같은 값 2개 이상 | `DUPLICATE_IDENTICAL` (정보) | 한 번 출력; 원본 목록과 중복 개수 보관 |
| 같은 키, 값이 하나라도 다름 | `DUPLICATE_CONFLICT` (차단 오류) | 묵시적인 first/last wins 금지 |
| `a`와 `a.b`가 동시에 존재 | `PATH_COLLISION` (차단 오류) | 객체/문자열 충돌 감지 |
| `a.b`와 `a.b.c` 공존 | `PATH_COLLISION` (차단 오류) | 상위 leaf가 객체와 충돌 |
| 유효한 데이터가 0개 | `NO_VALID_ENTRIES` (차단) | 다운로드 불가 |

동일성 비교는 **완전한 문자열 일치**로 처리한다. `"확인"`과 `"확인 "`은 다른 값이므로 충돌로 처리한다.

차단 오류가 존재하면 미리보기와 진단은 제공하지만 다운로드는 불가. 일부만 내보내는 옵션은 MVP 제외.

## 4. 내부 데이터 계약(초안)

~~~ts
type SourceKind = "screen" | "common";
type ResolutionKind = "direct" | "ancestor";

interface ExportEntry {
  finalKey: string;
  value: string; // original TextNode.characters
  sourceKind: SourceKind;
  resolution: ResolutionKind;
  nodeId: string;
  nodeName: string;
  pageId: string;
  pageName: string;
  frameId?: string;
  frameName?: string;
  ownerNodeId?: string; // inherited *key ancestor
}

type Severity = "info" | "warning" | "error";

interface ExportIssue {
  code: string; // e.g. DUPLICATE_CONFLICT, PATH_COLLISION
  severity: Severity;
  message: string;
  key?: string;
  nodeIds: string[];
}

interface ExportReport {
  scannedTextNodeCount: number;
  keyedTextNodeCount: number;
  exportedKeyCount: number;
  duplicateIdenticalCount: number;
  skippedCount: number;
  issues: ExportIssue[];
  entries: ExportEntry[];
  json?: Record<string, unknown>; // only when not blocked
}
~~~

이 인터페이스는 런타임/API 안정화를 위한 초안이다. 실제 타입은 구현 중 구조와 일치하도록 동기화한다.

## 5. 안전한 Nested JSON 변환

- 문자열 → 객체 덮어쓰기를 절대 허용하지 않는다.
- `__proto__`, `prototype`, `constructor` 세그먼트는 키 파싱 단계에서 거부한다.
- 중첩 객체는 `Object.create(null)` 등 프로토타입 안전한 맵을 권장한다.
- 정렬 후 직렬화하여 Figma 노드 방문 순서가 결과 파일 내용에 영향을 주지 않도록 한다.
- Figma 플러그인 실행 컨텍스트가 읽기를 수행하고, UI가 다운로드 처리하는 구조를 제안한다.

## 6. 진단/미리보기

키별 현재 값, 위치(Page/Frame/Node 이름과 ID), 검증 상태를 보여준다. 같은 키의 이슈가 발생한 경우 모든 관련 Node를 함께 보관한다. 사용자에게 표시하는 JSON은 디버그 메타데이터가 없는 실제 다운로드 내용과 동일해야 한다.
