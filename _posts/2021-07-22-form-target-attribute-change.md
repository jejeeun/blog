---
title: 팝업 사용 후 form submit 시 새 창 열림 문제 해결
date: 2021-07-22 10:45:00 +0900
categories: [Frontend, JavaScript]
tags: [javascript, form, target, popup, submit, troubleshooting]
---

GET을 POST로 변경하는 작업 중에 발생한 버그를 해결했다. 팝업으로 데이터 조회 후 검색 버튼을 누르면 새 창이 열리는 문제가 있었는데, form의 `target` 속성이 팝업창으로 설정된 채 남아있어서 생긴 문제였다.

## 문제 상황

팝업을 사용하는 페이지에서 다음과 같은 순서로 동작할 때:
1. 팝업 열어서 데이터 조회
2. 팝업 닫기
3. 검색 버튼 클릭
4. 예상과 다르게 새 창이 열림

### 문제가 있던 수정 전 코드

```javascript
/*수정 전*/
//조회
function search(){
    searchForm.resursName.value = searchForm.resursName.value.replace(/(^\s*)|(\s*$)/g, ''); //제품명 앞뒤 공백 제거
            
    $("#searchForm").prop('action', '/pm/prodPlan/rescList');
    $("#searchForm").submit();
}

//팝업
function _customSearch(customerGb){
    $('#customerGb').val(customerGb); 
             
    var popupOption = "width=700, height=500, toolbar=no, menubar=no, scrollbars=no, resizable=no";
    window.open('', 'findCompany', popupOption);
            
    $("#searchForm").prop('target', 'findCompany');
    $("#searchForm").prop('action', '/pm/prodPlan/popup/findCompany');
    $("#searchForm").submit();
}
```

### 문제 분석

**문제점은 `target` 속성에 있었습니다!**

1. 팝업창을 열면 `searchForm`의 `target`이 `'findCompany'`(팝업창)를 가리킴
2. 팝업창을 닫은 후에도 `target`이 여전히 `'findCompany'`로 설정된 상태
3. 이후 `search()` 함수로 조회를 하면 새 창이 열림

## 해결 과정

### 1단계: target을 _self로 명시적 설정

```javascript
/*수정 후_ver1*/
//조회
function search(){
    searchForm.resursName.value = searchForm.resursName.value.replace(/(^\s*)|(\s*$)/g, ''); //제품명 앞뒤 공백 제거
            
    $('#searchForm').attr('target', '_self');  // 현재 프레임으로 target 설정
    $("#searchForm").prop('action', '/pm/prodPlan/rescList');
    $("#searchForm").submit();
}

//팝업
function _customSearch(customerGb){
    $('#customerGb').val(customerGb); 
             
    var popupOption = "width=700, height=500, toolbar=no, menubar=no, scrollbars=no, resizable=no";
    window.open('', 'findCompany', popupOption);
            
    $("#searchForm").prop('target', 'findCompany');
    $("#searchForm").prop('action', '/pm/prodPlan/popup/findCompany');
    $("#searchForm").submit();
}
```

### 1단계의 문제점

**수정후_ver1의 문제점이 생겼습니다.**

팝업창을 거친 후 '조회'가 아닌 **또 다른 기능이 생기면 그때마다 `$('#searchForm').attr('target', '_self');`를 적어줘야 하는 문제점**이 발생했습니다.

### 2단계: 최적화된 해결책

**해결은 간단했습니다. 팝업호출 함수 내에 submit이 끝난 후 target을 변경해주면 됩니다.**

```javascript
/*수정 후_ver2 - 최종 해결책*/
//조회
function search(){
    searchForm.resursName.value = searchForm.resursName.value.replace(/(^\s*)|(\s*$)/g, ''); //제품명 앞뒤 공백 제거
            
    $("#searchForm").prop('action', '/pm/prodPlan/rescList');
    $("#searchForm").submit();
}

//팝업
function _customSearch(customerGb){
    $('#customerGb').val(customerGb); 
             
    var popupOption = "width=700, height=500, toolbar=no, menubar=no, scrollbars=no, resizable=no";
    window.open('', 'findCompany', popupOption);
            
    $("#searchForm").prop('target', 'findCompany');
    $("#searchForm").prop('action', '/pm/prodPlan/popup/findCompany');
    $("#searchForm").submit();
    $("#searchForm").prop('target', '_self');  // submit 후 즉시 target 복원
}
```

## Form Target 속성 이해하기

### Target 속성 값들

| 값 | 설명 | 사용 예시 |
|---|------|-----------|
| `_self` | 현재 프레임/윈도우 (기본값) | 같은 페이지에서 이동 |
| `_blank` | 새 윈도우/탭 | 새 창에서 열기 |
| `_parent` | 부모 프레임 | 프레임셋 환경에서 사용 |
| `_top` | 최상위 윈도우 | 전체 브라우저 윈도우 |
| `프레임명` | 지정된 프레임/윈도우 | 팝업창 이름 등 |

### Target 속성 동작 원리

```javascript
// 1. 기본 상태 (target 미설정 시 _self와 동일)
$("#searchForm").submit();  // 현재 페이지에서 이동

// 2. 팝업창을 target으로 설정
$("#searchForm").prop('target', 'popupWindow');
$("#searchForm").submit();  // popupWindow에서 결과 표시

// 3. target 복원
$("#searchForm").prop('target', '_self');
$("#searchForm").submit();  // 다시 현재 페이지에서 이동
```

## 해결책의 장점

### Before vs After 비교

| 구분 | Before (문제 상황) | After (해결 후) |
|------|-------------------|----------------|
| **팝업 후 조회** | 새 창에서 열림 | 현재 페이지에서 처리 |
| **코드 관리** | 모든 함수에서 target 설정 필요 | 팝업 함수에서만 관리 |
| **사용자 경험** | 예상치 못한 새 창 | 일관된 페이지 경험 |
| **개발 효율성** | 반복적인 코드 작성 | 한 곳에서 관리 |

### 최종 해결책의 특징

1. **중앙집중 관리**: 팝업 함수에서만 target 관리
2. **자동 복원**: submit 후 자동으로 _self로 복원
3. **코드 중복 제거**: 다른 함수에서 target 설정 불필요
4. **버그 방지**: target 설정 누락으로 인한 버그 방지

## 실무 적용 팁

### 1. 공통 팝업 함수 패턴

```javascript
function openPopup(popupName, url, options, formData) {
    // 팝업 창 열기
    window.open('', popupName, options);
    
    // 폼 설정
    $("#searchForm").prop('target', popupName);
    $("#searchForm").prop('action', url);
    
    // 추가 폼 데이터 설정
    if (formData) {
        Object.keys(formData).forEach(key => {
            $(`#${key}`).val(formData[key]);
        });
    }
    
    // 폼 제출 및 target 복원
    $("#searchForm").submit();
    $("#searchForm").prop('target', '_self');
}

// 사용 예시
function searchCustomer(customerGb) {
    openPopup(
        'findCompany',
        '/pm/prodPlan/popup/findCompany',
        'width=700, height=500, toolbar=no, menubar=no, scrollbars=no, resizable=no',
        { customerGb: customerGb }
    );
}
```

### 2. 에러 방지를 위한 안전장치

```javascript
function _customSearch(customerGb){
    try {
        $('#customerGb').val(customerGb);
        
        var popupOption = "width=700, height=500, toolbar=no, menubar=no, scrollbars=no, resizable=no";
        var popup = window.open('', 'findCompany', popupOption);
        
        // 팝업 블로커 체크
        if (!popup || popup.closed || typeof popup.closed == 'undefined') {
            alert('팝업이 차단되었습니다. 팝업 차단을 해제해주세요.');
            return;
        }
        
        $("#searchForm").prop('target', 'findCompany');
        $("#searchForm").prop('action', '/pm/prodPlan/popup/findCompany');
        $("#searchForm").submit();
    } catch (error) {
        console.error('팝업 열기 실패:', error);
    } finally {
        // 에러 발생 여부와 관계없이 target 복원
        $("#searchForm").prop('target', '_self');
    }
}
```

### 3. jQuery vs JavaScript 방식

```javascript
// jQuery 방식 (권장)
$("#searchForm").prop('target', '_self');

// JavaScript 방식
document.searchForm.target = '_self';
// 또는
document.getElementById('searchForm').target = '_self';
```

## 디버깅 팁

### Form Target 상태 확인

```javascript
// 현재 form의 target 확인
console.log('Current target:', $("#searchForm").prop('target'));

// 모든 form 요소의 target 확인
$('form').each(function(index) {
    console.log(`Form ${index} target:`, $(this).prop('target'));
});
```

### 브라우저 개발자 도구 활용

```javascript
// Console에서 실시간 target 변경 테스트
$("#searchForm").prop('target', 'testWindow');
console.log('Target changed to:', $("#searchForm").prop('target'));

$("#searchForm").prop('target', '_self');
console.log('Target reset to:', $("#searchForm").prop('target'));
```

## 개발 후기

이런 종류의 버그가 가장 찾기 어려운 것 같다. 기능 자체는 정상 동작하는데 가끔 예상과 다르게 동작하는 경우라서.

처음에는 원인을 몰라 당황했는데, 차근차근 추적해보니 target 속성 때문이었다.

해결책은 의외로 간단했다. 팝업 함수에서 submit 완료 후 target을 _self로 복원하는 것만으로 해결됐다.

이제 팝업 사용 시 target 속성 관리의 중요성을 깨달았다. 이런 경험들이 쌓여서 디버깅 스킬이 향상되는 것 같다.

앞으로 폼과 팝업을 함께 사용할 때는 target 관리를 더 주의깊게 해야겠다.