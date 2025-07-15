---
title: CSP style-src unsafe-inline 제거 여정 - 금융권 보안 감사 대응기
date: 2024-03-15 14:30:00 +0900
categories: [Frontend, Security]
tags: [csp, security, vue, compliance, finance]
---

금융권 보안 감사에서 CSP(Content Security Policy) 정책 위반이 지적되면서 `unsafe-inline`을 제거하는 여정을 시작했다. Quasar, Vue 컴포넌트, 외부 CDN까지 모든 인라인 스타일을 제거한 경험을 공유한다.

## 문제 발생 배경

금융 시스템 보안 감사 과정에서 CSP 정책 위반이 발견됐다. 특히 `style-src 'unsafe-inline'` 사용에 대한 강력한 제재가 들어왔다.

### 초기 CSP 설정
```
Content-Security-Policy: default-src 'self'; style-src 'self' 'unsafe-inline';
```

### 발생한 문제들
1. **콘솔 오류**: `Refused to apply inline style because it violates CSP directive`
2. **외부 CDN 차단**: CKEditor, Daum 주소검색 등 외부 리소스 차단
3. **지시어 오타**: `frame-ancestor` → `frame-ancestors` (복수형)
4. **레거시 페이지**: 일부 페이지에서 CSP 헤더 누락

## 1단계: 기본 CSP 이슈 해결

### 인라인 스타일 해시 방식 도입

Quasar 프레임워크는 컴포넌트 내부에서 스타일 바인딩을 자주 사용한다. 이를 해결하기 위해 해시 방식을 도입했다.

```javascript
// 컴포넌트에서 동적 스타일 생성
const dynamicStyle = computed(() => ({
  width: `${progress.value}%`,
  backgroundColor: isActive.value ? '#1976d2' : '#e0e0e0'
}))
```

이런 스타일들의 해시값을 미리 계산해서 CSP 헤더에 포함했다.

```
Content-Security-Policy: style-src 'self' 'sha256-abc123...' 'sha256-def456...';
```

### 외부 CDN 대응

외부 CDN을 사용하는 라이브러리들을 npm self-host 방식으로 변경했다.

```javascript
// Before: CDN 방식
<script src="https://cdn.ckeditor.com/4.16.0/standard/ckeditor.js"></script>

// After: npm 설치 후 로컬 번들링
npm install ckeditor4
import 'ckeditor4/ckeditor.js'
```

## 2단계: unsafe-inline 제거 압박과 대응

### 금융권 규제 요구사항
- 금감원 보고 항목에 CSS inline 허용 금지 명시
- 감사팀 요구: "문제 없다는 테스트 증적 제출"
- 개선 불가 시 "사장 책임 각서" 필요

### 보안 테스트 증적 준비

XSS 공격 시나리오를 직접 테스트하고 방어 증적을 수집했다.

```javascript
// 테스트 1: 사용자 입력에 악성 스크립트 삽입
const userInput = '<script>alert("XSS")</script><style>body{display:none}</style>'

// 결과: Vue의 템플릿 바인딩으로 HTML 이스케이프됨
// 화면 출력: &lt;script&gt;alert("XSS")&lt;/script&gt;
```

```javascript
// 테스트 2: DOM 조작을 통한 인라인 스타일 삽입 시도
document.getElementById('target').style.cssText = 'display: none !important'

// 결과: CSP 정책에 의해 차단됨
// 콘솔: Refused to apply inline style
```

## 3단계: 최종 보안 증적 수집

### 테스트 시나리오별 캡처

1. **Vue DOM 구조 확인**
   - F12 개발자 도구에서 `<div id="app">` 내부 구조 확인
   - 모든 스타일이 외부 CSS 파일에서 로드됨을 증명

2. **XSS 공격 시나리오 테스트**
   - 사용자 입력 필드에 악성 스크립트 삽입
   - HTML 이스케이프되어 무력화됨을 캡처

3. **CSP 정책 위반 테스트**
   - 외부 스크립트 로드 시도
   - 콘솔에 `Refused to load the script` 메시지 확인

### 감사팀 보고 자료

```json
{
  "csp_policy": "default-src 'self'; script-src 'self' 'sha256-...'; style-src 'self' 'sha256-...'; frame-ancestors 'none';",
  "unsafe_inline_usage": "제거 완료",
  "xss_protection": "Vue 템플릿 바인딩 + CSP 이중 차단",
  "external_resources": "모든 CDN 제거, npm self-host 변경",
  "test_results": "모든 XSS 시나리오 차단 확인"
}
```

## 최종 결과

### 보안 강화 효과
- `unsafe-inline` 완전 제거
- 모든 XSS 공격 벡터 차단
- 외부 CDN 의존성 제거
- 금융권 보안 감사 통과

### 시스템 안정성 향상
- Vue 컴포넌트 안정적 렌더링
- 외부 의존성 제거로 안정성 향상
- 사용자 입력 검증 강화

### 성능 및 유지보수성
- 외부 CDN 제거로 로딩 속도 향상
- 모든 리소스 버전 통합 관리
- 해시 기반 캐싱 최적화

## 개발 후기

처음에는 "그냥 `unsafe-inline` 넣고 넘어가자"는 생각이었지만, 실제로 모든 인라인 스타일을 제거하고 나니 코드 품질이 훨씬 향상됐다.

Quasar 프레임워크의 CSP 대응 기능을 활용하고, 외부 CDN 의존성을 제거하면서 전체적인 시스템 안정성도 개선됐다.

금융권 보안 감사는 까다롭지만, 이를 통해 보안에 대한 전반적인 이해도가 높아졌다. 앞으로도 모든 프로젝트에서 CSP를 기본으로 적용할 예정이다.

무엇보다 "보안은 타협할 수 없다"는 것을 다시 한 번 깨달았다. 처음부터 보안을 고려한 프론트엔드 아키텍처 설계의 중요성을 절감했다.