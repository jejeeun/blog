---
title: 클라이언트 전용 멀티파일 업로드/다운로드 구현
date: 2021-06-24 17:18:00 +0900
categories: [Frontend, JavaScript]
tags: [javascript, file-upload, file-download, filereader, multifile, frontend]
---

서버 연동 없이 JavaScript만으로 멀티파일 업로드, 다운로드, 삭제 기능을 구현했다. FileReader API를 활용해서 파일을 읽고 Data URL로 변환하여 브라우저에서 직접 다운로드할 수 있게 했다.

## 전체 코드 구조

```html
<!DOCTYPE html>
<html>
<head>
    <title>멀티파일 업로드/다운로드</title>
</head>
<body>
    <button id="uploadButton">파일추가</button>
    <input type="file" id="input_file" multiple style="display:none;">
    <div id="fileChange"></div>
    
    <script>
        var filesArr = [];
        var fileNum = 0;
        var maxFileNum = 5; // 최대 파일 개수
        
        // 파일 추가 버튼 클릭
        $('#uploadButton').click(function (e) {
            e.preventDefault();
            $('#input_file').click();
        });
        
        // 파일 선택 시
        $("#input_file").change(function(e){            
            fileCheck(e);
        });
        
        function fileCheck(e) {
            var files = e.target.files;
            
            // 파일 개수 제한 체크
            if (files.length + filesArr.length > maxFileNum) {
                alert('최대 ' + maxFileNum + '개까지 첨부 가능합니다.');
                return;
            }
            
            // 파일 배열에 추가
            for (var i = 0; i < files.length; i++) {
                filesArr.push(files[i]);
            }
            
            // 배열 처리
            filesArr.forEach(function (f) {
                if(f.name == ""){
                    return;
                }
                addFile(f);
            });
        }
        
        function addFile(f){
            var reader = new FileReader();
            
            reader.readAsDataURL(f);

            // 읽기 동작이 성공적으로 완료되었을 때 발생
            reader.onload = function (e) {
                $('#fileChange').append(
                    '<div id="file' + fileNum + '">'
                    + '<font>' + f.name + '</font>' 
                    + '    <a href="#" id="download' + fileNum + '" download="' + f.name + '" target="_blank">'
                    + '        <button type="button" onclick="fileDownload(\'' + fileNum + '\',\'' + reader.result + '\',\'' + f.name + '\')">다운로드</button>'
                    + '    </a>'
                    + '    <a href="#"><button type="button" onclick="fileDelete(\'file' + fileNum + '\')">파일삭제</button></a>'
                    + '</div>'
                );
                fileNum++;
            };
        }
        
        // 파일 다운로드
        function fileDownload(fileNum, src, fileName){
            var fileId = 'download' + fileNum;
            var element = document.getElementById(fileId);
            
            element.setAttribute('href', src);
            element.setAttribute('download', fileName);
        }
        
        // 파일 삭제
        function fileDelete(fileId) {
            $('#' + fileId).remove();
            
            // 배열에서도 제거
            var index = parseInt(fileId.replace('file', ''));
            filesArr.splice(index, 1);
        }
    </script>
</body>
</html>
```

## 핵심 구현 로직

### 1. 파일 선택 트리거

```javascript
// [파일추가] 버튼 클릭 시 
$('#uploadButton').click(function (e) {
    e.preventDefault();
    $('#input_file').click();
});
```

**핵심 포인트:**
- `[파일추가]` 버튼 클릭 시 숨겨진 `file` 타입 input 태그 실행
- `multiple` 속성으로 멀티업로드 가능
- `preventDefault()`로 기본 동작 방지

### 2. 파일 선택 처리

```javascript
// 파일 선택 시 
$("#input_file").change(function(e){            
    fileCheck(e);
});
```

파일 첨부 창에서 파일을 선택하면 `fileCheck()` 함수가 실행됩니다.

### 3. 파일 검증 및 배열 처리

```javascript
function fileCheck(e) {
    var files = e.target.files;
    
    // 파일 개수 제한 체크
    if (files.length + filesArr.length > maxFileNum) {
        alert('최대 ' + maxFileNum + '개까지 첨부 가능합니다.');
        return;
    }
    
    // 파일 배열에 추가
    for (var i = 0; i < files.length; i++) {
        filesArr.push(files[i]);
    }
    
    // 배열 처리
    filesArr.forEach(function (f) {
        if(f.name == ""){
            return;
        }
        addFile(f);
    });
}
```

**주요 기능:**
- 파일 개수 제한 검증
- 선택된 파일들을 `filesArr` 배열에 저장
- 각 파일을 `addFile()` 함수로 개별 처리

### 4. 파일 UI 생성

```javascript
function addFile(f){
    var reader = new FileReader();
    
    reader.readAsDataURL(f);

    // 읽기 동작이 성공적으로 완료되었을 때 발생
    reader.onload = function (e) {
        $('#fileChange').append(
            '<div id="file' + fileNum + '">'
            + '<font>' + f.name + '</font>' 
            + '    <a href="#" id="download' + fileNum + '" download="' + f.name + '" target="_blank">'
            + '        <button type="button" onclick="fileDownload(\'' + fileNum + '\',\'' + reader.result + '\',\'' + f.name + '\')">다운로드</button>'
            + '    </a>'
            + '    <a href="#"><button type="button" onclick="fileDelete(\'file' + fileNum + '\')">파일삭제</button></a>'
            + '</div>'
        );
        fileNum++;
    };
}
```

**FileReader API 활용:**
- `FileReader` 객체로 파일 읽기
- `readAsDataURL(f)`: 파일을 Data URL로 변환
- `reader.result`: 변환된 Data URL 경로 반환
- `onload` 이벤트: 읽기 성공 시 UI 생성

### 5. 파일 다운로드 구현

```javascript
function fileDownload(fileNum, src, fileName){
    var fileId = 'download' + fileNum;
    var element = document.getElementById(fileId);
    
    element.setAttribute('href', src);
    element.setAttribute('download', fileName);
}
```

**다운로드 메커니즘:**
- `reader.result`를 `src`로 전달받음
- `a` 태그의 `href` 속성에 Data URL 설정
- `download` 속성으로 파일명 지정
- 브라우저에서 직접 다운로드 처리

### 6. 파일 삭제 기능

```javascript
function fileDelete(fileId) {
    $('#' + fileId).remove();
    
    // 배열에서도 제거
    var index = parseInt(fileId.replace('file', ''));
    filesArr.splice(index, 1);
}
```

**삭제 로직:**
- DOM에서 해당 파일 요소 제거
- `filesArr` 배열에서도 해당 파일 제거
- 인덱스 기반으로 정확한 파일 식별

## 주요 특징

### 1. **서버 독립적 처리**
- 서버 없이 클라이언트에서만 동작
- FileReader API로 파일 읽기
- Data URL을 통한 직접 다운로드

### 2. **파일 개수 제한**
- `maxFileNum` 변수로 최대 개수 설정
- 실시간 개수 체크 및 알림

### 3. **개별 파일 관리**
- `fileNum`으로 각 파일 고유 식별
- 선택적 삭제 가능
- 파일별 다운로드 지원

### 4. **사용자 친화적 UI**
- 파일명 표시
- 다운로드/삭제 버튼 제공
- 직관적인 인터페이스

## 실무 활용 팁

1. **파일 타입 검증**: 확장자 필터링 추가 고려
2. **파일 크기 제한**: 대용량 파일 처리 시 성능 이슈 주의
3. **에러 처리**: FileReader 실패 시 예외 처리 필요
4. **메모리 관리**: 대량 파일 처리 시 메모리 사용량 모니터링

## 브라우저 호환성

- **FileReader API**: IE10+ 지원
- **Data URL**: 모든 모던 브라우저 지원
- **download 속성**: HTML5 지원 브라우저 필요

## 개발 후기

처음에 FileReader API를 접했을 때 readAsDataURL 메서드가 낯설었다. 하지만 이해하고 나니 파일을 Data URL로 변환하면 바로 다운로드 링크로 활용할 수 있다는 점이 흥미로웠다.

서버 없이도 이런 파일 처리가 가능하다는 것이 인상적이었고, 파일 개수 제한이나 삭제 기능을 추가하면서 실제 사용 가능한 수준으로 만들 수 있었다.

파일 업로드 기능은 대부분의 프로젝트에서 필요한 기능이라 이 코드를 재사용 가능하도록 잘 정리해두었다.