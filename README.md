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
8. [Tolgee 태그 정책](#tolgee-태그-정책)

---

## 아키텍처

```
┌─ Figma Plugin (로컬 설치, 한 번만) ─────────────────────────┐
│  manifest.json                                              │
│  code.js     ← Figma API, clientStorage, 레이어 수집        │
│  ui.html     ← 얇은 iframe 브릿지 (postMessage 중계)        │
│    └── <iframe src="GitHub Pages URL">                      │
│          └── index.html  ← 실제 앱 UI                       │
└─────────────────────────────────────────────────────────────┘

┌─ GitHub Repo ───────────────────────────────────────────────┐
│  index.html      ← 앱 UI (GitHub Pages 자동 배포)           │
│  config.json     ← 팀 공유 설정 (키 규칙, 화면 목록 등)     │
│  .github/
│    workflows/
│      deploy.yml  ← main push 시 GitHub Pages 자동 배포      │
└─────────────────────────────────────────────────────────────┘

┌─ Tolgee (Source of Truth) ──────────────────────────────────┐
│  모든 문자열 키와 번역값의 단일 저장소                        │
│  태그로 메타데이터 관리 (figma:3.1.1, lint:passed)           │
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
Tolgee API / Claude API
```

---

## 키 네이밍 규칙

### 기본 구조

```
{screen}.{section?}.{component}.{element?}.{state?}
```

- 구분자: `.` (점)
- 표기법: `snake_case`
- 영문·숫자·밑줄만 허용
- 세그먼트: 최소 2단계, 최대 5단계

### 세그먼트 정의

#### screen (필수)
IA 기준 화면 식별자. `config.json`의 `screenList`에 등록된 값만 사용.

| 값 | 설명 |
|---|---|
| `common` | 여러 화면에서 공통으로 쓰이는 문자열 |
| `onboarding` | 온보딩 플로우 |
| `home` | 홈 피드 |
| `profile` | 프로필 |
| `settings` | 설정 |
| `payment` | 결제 |

> 새 화면 추가는 플러그인 설정 탭 → 화면 목록에서 관리

#### section (선택)
한 화면에 논리적 구역이 여러 개일 때 사용.
```
onboarding.name_input
onboarding.email_input
```

#### component (필수)
UI 요소 종류. 아래 약어만 사용.

| 약어 | 의미 |
|---|---|
| `btn` | 버튼 |
| `label` | 텍스트, 제목, 설명 |
| `input` | 입력 필드 |
| `toast` | 토스트 메시지 |
| `modal` | 모달 |
| `tab` | 탭 |

#### element (선택)
컴포넌트 내 역할.

| 값 | 의미 |
|---|---|
| `cta` | 주요 진행 액션 (다음, 시작, 완료) |
| `save` | 저장 |
| `cancel` | 취소 |
| `confirm` | 확인/동의 |
| `delete` | 삭제 |
| `placeholder` | 입력 필드 안내 텍스트 |
| `title` | 제목 |
| `description` | 설명 |

#### state (선택)
상태나 변형.

| 값 | 의미 |
|---|---|
| `error` | 오류 상태 |
| `empty` | 빈 상태 |
| `loading` | 로딩 상태 |
| `success` | 성공 상태 |
| `disabled` | 비활성 상태 |

### 예시

| 상황 | 키 |
|---|---|
| 온보딩 이름 입력 화면 제목 | `onboarding.name_input.label.title` |
| 온보딩 다음 버튼 | `onboarding.name_input.btn.cta` |
| 이름 미입력 에러 메시지 | `onboarding.name_input.input.error` |
| 홈 피드 빈 상태 | `home.feed.label.empty` |
| 결제 실패 토스트 | `payment.toast.error` |
| 공통 확인 버튼 | `common.btn.confirm` |
| 공통 네트워크 에러 | `common.label.error` |

### 공통(common) 처리 원칙

- 3개 이상의 화면에서 **동일한 문자열**이 사용되면 `common`으로 이동
- "거의 같지만 약간 다른" 문자열은 각 화면에 별도로 두는 것을 권장 (번역 분기 대비)
- 공통 버튼(확인, 취소, 저장)은 원칙적으로 `common` 사용

### 금지 사항

```
❌ okButton          (camelCase 금지)
❌ ok-button         (하이픈 금지)
❌ OK_BTN            (대문자 금지)
❌ btn               (screen 없이 단독 사용 금지)
❌ onboarding.1.btn  (숫자 단독 세그먼트 금지)
```

---

## config.json 관리 정책

`config.json`은 팀 공유 설정 파일이에요. GitHub에 커밋되어 모든 구성원이 동일한 설정을 사용해요.

### 구조

```json
{
  "featureMap": {
    "1": "회원가입",
    "2": "홈 피드",
    "3": "온보딩"
  },
  "screenList": [
    "common", "onboarding", "home", "profile"
  ],
  "tolgee": {
    "url": "https://your-tolgee-server.com",
    "projectId": "3"
  },
  "github": {
    "owner": "your-org",
    "repo": "string-key-tool",
    "branch": "main"
  },
  "keyRules": {
    "component": {
      "btn":   ["버튼", "클릭", "확인", "취소", "저장", "다음"],
      "input": ["입력", "검색어"],
      "toast": ["토스트", "알림"]
    },
    "element": {
      "cta":    ["다음", "시작", "완료"],
      "save":   ["저장"],
      "cancel": ["취소"]
    },
    "state": {
      "error":   ["오류", "실패"],
      "empty":   ["없어요", "비어"],
      "loading": ["로딩"]
    }
  }
}
```

### 수정 권한

| 항목 | 수정 주체 | 방법 |
|---|---|---|
| `featureMap` | 기획/개발 | 플러그인 설정 탭 또는 직접 PR |
| `screenList` | 기획/개발 | 플러그인 설정 탭 또는 직접 PR |
| `keyRules` | 개발 | 직접 PR (코드 리뷰 권장) |
| `tolgee`, `github` | 개발/인프라 | 직접 PR |

### 개인 보관 항목 (config.json에 절대 포함 금지)

아래 항목은 각자의 브라우저 `localStorage`에만 저장돼요.

- Tolgee API Key (`tgpak_...`)
- GitHub Personal Access Token (`ghp_...`)
- Claude API Key (`sk-ant-...`)

---

## 설치 방법

### 1. 레포 준비

```bash
git clone https://github.com/{owner}/{repo}.git
cd {repo}
```

### 2. config.json 수정

`config.json`에서 실제 값으로 교체:
```json
{
  "tolgee": { "url": "https://your-tolgee.com", "projectId": "3" },
  "github": { "owner": "your-org", "repo": "string-key-tool", "branch": "main" }
}
```

### 3. GitHub Pages 활성화

GitHub 레포 → Settings → Pages → Source: **GitHub Actions**  
`main` 브랜치에 push하면 자동 배포돼요.

### 4. ui.html URL 교체

`ui.html`에서 `GITHUB_PAGES_URL`을 실제 배포 URL로 교체:
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

플러그인 설정 탭에서:
- **Tolgee API Key** 입력 → 저장
- **GitHub Personal Access Token** 입력 → 저장
  - 권한: `repo` (config.json 업데이트용)
- **Claude API Key** 입력 → 저장 (선택, 없으면 규칙 기반 추천)

---

## 기능 설명

### 문자열 키 탭

#### 단일 검색 모드
1. 화면(screen) 선택
2. 문자열 입력 또는 Figma에서 텍스트 레이어 선택 (자동 입력)
3. **유사 문자열 검색** — Tolgee 키 목록에서 유사도 계산
   - 재사용 추천 (기본 85% 이상): 기존 키 재사용
   - 유사 (기본 50% 이상): 참고용으로 표시
4. 신규 키 추천 — 규칙 기반 또는 Claude API (맥락 포함)
5. Figma 화면 태그 연결 (`figma:3.1.1` 형식)
6. Tolgee 등록

#### 프레임 일괄 모드
1. Figma에서 프레임을 여러 개 선택
2. **프레임 일괄** 버튼 클릭 → 내부 텍스트 자동 수집 + 중복 제거
3. 각 문자열에 키 자동 추천
4. 개별 수정 후 등록, 또는 **전체 등록**

#### 유사도 임계값 조정
- 헤더의 슬라이더로 실시간 조정 가능
- 재사용 임계값: 기본 85% (이 이상이면 "재사용 추천" 뱃지)
- 유사 임계값: 기본 50% (이 이상이면 검색 결과에 표시)

### 프레임 이름 탭

Figma 화면 번호 체계: `{기능}.{버전}.{순서}`

```
3.1.1  →  기능3(온보딩) / 버전1 / 1번 화면
3.1.2  →  기능3(온보딩) / 버전1 / 2번 화면
```

1. Figma에서 프레임 여러 개 선택
2. 기능번호·버전 입력
3. 미리보기 확인 (X좌표 기준 좌→우 자동 정렬)
4. **프레임 이름 변경** 클릭

### 설정 탭

| 항목 | 저장 위치 | 설명 |
|---|---|---|
| Claude API Key | localStorage | AI 키 추천용, 없으면 규칙 기반 |
| Tolgee API URL | config.json (자동) | 팀 공통, 읽기 전용 |
| Tolgee Project ID | config.json (자동) | 팀 공통, 읽기 전용 |
| Tolgee API Key | localStorage | 개인 발급 |
| GitHub Owner/Repo/Branch | config.json (자동) | 팀 공통, 읽기 전용 |
| GitHub Token | localStorage | config.json 업데이트용 |
| 화면 목록 | config.json | 팀 공유, 플러그인에서 수정 가능 |
| 기능 이름 매핑 | config.json | 팀 공유, 플러그인에서 수정 가능 |

---

## 저장소 정책

### Tolgee가 Source of Truth

- 모든 문자열 키는 Tolgee에 등록 후 각 플랫폼(iOS/Android/Server)으로 동기화
- 플랫폼별 파일에서 직접 키를 추가하지 않는 것을 원칙으로 함

### 플랫폼별 네임스페이스

Tolgee namespace = screen 이름으로 통일

```
iOS:     onboarding.strings, home.strings, ...
Android: onboarding.xml, home.xml, ...
Server:  onboarding.json, home.json, ...
```

### Figma 화면 번호 ↔ 개발 키 매핑

Figma 화면 번호는 Tolgee 키 태그로 관리:
```
키: onboarding.name_input.btn.cta
태그: figma:3.1.1, figma:3.1.2, lint:passed
```

---

## 업데이트 방법

### 앱 UI 업데이트 (index.html)

```bash
# index.html 수정 후
git add index.html
git commit -m "feat: 기능 설명"
git push origin main
# → GitHub Actions가 자동 배포 (약 1~3분)
# → Figma 플러그인 재실행 시 자동으로 최신 버전 로드
```

### 플러그인 파일 업데이트 (code.js, ui.html, manifest.json)

로컬 플러그인 파일을 직접 교체 후 Figma에서 플러그인 재실행.  
팀 배포 시에는 각자 파일을 교체해야 해요.

### config.json 업데이트

플러그인 설정 탭에서 화면 목록·기능 이름 수정 후 저장하면 GitHub API로 자동 커밋.  
또는 직접 PR로 수정 가능.

---

## Tolgee 태그 정책

| 태그 | 의미 | 부착 시점 |
|---|---|---|
| `lint:passed` | 키 네이밍 규칙 검증 통과 | 이 도구로 등록 시 자동 |
| `figma:3.1.1` | 연결된 Figma 화면 번호 | 사용자가 수동 연결 |
| `figma:3.1.2` | 동일 키가 여러 화면에 사용될 때 추가 | 사용자가 수동 연결 |

### Lint 규칙 (n8n 자동 검증)

Tolgee에 키가 생성/수정되면 n8n webhook이 발동하여 아래를 검증:

1. **포맷**: `snake_case` + `.` 구분자
2. **세그먼트 수**: 2~5단계
3. **screen**: `config.json`의 `screenList` 허용 목록에 있는지 확인

검증 실패 시 Google Chat으로 알림 발송.

---

## 기여 가이드

- `keyRules` 변경은 팀 리뷰 후 반영
- 새 screen 추가는 플러그인 설정 탭에서 추가 → config.json 자동 커밋
- 키 네이밍 규칙 관련 논의는 이슈로 관리