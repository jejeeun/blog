---
title: 동적 테이블 행 추가와 Ajax 일괄 전송 구현
date: 2021-06-18 18:39:00 +0900
categories: [Frontend, JavaScript]
tags: [javascript, ajax, jquery, table, controller, jsp]
---

사용자가 선택한 항목들을 화면에 동적으로 추가하고, 저장 버튼 클릭 시 모든 데이터를 한 번에 서버로 전송하는 기능을 구현했다.

## 구현 배경

처음에는 항목을 추가할 때마다 즉시 Ajax 요청을 보내려고 했다. 하지만 이 방식은 사용자가 5개를 추가하면 서버로 5번 요청을 보내야 하고, 추가할 때마다 페이지가 새로고침되는 문제가 있었다.

이런 비효율성을 해결하기 위해 전역변수를 활용해서 프론트엔드에서 데이터를 임시 저장하고, 저장 버튼을 누를 때 한 번에 처리하는 방식으로 변경했다.

## 구현 과정

### 1. 기본 테이블 만들기

```html
<table id="authorityTbl">
    <colgroup>
        <col style="width:8.4%">    
        <col style="width:87.6%">        
        <col style="width:4%">        
    </colgroup>
    <tbody>
    </tbody>
</table>

<div>
    <a href="" onclick="return false;">
        <button id="registerBtn">저장</button>
    </a>
</div>
```

tbody는 비워두고 JavaScript로 내용을 채워넣는다.

### 2. 첫 번째 행 만들기

```javascript
var param = "$(this).closest('tr')";
var insertAuthorityHtml =        
    '<tr id="newFrm">'
    +        '<td id="newUserNoTd">'
    +            '<input type="hidden" id="new_userNo" name="new_userNo" value="new"/>'
    +        '</td>'
    +        '<td id="new_userName">'
    +        '</td>'
    +        '<td onClick="javascript:userPopup('+param+');">'
    +            '<a href="" onclick="return false;">'
    +                '<i class="fas fa-user-plus" style="font-size:20px;"></i>'
    +            '</a>'
    +        '</td>'
    +    '</tr>';
        
$("#authorityTbl > tbody").empty();
$("#authorityTbl > tbody:last").append(insertAuthorityHtml);
```

첫 번째 행은 사용자 추가 버튼(+)이 있는 행이다. 이 버튼을 누르면 팝업이 뜬다.

### 3. 팝업에서 사용자 선택

노란색 + 버튼을 누르면 `userPopup()` 실행:
- 사용자 목록을 보여주는 팝업
- 원하는 사용자를 선택하면 원래 페이지로 돌아옴

### 4. 선택한 사용자를 테이블에 추가하기

```javascript
var changeCnt = 0;
var changeArr = new Array();

function addRow(userNo, userName) {
    var insertTr = $("#newFrm");
        
    var len = $("#authorityTbl tr").length;
        
    var htmlStr =    
        '<tr>'
    +    '<td>'
    +        len
    +        '<input type="hidden" id="' + userNo + '_userNo" name="' + userNo + '_userNo" value="' + userNo + '"/>'
    +    '</td>'
    +    '<td>'
    +        userName
    +    '</td>'
    +    '<td onClick="javascript:deleteRow(this);">'
    +            '<i class="fas fa-trash-alt" style="font-size:20px;"></i>'
    +        '</a>'
    +    '</td>'
    +    '</tr>';

    insertTr.before(htmlStr);
        
    changeCnt++;
    changeArr.push(userNo);
}
```

팝업에서 '바나나'를 선택했다면 `addRow(바나나의ID, '바나나')`가 실행된다.

중요한 건 `changeCnt`랑 `changeArr`이다. 이 변수들이 전역변수라서 페이지가 새로고침되지 않는 한 계속 유지된다. 
- `changeCnt`: 몇 개를 추가했는지 카운트
- `changeArr`: 추가된 사용자들의 ID를 저장

### 5. 저장 버튼 눌러서 서버로 전송

```javascript
/* 저장 버튼 */
$("#registerBtn").click(function() {
    if(changeCnt == 0) {
        alert('변경된 부분이 없습니다.');
        return false;
    }
    
    var data = "&changeArr=" + changeArr;

    $.ajax({ 
        type : 'post',
        dataType : 'json',
        url  : '/pm/systemManag/updateList',
        data : data, 
        success : function(result){
            changeCnt = 0;
            changeArr.length = 0;
        },
        error : function(request,status,error){
            return;
        }
    });
});
```

저장 버튼을 누르면:
1. 변경된 게 없으면 alert 띄우고 끝
2. `changeArr`을 서버로 전송
3. 성공하면 `changeCnt`와 `changeArr` 초기화

### 6. 서버에서 데이터 받기

```java
@ResponseBody
@RequestMapping("/updateList")
public HashMap<String, Object> updateList(Model model, HttpServletRequest request, HttpSession session
    ,@RequestParam(value = "changeArr") List<String> userNoArr) throws Exception {
    
    HashMap<String, Object> hmap = new HashMap<>();
    ElementInfo elementInfo = new ElementInfo();
    
    for(int i=0; i<userNoArr.size(); i++) {
        String userNo = userNoArr.get(i);
        elementInfo.setUserNo(userNo);
        
        elementService.insertList(elementInfo);
    }
    return hmap;
}
```

여기서 중요한 건 JavaScript에서는 배열로 보내지만, Java에서는 `List<String>`로 받는다는 것이다. 
받은 데이터를 for문 돌려서 하나씩 DB에 저장하면 끝이다.

## 개발 후기

처음에는 "추가할 때마다 서버로 보내는 게 맞지 않나?"라고 생각했는데, 실제로 구현해보니 너무 비효율적이었다.

사용자가 10개를 추가하면 서버로 10번 통신해야 하고, 그때마다 페이지가 새로고침되어 사용자 경험이 좋지 않았다.

전역변수를 활용해서 프론트엔드에서 상태를 관리하는 방식으로 변경하니 훨씬 부드럽게 동작했다. 저장 버튼 클릭 시 일괄 처리하는 것이 올바른 접근 방식인 것 같다.

JavaScript 배열을 Java List로 받는 부분도 처음에는 헷갈렸는데, 이제는 이런 데이터 변환이 자연스럽다.

이런 작은 최적화들이 모여서 전체 시스템의 성능을 좌우한다는 것을 깨달았다.