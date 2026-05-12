# String Key Tool — Figma Plugin

다국어 문자열 키 관리 Figma 플러그인.

## 구조

```
figma-plugin/   ← Figma에 로컬 설치 (한 번만)
  manifest.json
  code.js       ← Figma API (clientStorage, 프레임 조작)
  ui.html       ← 얇은 iframe 브릿지

index.html      ← 실제 앱 UI (GitHub Pages로 자동 배포)
```

## 설치 방법

### 1. GitHub Pages 배포
1. 이 레포를 fork 또는 clone
2. GitHub → Settings → Pages → Source: `GitHub Actions`
3. `main` 브랜치에 push하면 자동 배포
4. 배포 URL 확인: `https://{username}.github.io/{repo-name}/`

### 2. ui.html URL 교체
`figma-plugin/ui.html` 파일에서 `GITHUB_PAGES_URL`을 실제 URL로 교체:
```html
<iframe src="https://your-username.github.io/string-key-tool/" ...>
```

### 3. Figma 플러그인 설치
Figma 데스크탑 앱 → Plugins → Development → Import plugin from manifest
→ `figma-plugin/manifest.json` 선택

## 업데이트 방법

`index.html` 수정 후 `main`에 push → GitHub Actions가 자동 배포.
Figma 플러그인은 재설치 불필요.

## 기능

- **문자열 키 탭**: 문자열 입력 → Tolgee 유사 키 검색 → 재사용/신규 등록
- **프레임 이름 탭**: 선택한 프레임을 `기능.버전.순서` 형식으로 일괄 변경
- **설정 탭**: Tolgee API 연결, 화면 목록 관리, 기능 이름 매핑 (영구 저장)
