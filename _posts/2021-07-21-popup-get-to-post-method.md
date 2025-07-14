---
title: 팝업 페이지 GET → POST 방식 변경 구현
date: 2021-07-21 17:43:00 +0900
categories: [Frontend, JavaScript]
tags: [javascript, popup, form, get, post, window-open]
---

GET 방식으로 팝업을 열면 URL에 파라미터가 노출되는 문제가 있었다. POST 방식으로 변경해서 파라미터를 숨기고 보안성을 높였다. `window.open()`과 `form.submit()`을 조합해서 해결했다.

## 변경 전후 비교

### 기존 GET 방식 코드

```javascript
function searchResource(rowIndex){
    var productGb = $('#productGb').val();
    _searchResource(productGb, rowIndex);
}

function _searchResource(productGb, rowIndex) {
    var url = "/pm/prodPlan/popup/findResource?productGb=" + productGb + "&rowIndex=" + rowIndex;
    
    window.open(url, "", "width=700, height=530, toolbar=no, menubar=no, scrollbars=no, resizable=no");
    return;
}
```

**GET 방식의 문제점:**
- URL에 파라미터가 노출됨
- 긴 데이터 전송 시 URL 길이 제한
- 보안상 취약점 존재
- 브라우저 히스토리에 파라미터 기록

### 개선된 POST 방식 코드

```javascript
function searchResource(rowIndex){
    $('#rowIndex').val(rowIndex); 
    
    var pop_title = "popupOpener";
    
    window.open("", pop_title, "width=700, height=530, toolbar=no, menubar=no, scrollbars=no, resizable=no");
    
    var frmData = document.prodForm;
    frmData.target = pop_title;
    frmData.action = "/pm/prodPlan/popup/findResource";
    
    frmData.submit();
}
```

**POST 방식의 장점:**
- URL에 파라미터 노출 안됨
- 대용량 데이터 전송 가능
- 보안성 향상
- RESTful API 설계에 적합

## 구현 과정 상세

### 1. HTML Form 구조

```html
<form name="prodForm" method="POST">
    <input type="hidden" id="rowIndex" name="rowIndex" value="">
    <input type="hidden" id="productGb" name="productGb" value="">
    
    <!-- 기타 필요한 input 태그들 -->
    <button type="button" onclick="searchResource(1)">팝업 열기</button>
</form>
```

**필수 요소:**
- `form` 태그에 `name="prodForm"` 속성 필요
- `method="POST"` 설정
- 전달할 파라미터를 `hidden` input으로 선언

### 2. JavaScript 핵심 로직

```javascript
function searchResource(rowIndex){
    // 1. 동적으로 파라미터 값 설정
    $('#rowIndex').val(rowIndex); 
    
    // 2. 팝업 윈도우 이름 설정
    var pop_title = "popupOpener";
    
    // 3. 빈 팝업 창 열기
    window.open("", pop_title, "width=700, height=530, toolbar=no, menubar=no, scrollbars=no, resizable=no");
    
    // 4. 폼 데이터 설정
    var frmData = document.prodForm;
    frmData.target = pop_title;      // 팝업 창을 타겟으로 설정
    frmData.action = "/pm/prodPlan/popup/findResource";  // 요청 URL
    
    // 5. 폼 제출 (POST 방식)
    frmData.submit();
}
```

### 3. 단계별 실행 과정

1. **파라미터 설정**: `$('#rowIndex').val(rowIndex)`로 hidden input에 값 저장
2. **팝업 창 생성**: `window.open("", pop_title, options)`로 빈 팝업 창 열기
3. **폼 타겟 설정**: `frmData.target = pop_title`로 폼 제출 대상을 팝업 창으로 지정
4. **액션 URL 설정**: `frmData.action = "URL"`로 요청 주소 지정
5. **폼 제출**: `frmData.submit()`으로 POST 방식 전송

## 서버 컨트롤러 처리

### Spring Boot 컨트롤러 구현

```java
@Controller
@RequestMapping("/pm/prodPlan")
public class ProductPlanController {
    
    @RequestMapping("/popup/findResource")
    public String findResource(HttpServletRequest request, Model model) throws Exception {
        
        // POST 방식으로 전송된 파라미터 받기
        model.addAttribute("rowIndex", request.getParameter("rowIndex"));
        model.addAttribute("productGb", request.getParameter("productGb"));
        
        return "/pm/popup/findResource";
    }
}
```

**컨트롤러 특징:**
- GET/POST 방식 모두 동일하게 `request.getParameter()`로 처리
- `@RequestMapping`은 기본적으로 GET, POST 모두 허용
- 필요시 `@PostMapping` 어노테이션으로 POST만 허용 가능

## 구현 체크리스트

### 현재 페이지 설정
- [ ] form 태그에 `name="prodForm"` 속성 추가
- [ ] form 태그에 `method="POST"` 설정
- [ ] 전달할 파라미터를 hidden input으로 선언
- [ ] JavaScript 함수에서 파라미터 값 동적 설정

### 팝업 페이지 설정
- [ ] 컨트롤러에서 파라미터 받기
- [ ] Model에 파라미터 추가하여 JSP에 전달
- [ ] 필요시 파라미터 검증 로직 추가

## 활용 예시

### 다중 파라미터 전송

```javascript
function searchResourceAdvanced(rowIndex, productType, category){
    // 여러 파라미터 설정
    $('#rowIndex').val(rowIndex);
    $('#productGb').val($('#productGb').val());
    $('#productType').val(productType);
    $('#category').val(category);
    
    var pop_title = "popupOpener";
    
    window.open("", pop_title, "width=800, height=600, toolbar=no, menubar=no, scrollbars=yes, resizable=yes");
    
    var frmData = document.prodForm;
    frmData.target = pop_title;
    frmData.action = "/pm/prodPlan/popup/findResourceAdvanced";
    
    frmData.submit();
}
```

### 팝업 옵션 커스터마이징

```javascript
// 반응형 팝업 크기
function getPopupOptions() {
    var width = Math.min(800, screen.width * 0.8);
    var height = Math.min(600, screen.height * 0.8);
    var left = (screen.width - width) / 2;
    var top = (screen.height - height) / 2;
    
    return `width=${width}, height=${height}, left=${left}, top=${top}, toolbar=no, menubar=no, scrollbars=yes, resizable=yes`;
}

function searchResource(rowIndex){
    $('#rowIndex').val(rowIndex);
    
    var pop_title = "popupOpener";
    
    window.open("", pop_title, getPopupOptions());
    
    var frmData = document.prodForm;
    frmData.target = pop_title;
    frmData.action = "/pm/prodPlan/popup/findResource";
    
    frmData.submit();
}
```

## 보안 개선 효과

| 구분 | GET 방식 | POST 방식 |
|------|----------|-----------|
| **URL 노출** | 파라미터 노출 | 파라미터 숨김 |
| **브라우저 히스토리** | 기록됨 | 기록 안됨 |
| **서버 로그** | URL에 파라미터 기록 | Body에 파라미터 전송 |
| **북마크** | 파라미터 포함 | 파라미터 미포함 |
| **데이터 크기** | URL 길이 제한 | 대용량 데이터 가능 |

## 실무 팁

1. **팝업 블로커 대응**: 사용자 액션에 의해 호출되도록 구현
2. **중복 제출 방지**: 폼 제출 후 버튼 비활성화 처리
3. **파라미터 검증**: 서버에서 필수 파라미터 검증 로직 추가
4. **에러 처리**: 팝업 생성 실패 시 대체 방안 마련

## 개발 후기

GET 방식은 URL에 파라미터가 노출되어 보안상 우려가 있었는데, POST로 변경하니 훨씬 안전해졌다.

처음에 `window.open()`에 빈 문자열을 넣는 부분이 이해가 안 됐는데, 팝업 창을 먼저 열고 그 다음에 폼을 제출하는 방식이라는 것을 깨달았다.

`form.target`을 팝업창 이름으로 설정하는 것이 핵심 포인트다. 이렇게 하면 폼 결과가 팝업창에 표시된다.

이 방법을 익히고 나니 다른 팝업들도 POST 방식으로 변경할 수 있을 것 같다. 보안성 측면에서도 더 나은 접근 방식이다.