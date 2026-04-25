# 생기부 작성 도우미 — 프로젝트 가이드

> 미래의 Claude 세션을 위한 컨텍스트 문서. 모든 작업 전에 반드시 읽을 것.

---

## 1. 프로젝트 개요

한국 고등학교 교사가 학교생활기록부(생기부)를 작성하기 위한 단일 페이지 HTML 프로그램 두 개. 사용자가 학생 정보·키워드를 입력하면 AI가 NEIS 양식에 맞는 문장을 생성한다.

- **helper1.html** — 창의적 체험활동(자율/동아리/진로/봉사) + 행동특성 종합의견 작성용. 학생 수준(상/중/하) 옵션 지원.
- **helper2.html** — 교과 세부능력 및 특기사항(세특) 작성용. 17개 과목군 분류, 성취도(A/B/C/D/E) 옵션 지원.

배포: GitHub Pages — https://jerry5026-dotcom.github.io/saenggibu-writer/

---

## 2. 파일 구조

```
생기부 작성 프로그램/
├── helper1.html              ← 본체 (창체+행특)
├── helper2.html              ← 본체 (세특)
├── 생기부 작성 도우미1.html   ← helper1.html로 자동 리다이렉트 (한글 URL 보존)
├── 생기부 작성 도우미2.html   ← helper2.html로 자동 리다이렉트
├── index1.html               ← helper1.html로 리다이렉트 (짧은 URL)
├── index2.html               ← helper2.html로 리다이렉트
├── CLAUDE.md                 ← 이 문서
└── .claude/worktrees/modest-jang/  ← 작업용 워크트리
```

**원칙**: 코드 수정은 `helper1.html`, `helper2.html` 두 파일에만. 리다이렉트 파일은 건드리지 않는다.

---

## 3. 기술 스택 / 아키텍처

- **프론트엔드만**: 단일 HTML, 바닐라 JS, 외부 프레임워크 없음, 빌드 단계 없음
- **상태 저장**: `localStorage` (사용자 입력값, API 키, 선택 모델 등)
- **AI 호출**: 브라우저에서 직접 OpenAI/Gemini/Claude API fetch
- **타임아웃**: `AbortController` 사용 — 문장 생성 60초, 키 검증 8초

### 멀티 프로바이더 AI 키 구조

```js
S.api = {
  active: 'openai' | 'gemini' | 'claude',
  keys:   { openai: '...', gemini: '...', claude: '...' },
  models: { openai: '...', gemini: '...', claude: '...' }
}
```

옛 단일 키 구조(`{provider, key, model}`)에서 자동 마이그레이션. 헤더 드롭다운으로 활성 프로바이더 전환.

### 키 형식 검증 + 실 ping

| 프로바이더 | 형식 | 검증 엔드포인트 |
|---|---|---|
| OpenAI | `sk-` 시작 | `/v1/models` |
| Anthropic | `sk-ant-` 시작 | `/v1/messages` (작은 ping) |
| Gemini | `AIzaSy` + 39자 | `/v1beta/models` |

### Anthropic 모델 ID 처리
실제 API에는 날짜 포함 ID(예: `claude-haruki-20251001`) 필요. 드롭다운에는 `formatModelName()`으로 날짜만 잘라 표시.

### Gemini JSON 강제
`responseMimeType: "application/json"` (OpenAI의 `response_format`과 동일 효과).

---

## 4. NEIS 글자/바이트 규칙 (절대 정확히 지킬 것)

- **바이트가 진짜 제한** (NEIS 기준). 글자수는 참고용.
- **한글 = 3바이트**, **영문/숫자/특수문자/공백 = 1바이트** (UTF-8)
- 바이트 카운트: `new TextEncoder().encode(str).length`
- 글자 카운트: `[...str].length` (이모지·결합문자 정확히 처리)
- **목표 바이트 초과 즉시 빨간색** (1.0 임계, 5% 여유 없음). 미만이면 녹색.

---

## 5. 프롬프트 시스템 (가장 민감한 영역)

### 절대 준수 규칙 (두 파일 공통 핵심)
1. NEIS 양식 준수, 생기부 어휘만 사용
2. 인성 평가 단정 금지 ("성실하다" 등 직접 평가 표현 금지)
3. 추상적 형용사 남발 금지 — 행동·성과·결과로 표현
4. 학습 내용 실재성 (helper2 세특 전용) — 과목 개념을 임의 창작 금지
5. 학생 익명성 — 이름·반·번호 노출 금지
6. 동일 문구 반복 금지
7. 학생 수준에 맞는 어휘 선택
8. 결과 중심이 아닌 과정·성장 서술

### 학생 수준 / 성취도 가이드
- helper1: **상/중/하** — 어휘 난이도와 활동 깊이 다름
- helper2: **A/B/C/D/E** — 학업 성취 정도에 맞는 표현 강도
- 두 파일 모두 레벨별 권장 표현 사전을 내장

### 금지/주의 어휘 (이전 세션에서 확정)
- `'탁월한'` — 권장에서 제외, `'심층적으로'` 등으로 대체
- `'성실히'`, `'성실하다'` — 직접 평가 → `'꾸준히 참여하여'`, `'책임감 있게 수행하여'` 등 행동 중심으로 대체

### helper2 세특 과목군
`getSubjectGroup()`이 입력 과목명을 17개 군(국어/수학/영어/사회/과학/체육/예술/기술가정/제2외국어/한문/교양/직업/...)으로 분류해 군별 어휘를 적용.

---

## 6. UI / 반응형

### 데스크톱 (901px+)
원본 카드 레이아웃 유지. 손대지 않는다.

### 모바일/태블릿 (≤900px)
- 카드 레이아웃 단순화
- `position: absolute` → `position: static` 전환 (카드 내 푸터 영역)
- 입력 필드 스택 정렬
- 추가 작은 화면 분기점: 480px

### 흔히 빠지는 함정
- `<col>` 요소에 `calc()` 혼합 단위 → 작동 안 함. 단순 % 사용.
- `<td>`에 `display: flex` 직접 → `display: table-cell` 무효화. 래퍼 div 사용.
- 테이블 셀 안 `height: 100%` → 비신뢰. `position: absolute` + `padding-bottom`으로 푸터 핀.

---

## 7. 푸터 (두 파일 동일)

```html
<div class="footer">
  선생님의 소중한 퇴근 시간을 응원합니다 &nbsp;|&nbsp;
  <a href="mailto:jerry5026@gmail.com" class="footer-email">jerry5026@gmail.com</a>
  <div class="footer-sub">
    마지막 업데이트 <span id="lastUpdate"></span> · vX.Y.Z · 계속 다듬어 나가는 중입니다
  </div>
</div>
<script>
(function(){
  const d = new Date(document.lastModified);
  const s = `${d.getFullYear()}.${String(d.getMonth()+1).padStart(2,'0')}.${String(d.getDate()).padStart(2,'0')}.`;
  const el = document.getElementById('lastUpdate');
  if (el) el.textContent = s;
})();
</script>
```

- **날짜**: `document.lastModified` 자동 (수정 불필요)
- **버전**: 하드코딩 — 아래 9번 규칙대로 자동 증가

---

## 8. 배포 흐름

```
워크트리(claude/modest-jang)에서 작업
    ↓ commit & push
원격 워크트리 브랜치 갱신
    ↓ main 체크아웃 → fast-forward merge → push
GitHub Pages 자동 빌드/배포 (1~2분)
```

명령 시퀀스:
```bash
# 워크트리에서
git add -A && git commit -m "..." && git push

# 메인 리포에서
cd "../../.."  # 또는 절대경로로 메인 디렉토리
git checkout main && git merge claude/modest-jang && git push
```

GitHub 토큰은 사용자가 직접 관리. 자동 푸시 가능.

---

## 9. 버전 자동 증가 규칙 (SemVer)

수정 작업마다 `helper1.html`, `helper2.html` 두 파일의 푸터 `vX.Y.Z`를 자동 증가시킨다.

| 변경 종류 | 예시 | 변화 |
|---|---|---|
| **메이저** | 프롬프트 구조 전면 개편, 핵심 기능 영역 교체, 사용 흐름 변경 | `1.x.x` → `2.0.0` |
| **마이너** | 새 기능/항목/옵션 추가 (학생 수준, 멀티 프로바이더 등) | `1.0.x` → `1.1.0` |
| **패치** | 오타 수정, 색상/문구 조정, 작은 버그, 리팩터링 | `1.0.0` → `1.0.1` |

### 운영 원칙
- 두 파일은 **항상 같은 버전**
- 사용자 향 변경이 아닌 메타 작업(CLAUDE.md, README 등)은 버전 올리지 않음
- 작업 보고 시 "이번 변경은 패치라 v1.0.0 → v1.0.1로 올렸습니다" 식으로 함께 알림

---

## 10. 사용자 의사소통 규칙

- **언어**: 한국어
- **확인 우선**: 큰 변경(프롬프트 수정, 구조 변경)은 먼저 의견 제시 → 사용자 승인 후 작업
- **변경 위치 명시**: "helper1.html 1234번째 줄의 X 함수를 Y로 수정" 식으로 정확히
- **사용자가 "수정하지 말고 알려줘"라고 하면 절대 코드 수정 금지** — 분석/설명만
- **오타/모순 발견 시 임의 수정 금지** — 먼저 알리고 확인받기
- **사용 설명서 동기화**: 새 기능을 추가하면 프로그램 내 "사용 방법" 섹션도 함께 업데이트

---

## 11. 자주 쓰이는 함수/심볼 (빠른 위치 참조)

helper1.html / helper2.html 공통 (이름은 같으나 구현 약간 다를 수 있음):
- `S` — 전역 상태 객체 (localStorage 동기화)
- `validateApiKey(provider, key)` — 키 형식 + 실 ping 검증
- `saveApiSettings()` — S.api 저장
- `switchActiveProvider(provider)` — 활성 프로바이더 전환
- `openApiModal()`, `openApiModalFor(provider)` — API 설정 모달
- `selectProv(provider)`, `populateModels()` — 모달 내 선택
- `renderProviderDropdown()`, `toggleProviderDropdown()` — 헤더 드롭다운
- `formatModelName(id)` — Anthropic 날짜 접미사 제거
- `parseResult(text)` — AI 응답 파싱
- helper2 전용: `getSubjectGroup(subjectName)` — 17 과목군 매핑

---

## 12. 알려진 제약 / 주의

- API 키는 `localStorage`에만 저장 (서버 전송 없음). 브라우저 변경 시 재입력 필요.
- 프롬프트 본문은 매우 길고 상호 참조가 많음 — 한 곳 수정 시 다른 곳과 모순되지 않는지 반드시 검토.
- 두 파일 사이 공통 로직 변경은 **항상 양쪽 동시에** 적용.
- 한글 파일명 URL은 영구 보존 (구 링크 깨지지 않도록).

---

## 13. 향후 개선 후보 (참고만, 자동 적용 금지)

- 프롬프트 A/B 테스트 인프라
- 생성 결과 히스토리 더 길게 (현재 ↩ 이전 버전 모달은 단일 단계)
- 과목별 좋은 예시 문장 사전 확장
- 출력 후 자동 바이트 초과 시 압축 재생성 옵션
- 다크 모드
- PWA화 (오프라인 사용)

이 항목들은 사용자가 명시 요청하기 전엔 건드리지 않는다.
