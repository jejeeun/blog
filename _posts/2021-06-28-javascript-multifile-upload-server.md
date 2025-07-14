---
title: 멀티파일 업로드 Spring Boot 서버 연동 구현
date: 2021-06-28 15:30:00 +0900
categories: [Frontend, Backend]
tags: [javascript, file-upload, spring-boot, ajax, formdata, multipartfile, server]
---

이전에 클라이언트 전용으로 구현했던 멀티파일 업로드에 Spring Boot 서버 연동을 추가했다. FormData와 Ajax를 활용해서 여러 파일을 한 번에 서버로 전송하고, 파일 용량 제한도 설정했다.

## 이전 포스팅 참고

지난 포스팅에서 멀티파일 업로드와 다운로드의 간단한 예를 보였습니다. 서버와의 통신 없이 JavaScript에서만 해결이 가능했습니다.

> **관련 포스팅:** [JavaScript MultiFile Upload, Download (서버X, 파일 개수 제한, 삭제)](/blog/posts/javascript-multifile-upload-download/)

이번에는 **두 가지 핵심 기능**을 구현합니다:
1. **파일 용량 제한** (Spring Boot Properties 설정)
2. **Ajax를 통한 멀티파일 서버 전송**

## 1. 파일 용량 제한 설정

### Spring Boot 2.x 버전 (Gradle 기준)

`application.properties` 파일에 다음 설정을 추가합니다:

```properties
spring.servlet.multipart.maxFileSize=5MB
spring.servlet.multipart.maxRequestSize=5MB
```

**설정 설명:**
- `maxFileSize`: 개별 파일 최대 크기
- `maxRequestSize`: 전체 요청 최대 크기 (여러 파일 합계)

5MB로 제한할 경우 위와 같이 설정하면 됩니다.

## 2. 멀티파일 서버 전송 구현

### 2.1 HTML 구조

```html
<!DOCTYPE html>
<html>
<body>
    <form>
        <input type="file" id="input_file" multiple>
        <button type="button" onclick="fileUpload()">저장</button>
    </form>
    <div id="fileList"></div>
    
    <script>
        var content_files = [];
        
        // 파일 업로드 함수
        function fileUpload() {
            var form = $("form")[0];        
            var formData = new FormData(form);
            
            // 삭제되지 않은 파일만 추가
            for (var x = 0; x < content_files.length; x++) {
                if(!content_files[x].is_delete){
                    formData.append("article_file", content_files[x]);
                }
            }
            
            // Ajax 파일 전송
            $.ajax({
                type: "POST",
                enctype: "multipart/form-data",
                url: "/common/fileUpload",
                data: formData,
                processData: false,
                contentType: false,
                success: function (data) {
                    if(data.msg == "success"){
                        alert("file upload successed.");
                    } else {
                        alert("file upload failed.");
                    }
                },
                error: function (xhr, status, error) {
                    alert("서버오류로 지연되고있습니다. 잠시 후 다시 시도해주시기 바랍니다.");
                    return false;
                }
            });
        }
    </script>
</body>
</html>
```

### 2.2 핵심 JavaScript 로직

#### FormData 객체 생성

```javascript
var form = $("form")[0];        
var formData = new FormData(form);
```

**설명:**
- `$("form")[0]`: 페이지의 첫 번째 form 태그 선택
- `FormData(form)`: form 요소를 기반으로 FormData 객체 생성
- FormData는 **key/value 쌍의 데이터를 JSON 형식으로 처리** 가능

> **참고:** [FormData - Web API | MDN](https://developer.mozilla.org/ko/docs/Web/API/FormData)

#### 파일 데이터 추가

```javascript
for (var x = 0; x < content_files.length; x++) {
    // 삭제 안한것만 담아 준다. 
    if(!content_files[x].is_delete){
        formData.append("article_file", content_files[x]);
    }
}
```

**핵심 로직:**
- `content_files`: 전역 변수로 선택된 파일들 저장
- `is_delete`: 파일 삭제 여부 플래그
- `formData.append()`: 키가 `article_file`인 쌍에 파일 객체 추가

#### Ajax 멀티파일 전송

```javascript
$.ajax({
    type: "POST",
    enctype: "multipart/form-data",
    url: "/common/fileUpload",
    data: formData,
    processData: false,    // 필수: jQuery가 데이터를 처리하지 않도록 설정
    contentType: false,    // 필수: Content-Type 헤더를 설정하지 않도록 설정
    success: function (data) {
        if(data.msg == "success"){
            alert("file upload successed.");
        } else {
            alert("file upload failed.");
        }
    },
    error: function (xhr, status, error) {
        alert("서버오류로 지연되고있습니다. 잠시 후 다시 시도해주시기 바랍니다.");
        return false;
    }
});
```

**중요한 설정:**
- `processData: false`: jQuery가 데이터를 자동 처리하지 않도록 설정
- `contentType: false`: Content-Type 헤더를 자동 설정하지 않도록 설정
- 이 두 설정은 **파일 업로드 시 필수**

## 3. Spring Boot 컨트롤러 구현

### 3.1 컨트롤러 코드

```java
@Controller
@RequestMapping("/common")
public class NoticeController {
    
    @Autowired
    private FileService fileService;
    
    @Autowired
    private CommonCodeService commonCodeService;
    
    @ResponseBody
    @RequestMapping("/fileUpload")
    public HashMap<String, Object> fileUpload(
            HttpSession session, 
            HttpServletRequest request, 
            @RequestParam("article_file") List<MultipartFile> multipartFile) {
        
        HashMap<String, Object> hmap = new HashMap<>();
        String msg = "fail";
        
        try {
            // 파일이 있을때
            if(multipartFile.size() > 0 && !multipartFile.get(0).getOriginalFilename().equals("")) {
                
                for(MultipartFile file : multipartFile) {
                    
                    String uploadPath = request.getSession().getServletContext().getRealPath("/files/pm");
                    String savedName = file.getOriginalFilename();

                    // 파일 업로드 후 파일 경로 리턴
                    String filePath = fileService.uploadFile(uploadPath, file); 
                    
                    FileInfo fileInfo = new FileInfo();
                    fileInfo.setFileId(String.format("%012d", fileService.selectMaxFileId() + 1));
                    fileInfo.setSeq(fileService.selectMaxSeq() + 1);
                    fileInfo.setFilePath(filePath);
                    fileInfo.setFileOriginName(savedName);
                    fileInfo.setRegistrationDate(new Date());
                    fileInfo.setRegistrationId(session.getAttribute("userNo").toString());
                    
                    fileService.insertFileInfo(fileInfo);
                    String fileId = fileService.selectFileIdMaxSeq();

                    hmap.put("fileId", fileId);
                    hmap.put("fileName", savedName);
                }
                msg = "success";
            }
            // 파일 업로드 없이 글을 등록하는 경우
            else {
                msg = "success";
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
        
        hmap.put("msg", msg);
        return hmap;
    }
}
```

### 3.2 컨트롤러 핵심 로직

#### 파라미터 받기

```java
@RequestParam("article_file") List<MultipartFile> multipartFile
```

- JavaScript에서 `formData.append("article_file", file)`로 전송한 파일들
- `List<MultipartFile>`로 여러 파일을 한번에 받음

#### 파일 처리 로직

```java
for(MultipartFile file : multipartFile) {
    String uploadPath = request.getSession().getServletContext().getRealPath("/files/pm");
    String savedName = file.getOriginalFilename();
    
    // 파일 업로드 후 파일 경로 리턴
    String filePath = fileService.uploadFile(uploadPath, file); 
    
    // 파일 정보 DB 저장
    FileInfo fileInfo = new FileInfo();
    // ... 파일 정보 설정
    fileService.insertFileInfo(fileInfo);
}
```

**처리 과정:**
1. 각 파일을 순차적으로 처리
2. 서버 경로에 실제 파일 저장
3. 파일 정보를 데이터베이스에 저장
4. 응답 데이터 구성

## 주요 개선사항

### 이전 버전 대비 차이점

| 구분 | 이전 버전 | 현재 버전 |
|------|-----------|-----------|
| **저장 위치** | 브라우저 메모리 | 서버 파일시스템 |
| **용량 제한** | 클라이언트 메모리 한계 | 서버 설정으로 제한 |
| **영속성** | 페이지 새로고침 시 소실 | 영구 저장 |
| **보안** | 클라이언트 의존적 | 서버 검증 가능 |

### 파일 업로드 플로우

```
1. 사용자 파일 선택
2. JavaScript로 FormData 생성
3. Ajax로 서버 전송
4. Spring Boot 컨트롤러 수신
5. 파일 시스템에 저장
6. DB에 메타데이터 저장
7. 클라이언트에 결과 응답
```

## 실무 활용 팁

### 1. 에러 처리 개선

```javascript
error: function (xhr, status, error) {
    if (xhr.status === 413) {
        alert("파일 크기가 너무 큽니다. 5MB 이하로 업로드해주세요.");
    } else {
        alert("서버오류로 지연되고있습니다. 잠시 후 다시 시도해주시기 바랍니다.");
    }
    return false;
}
```

### 2. 파일 확장자 제한

```java
// 컨트롤러에서 파일 확장자 검증
String fileName = file.getOriginalFilename();
String extension = fileName.substring(fileName.lastIndexOf("."));
if (!Arrays.asList(".jpg", ".png", ".pdf").contains(extension.toLowerCase())) {
    throw new IllegalArgumentException("허용되지 않는 파일 형식입니다.");
}
```

### 3. 업로드 진행률 표시

```javascript
$.ajax({
    // ... 기존 설정
    xhr: function() {
        var xhr = new window.XMLHttpRequest();
        xhr.upload.addEventListener("progress", function(evt) {
            if (evt.lengthComputable) {
                var percentComplete = evt.loaded / evt.total * 100;
                $('#progressBar').val(percentComplete);
            }
        }, false);
        return xhr;
    }
});
```

## 보안 고려사항

1. **파일 타입 검증**: 확장자뿐만 아니라 실제 파일 헤더 검증
2. **바이러스 스캔**: 대용량 파일 업로드 시 검증 필요
3. **경로 조작 방지**: 파일명 검증으로 디렉토리 트래버설 방지
4. **용량 제한**: 서버 리소스 보호를 위한 적절한 제한

## 개발 후기

FormData를 처음 사용해본 경험이었는데, 파일 업로드에 특화된 편리한 API라는 것을 알 수 있었다.

JavaScript 배열을 Java List로 받는 부분도 처음에는 헷갈렸지만, 이제는 자연스럽게 처리할 수 있다.

특히 `processData: false`와 `contentType: false` 설정을 빼먹어서 한참 디버깅했던 기억이 있다. 파일 업로드 시에는 반드시 필요한 설정이다.

서버에 파일 용량 제한을 설정하는 것도 중요하다는 것을 깨달았다. 추후 대용량 파일로 인한 서버 부하를 방지할 수 있다.

이번 구현을 통해 멀티파일 업로드 기능에 대한 전반적인 이해도가 높아졌다.