# String Key Tool

다국어(i18n) 문자열 키를 팀 전체가 일관되게 관리하기 위한 도구예요.  
Figma 플러그인 + 웹 앱으로 구성되며, Tolgee를 Source of Truth로 사용해요.

---

## 목차

1. [아키텍처](#아키텍처)
2. [키 네이밍 규칙](#키-네이밍-규칙)
3. [config.json 관리 정책](#configjson-관리-정책)
4. [설치 방법](#설치-방법)
5. [기능 설명](#기능-설명)
6. [저장소 정책](#저장소-정책)
7. [업데이트 방법](#업데이트-방법)

---

## 아키텍처

```
┌─ Figma Plugin (로컬 설치) ───────────────────────────────────┐
│  manifest.json                                              │
│  code.js     ← Figma API, 레이어 수집, PNG export           │
│  ui.html     ← iframe 브릿지 (postMessage 중계)             │
│    └── <iframe src="GitHub Pages URL">                      │
│          └── index.html  ← 실제 앱 UI                       │
└─────────────────────────────────────────────────────────────┘

┌─ GitHub Repo ───────────────────────────────────────────────┐
│  index.html      ← 앱 UI (GitHub Pages 자동 배포)           │
│  config.json     ← 팀 공유 설정                              │
│  .github/workflows/deploy.yml  ← 자동 배포                  │
└─────────────────────────────────────────────────────────────┘

┌─ Tolgee (Source of Truth) ──────────────────────────────────┐
│  모든 문자열 키, 번역값, 스크린샷 저장                         │
│  Description으로 Figma 화면 정보 관리                         │
└─────────────────────────────────────────────────────────────┘
```

### 메시지 흐름

```
Figma 캔버스 이벤트
      ↓
code.js (Figma 샌드박스)
      ↓ postMessage
ui.html (브릿지)
      ↓ postMessage
index.html (앱 UI, GitHub Pages iframe)
      ↓ fetch
Tolgee API / Claude API (선택)
```

---

## 키 네이밍 규칙

### 기본 구조

```
{screen}.{section?}.{component}.{element?}.{state?}
```

- 구분자: `.` (점)
- 표기법: **camelCase** (세그먼트 내부)
- `?` 표시는 선택 세그먼트
- 세그먼트: 최소 2단계, 최대 5단계

### 세그먼트 정의

#### screen (필수)
IA 기준 화면 식별자. `config.json`의 `screenList`에 등록된 값만 사용.

#### section (선택)
한 화면에 논리적 구역이 여러 개일 때 사용.
```
home.header
settings.list
```

#### component (필수)
UI 요소 종류. 아래 값만 허용.

| 값 | 의미 |
|---|---|
| `btn` | 버튼 |
| `label` | 텍스트, 제목, 설명 |
| `input` | 입력 필드 |
| `toast` | 토스트 |
| `snackbar` | 스낵바 |
| `modal` | 모달, 팝업, 다이얼로그 |
| `tab` | 탭 |
| `chip` | 칩 |
| `banner` | 배너 |
| `segmented` | 세그먼티드 컨트롤 |
| `sheet` | 바텀시트 |
| `list` | 리스트 |
| `card` | 카드 |
| `image` | 이미지 |
| `link` | 링크 |
| `switch` | 스위치/토글 |
| `checkbox` | 체크박스 |
| `radio` | 라디오 |

#### element (선택)
컴포넌트 내 역할.

| 값 | 의미 |
|---|---|
| `search` | 검색 |
| `notification` | 알림 |
| `password` | 비밀번호 |
| `title` | 제목 |
| `description` | 설명 |
| `cta` | 주요 진행 액션 (다음, 시작, 완료) |
| `save` | 저장 |
| `cancel` | 취소 |
| `confirm` | 확인 |
| `delete` | 삭제 |
| `close` | 닫기 |
| `back` | 이전/뒤로 |

#### state (선택)
상태나 변형.

| 값 | 의미 |
|---|---|
| `error` | 오류 |
| `empty` | 빈 상태 |
| `loading` | 로딩 |
| `success` | 성공 |
| `disabled` | 비활성 |
| `placeholder` | 입력 안내 |
| `hint` | 힌트/도움말 |
| `pressed` | 눌린 상태 |
| `focused` | 포커스 상태 |
| `selected` | 선택된 상태 |

### 예시

| 상황 | 키 |
|---|---|
| 홈 헤더 검색 버튼 | `home.header.btn.search` |
| 설정 알림 목록 비활성 | `settings.list.label.notification.disabled` |
| 로그인 비밀번호 입력 오류 | `login.input.password.error` |
| 공통 확인 버튼 | `common.btn.confirm` |
| 공통 취소 버튼 | `common.btn.cancel` |
| 공통 네트워크 에러 | `common.label.error` |
| 온보딩 이름 입력 placeholder | `onboarding.nameInput.input.placeholder` |

### 공통(common) 처리 원칙

- 3개 이상의 화면에서 동일한 문자열이 사용되면 `common`으로 이동
- "거의 같지만 약간 다른" 문자열은 각 화면에 별도로 두는 것을 권장 (번역 분기 대비)

### 플랫폼별 키 변환 규칙

| 플랫폼 | 변환 | 예시 |
|---|---|---|
| Tolgee (SoT) | dot 그대로 | `home.header.btn.search` |
| iOS | dot 그대로 | `home.header.btn.search` |
| Android | `.` → `_` | `home_header_btn_search` |

### 금지 사항

```
❌ okButton          (screen.component 구조 없이 camelCase만 사용 금지)
❌ ok-button         (하이픈 금지)
❌ OK_BTN            (대문자 금지)
❌ btn               (screen 없이 단독 사용 금지)
❌ onboarding.1.btn  (숫자 단독 세그먼트 금지)
```

---

## config.json 관리 정책

`config.json`은 팀 공유 설정 파일이에요. GitHub에 커밋되어 모든 구성원이 동일한 설정을 사용해요.  
앱 시작 시 GitHub Pages에서 자동으로 불러와요. (Token 불필요, 읽기 전용)

### 구조

```json
{
  "featureMap": {
    "1": "회원가입",
    "3": "온보딩"
  },
  "screenList": [
    "common", "onboarding", "home", "profile"
  ],
  "tolgee": {
    "url": "https://your-tolgee-server.com",
    "projectId": "3",
    "langs": ["ko-KR", "en-US", "ja-JP"]
  },
  "github": {
    "owner": "your-org",
    "repo": "string-key-tool",
    "branch": "main"
  },
  "keyRules": {
    "component": { "btn": ["버튼", "클릭", "..."], "...": [] },
    "element":   { "cta": ["다음", "시작", "..."], "...": [] },
    "state":     { "error": ["오류", "실패", "..."], "...": [] }
  }
}
```

### langs 설정

`tolgee.langs` 배열의 첫 번째 언어가 기본 언어예요.  
키 등록 시 기본 언어만 입력값으로 저장되고, 나머지는 빈 슬롯으로 생성돼요.  
Claude API Key가 있으면 등록 시 자동 번역해서 모든 언어를 채워요.

### 수정 권한

| 항목 | 수정 주체 | 방법 |
|---|---|---|
| `featureMap`, `screenList` | 기획/개발 | 플러그인 화면 관리 탭 또는 직접 PR |
| `keyRules` | 개발 | 직접 PR (팀 리뷰 권장) |
| `tolgee`, `github` | 개발/인프라 | 직접 PR |

### 개인 보관 항목 (config.json에 절대 포함 금지)

로컬 브라우저 `localStorage` 및 Figma `clientStorage`에만 저장돼요.

- Tolgee API Key (필수)
- Claude API Key (선택)

---

## 설치 방법

### 1. 레포 준비

```bash
git clone https://github.com/{owner}/{repo}.git
```

### 2. config.json 수정

```json
{
  "tolgee": {
    "url": "https://your-tolgee.com",
    "projectId": "3",
    "langs": ["ko-KR", "en-US", "ja-JP"]
  },
  "github": {
    "owner": "your-org",
    "repo": "string-key-tool",
    "branch": "main"
  }
}
```

### 3. GitHub Pages 활성화

GitHub 레포 → Settings → Pages → Source: **GitHub Actions**  
`main` 브랜치에 push하면 자동 배포돼요.

### 4. ui.html URL 교체

```html
<script>
  document.getElementById('app-frame').src =
    'https://your-org.github.io/string-key-tool/?t=' + Date.now();
</script>
```

### 5. Figma 플러그인 설치

Figma 데스크탑 앱 → Plugins → Development → **Import plugin from manifest**  
→ `manifest.json` 선택

### 6. 개인 설정 (최초 1회)

플러그인 실행 → 설정 탭에서:

| 항목 | 필수 | 설명 |
|---|---|---|
| Tolgee API Key | 필수 | 키 조회·등록용 |
| Claude API Key | 선택 | 키 추천·자동 번역. 없으면 규칙 기반 동작 |

저장 후 **↻ 지금 동기화**를 눌러 config.json을 불러오면 준비 완료예요.

> Figma 플러그인에서 입력한 API Key는 `clientStorage`에 저장되어 껐다 켜도 유지돼요.

---

## 기능 설명

### 탭 구성

| 탭 | 환경 | 설명 |
|---|---|---|
| 키 설정 | 공통 | 키 검색·등록 |
| 언어 변경 | Figma 전용 | 프레임 텍스트를 다른 언어로 교체 |
| 프레임 | Figma 전용 | 프레임 이름 일괄 변경 |
| 화면 관리 | 공통 | 화면 목록·기능 이름 매핑 |
| 설정 | 공통 | API Key·유사도 임계값 |

---

### 키 설정 탭

#### 문자열 검색 모드

1. 문자열 직접 입력 또는 Figma에서 텍스트 레이어 선택 (자동 입력)
2. **유사 문자열 검색** 클릭 → Tolgee 키 목록에서 유사도 계산
   - 부분 포함 검색 지원 (짧은 검색어로 긴 문자열도 탐지)
   - 재사용 추천: 기본 85% 이상
   - 유사: 기본 50% 이상
3. **신규 키 추천** 카드 (이미 존재/신규 뱃지 포함)
   - 규칙 기반 후보 즉시 표시
   - Claude API Key 있으면 레이어 맥락(부모/컴포넌트/형제) 기반으로 백그라운드 개선
4. 키 선택 또는 직접 입력 후 **Tolgee 등록**
   - Claude API Key 있으면 모든 언어 자동 번역
   - ☑ 레이어 이름을 키값으로 변경 (Figma 전용)
   - ☑ 현재 프레임 스크린샷 첨부 — 텍스트 위치에 빨간 박스 하이라이트 포함 (Figma 전용)
5. Description에 Figma 화면 이름 자동 누적 기록

#### 화면 내 문구 등록 모드

1. Figma에서 프레임·컴포넌트·인스턴스·슬롯 선택 (눈이 꺼진 레이어 자동 제외)
2. **화면 내 문구 등록** 탭 클릭 → 내부 텍스트 자동 수집
3. 진행도 표시와 함께 각 문자열에 키 자동 추천 (중복 방지)
   - 이미 Tolgee에 등록된 문자열은 주황색 배지로 표시
4. 개별 수정 후 등록 또는 **전체 등록**
5. 각 항목 등록 시 레이어 이름 변경 + 스크린샷 첨부 + 자동 번역 동작

---

### 언어 변경 탭 (Figma 전용)

1. Figma에서 프레임·컴포넌트 선택
2. 교체할 언어 선택
3. **교체 적용** 클릭 → Tolgee에서 번역값 로드 후 즉시 적용
   - 레이어 이름이 키값인 텍스트를 모두 교체
   - 한→일→영 연속 교체 가능

---

### 프레임 탭 (Figma 전용)

화면 번호 체계: `{기능}.{버전}.{순서}`

```
3.1.1  →  기능3 / 버전1 / 1번 화면
3.1.2  →  기능3 / 버전1 / 2번 화면
```

1. Figma에서 프레임 여러 개 선택
2. 기능번호·버전 입력
3. 미리보기 확인 (X좌표 기준 좌→우 정렬)
4. **선택된 프레임 이름 변경**

---

### 화면 관리 탭

- **화면 목록** — 키 탭의 화면 드롭다운 선택지 관리
- **기능 이름 매핑** — 기능번호 → 이름 매핑 (예: `3` → `온보딩`)

---

### 설정 탭

| 항목 | 저장 위치 | 설명 |
|---|---|---|
| 화면 모드 | localStorage | 라이트/다크 (기본: 라이트) |
| Claude API Key | localStorage + clientStorage | 키 추천·자동 번역 (선택) |
| Tolgee API Key | localStorage + clientStorage | 키 조회·등록 (필수) |
| 유사도 임계값 | 세션 | 재사용/유사 기준 조정 |

> Tolgee URL·ProjectID, GitHub Owner·Repo·Branch는 config.json에서 자동으로 불러와요.  
> **↻ 지금 동기화** 버튼으로 수동 갱신도 가능해요.

---

## 저장소 정책

### Tolgee가 Source of Truth

- 모든 키는 Tolgee에 등록 후 각 플랫폼으로 동기화
- 플랫폼 파일에서 직접 키를 추가하지 않는 것을 원칙으로 함

### 다국어 등록 방식

```
ko-KR → 입력한 문자열 값으로 등록
en-US → 자동 번역 (Claude API Key 있을 때) 또는 빈 슬롯
ja-JP → 자동 번역 (Claude API Key 있을 때) 또는 빈 슬롯
```

### Figma 화면 정보 매핑

Figma 화면 이름은 Tolgee 키의 **Description**에 누적 저장해요.  
기존 description이 있으면 새 화면 이름을 추가하고 알파벳/숫자 순으로 정렬해요.

```
키:          onboarding.nameInput.btn.cta
Description: Figma: 3.1.1, 3.1.2
```

### 스크린샷

키 등록 시 해당 텍스트가 속한 최상위 프레임을 PNG로 캡처해서 Tolgee에 첨부해요.  
**텍스트 노드 위치에 빨간 박스**가 이미지에 직접 그려져서 어떤 문자열인지 한눈에 확인할 수 있어요.  
여러 프레임을 선택하면 프레임 수만큼 스크린샷이 각각 첨부돼요.

---

## 업데이트 방법

### 앱 UI (index.html, config.json)

```bash
git add index.html config.json
git commit -m "feat: 설명"
git push origin main
# GitHub Actions 자동 배포 (약 1~3분)
# Figma 플러그인 재실행 시 자동으로 최신 버전 로드
```

### 플러그인 파일 (code.js, ui.html, manifest.json)

로컬 파일 교체 후 Figma에서 플러그인 재실행.

---

## 기여 가이드

- `keyRules` 변경은 팀 리뷰 후 반영
- 새 screen은 화면 관리 탭에서 추가 후 config.json 직접 커밋
- 새 언어는 `config.json`의 `tolgee.langs` 배열에 추가
- 키 네이밍 규칙 논의는 이슈로 관리
