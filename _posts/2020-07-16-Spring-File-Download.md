---
layout: post
title: Spring File Download 정리
excerpt: "Spring MVC 프레임워크에서 파일 다운로드를 구현하는 방법에 대해 정리해보았다."
categories: [TIL - Spring]
comments: true
---

Spring File 다운로드
=====

DB에 있는 자료를 (굳이) 엑셀로 다운받는 기능을 추가해달라고 하셔서, 기존 시스템에 추가 기능을 개발하게 되었다. 

엑셀로 변환하는 과정은 (Apache POI 사용) 어렵지 않았다. 그러나 Spring 서버에서 파일을 사용자에게 보내줄 때 어떤 방법으로 보내는 것이 좋은지에 대해 고민했던 과정을 적어보았다.

## 웹 서버에서 파일을 전송하는 방법
사실 브라우저를 통해 서버에 요청을 보내는 것 자체가 이미 파일을 다운로드 하는 행위이다.
브라우저는 웹 서버에서 제공하는 정적 파일을 가져와 렌더링하기 때문이다. 그러나, 브라우저에서 렌더링을 지원하지 않는 형식의 파일을 사용자의 요청에 따라 내용을 다르게 만들어서 내보내기 때문에 단순한 방법으로 해결할 수 없었다.

## request, response
내보내려는 파일은 xlsx형식이며 요청에 따라 파일의 크기가 커질 수 있기 때문에 request와 response의 헤더를 조금 만져줄 필요가 있다.

먼저, request에는 사용자의 요청이 담긴 form을 전송하니 content-type을 mutlipart/form-data로 설정하였다. 또 헤더에 있는 파라미터는 아니지만, js단에서 `XMLHttpRequest` 객체(이하, xhr 객체)를 사용하여 요청을 보냈기 때문에 응답이 어떤 종류의 데이터를 포함하는지 명시하기 위해 객체의 `responseType`을 blob으로 고정하였다.
(js Blob 객체에 대한 설명은 아래 단락)

response에는 `contentType`을 `application/octet-stream`으로 두어, binary 형태로 데이터를 전송하겠다고 명시했다. (Excel 파일의 표준 MIME type은 `application/vnd.ms-excel`이지만, 확장자를 '.xlsx'로 해달라고 요청하셔서..)
`contentLength`를 파일의 크기만큼 지정하고, `Content-Disposition`를 attachment로 명시하여 다운로드 되어야하는 파일이라는 것을 알려준다.(추가적으로, filename도 지정함)

## Front(javascript)
파일을 다운로드 받을때 페이지를 다시 로드하거나 새로 창을 열게하지 않기 위하여  `XMLHttpRequest` 객체를 이용하여 비동기 요청을 보냈다. `XMLHttpRequest` 객체의 responseType을 'blob'으로 세팅했기 때문에 서버의 응답은 아래와 같이 Blob객체로 리턴된다. 
![](/img/blob-capture.png)

js에서 Blob객체는 큰 사이즈의 binary를 다루기 위한 객체이며 해당 객체를 DOM에서 참조할 수 있도록 하는 메소드를 지원한다.

아래는 이번에 프론트단에서 실제로 사용한 코드를 정리한 것이다.
```javascript
function saveOrOpenBlob(blob, fileName, blobUrl) {
    let a = document.createElement('a');
    blobUrl = window.URL.createObjectURL(blob);
    a.href = blobUrl;
    a.download = fileName;
    a.dispatchEvent(new MouseEvent('click'));
}

excelBtn.addEventListener("click", function(){
    let searchForm = document.getElementById("listForm");
    let data = new FormData(searchForm);
    let blobUrl = "";

    let xhr = new XMLHttpRequest();
    xhr.open('POST', "file-download-url", true);
    xhr.responseType = 'blob';
    xhr.onloadstart = function(e){
        // 파일 생성 대기 메세지
    };
    xhr.onload = function(e) {
        let resp = xhr.response;
        if(resp.type !== "application/octet-stream" || resp.size === 0){
            // 에러 메세지
        } else{
            let contentDispo = xhr.getResponseHeader('Content-Disposition');
            let fileName = contentDispo.match(/filename[^;=\n]*=((['"]).*?\2|[^;\n]*)/)[1];
            saveOrOpenBlob(resp, fileName, blobUrl);
        }
    };
    xhr.onloadend = function(e){
        // 파일 생성 대기 해제
        // 파일 다운로드 종료 후 Blob url 해제
        window.URL.revokeObjectURL(blobUrl);
    };
    xhr.send(data);
});
```

`URL.createObjectURL()`은 Blob 객체를 나타내는 URL을 포함한 DOMString을 생성한다. 이 url은 document에서만 유효하기 때문에 file:URL과 다르게 보안 이슈에서 벗어날 수 있다. 그러나, 여러 번 요청 시 기존 URL을 유효하다고 판단하여 GC가 일어나지 않아 메모리 누수가 일어날 수 있다. 때문에 `URL:revokeObjectURL()` 메소드를 통해 명시적으로 기존 url을 폐기한다.

## BackEnd(Spring)
다음은 스프링 기반 백엔드에서 파일을 클라이언트에게 보내는 방법이다.
다양한 방법이 있지만 실패한 방법을 포함해 시도해본 방법 몇 가지를 소개하겠다.

### Stream1
가장 첫 번째로 생각나는 방법은 파일을 Stream으로 파일을 읽고, response에 직접 파일을 쓰는 것이었다.

```java
@RequestMapping(value = "/file/download")
public void fileDownload(HttpServletResponse response) throws IOException {
    // Response Header 설정
    // 파일 가져오기 
    ......

    InputStream inputStream = new FileInputStream(file);
    assert inputStream != null;
    byte[] byteArr = new byte[inputStream.available()];
    inputStream.read(byteArr);

    response.getOutputStream().write(byteArr);
    response.flushBuffer();
    inputStream.close();
}
```

그런데, 이렇게 짜고나니까 두 가지 문제점을 발견했다.

첫 번째 문제점은 파일을 메모리에 온전히 올려서 다시 보낸다는 것이다. 데이터가 많아진다면 생성되는 엑셀 파일의 크기도 커질 것이고, 동시에 요청이 들어온다면 `파일의 크기 * 요청의 수` 만큼의 추가적인 메모리(heap space)가 필요해질 것이다.

두 번째 문제점은 `inputStream.available()`에 있었는데 Java API Doc을 읽어보면 그 문제를 알 수 있다.
```
Returns an estimate of the number of bytes that can be read (or skipped over) from this input stream without blocking by the next invocation of a method for this input stream. The next invocation might be the same thread or another thread. A single read or skip of this many bytes will not block, but may read or skip fewer bytes.
Note that while some implementations of InputStream will return the total number of bytes in the stream, many will not. It is never correct to use the return value of this method to allocate a buffer intended to hold all data in this stream.
```
즉, 나는 `available()` 메소드를 파일의 사이즈만큼 가져와 한 번에 읽기 위해 사용했지만 다른 스레드에서 해당 파일을 사용하기 위해 blocking을 한다면 파일의 사이즈만큼 byte를 리턴하지 않을 수 있다는 뜻.

첫 번째, 두 번째 모두 내버려둘 수 없는 문제라 다음 해결방법을 찾아보았다.

### Spring Resource 활용하기
Stackoverflow에서 가장 많이 나왔던 답변은 바로 ResponseEntity를 활용하는 것이었다. ResponseEntity 클래스는 HttpEntity를 상속받은 클래스로, RESTful 한 응답을 만들때 자주 사용된다. @ResponseBody는 메소드의 리턴만 응답하는데에 반해, ResponseEntity를 리턴 타입으로 가지면 헤더, 바디, 상태코드를 포함하고 있어 조금 더 견고하고 안정적인 API를 짤 수 있다.

여기서는, InputStreamResource를 ResponseEntity의 제네릭 활용을 통해 서버 메모리에 전부 올리지 않고 사용자에게 보내기 위한 방법으로 사용하였다.

```java
@RequestMapping(value = "/file/download")
public ResponseEntity<InputStreamResource> fileDownload() throws IOException {
    // 파일 가져오기 
    ......

    InputStreamResource resource = new InputStreamResource(new FileInputStream(file));
    return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"filename.xlsx\"")
            .contentLength(resource.contentLength())
            .contentType(MediaType.parseMediaType("application/octet-stream"))
            .body(resource);
}
```

잘 될 것이라 기대하고 테스트를 했는데 파일이 다운로드되지 않았다. 디버깅하면서 분명 resource를 잘 읽어온걸 확인했는데 왜 다운로드가 안되는지 한참을 헤메다가 결국 답을 찾아냈다. 
ResponseEntity 혹은 @ResponseBody등을 사용하면 리턴타입이 ViewResolver를 거치지 않고 HttpMessageConverter를 거쳐 알맞게 변환되어 응답으로 나가게 되는데, 프로젝트에 등록된 HttpMessageConverter 중 InputStreamResource 클래스를 알맞은 응답형태로 변환하는 Converter가 없었기 때문이다.

이 것을 해결하려면 스프링 설정(servletContext)에서 아래와 같이 ResourceHttpMessageConverter를 제대로 등록해주어야 한다.

```xml
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
    <property name="webBindingInitializer">
        <bean class="....MyBindingInitializer"/>
    </property>
    <property name="messageConverters">
        <list>
            ......
            <!-- Resource converter -->
            <bean class="org.springframework.http.converter.ResourceHttpMessageConverter" />
            ......
        </list>
    </property>
</bean>
```

Converter를 등록하자 제대로 작동되었다. 그러나, 다른 messageConverter에 어떤 영향을 줄 지 모르기 때문에 나만 개발하는 것이 아닌 프로젝트의 설정을 건드는 것은 단독으로 결정할 수 없었다. (그리고, 테스트는 해보지 않았지만 InputStreamResource 객체에서 getInputStream 메소드가 호출될 때 "socket closed" exception이 뜰 수도 있다고 한다.)

### Stream2
다시 돌아와서 Stream을 이용한 방법을 고민하기 시작했다.
첫 시도에서 문제가 되었던 부분은 파일을 모든 크기만큼 읽어오는 것과 `available()` 메소드의 불안정성이었다.
이러한 부분을 해소하기 위해 buffer를 이용해 정해진 크기만큼 읽고 쓰는 방법을 채택했다.

```java
@RequestMapping(value = "/file/download")
public void fileDownload(HttpServletResponse response) throws IOException {
    // Response Header 설정
    // 파일 가져오기 
    ......

    InputStream inputStream = new FileInputStream(file);
    assert inputStream != null;

    int n;
    byte[] data = new byte[2048];
    
    while((n = inputStream.read(data, 0, data.length)))
        response.getOutputStream().write(data, 0, n);

    response.flushBuffer();
    inputStream.close();
}
```

그러나, spring이나 다른 라이브러리에서 제공하는 copy 유틸들이 많이 있다는 것을 알고 여러가지 중 StreamUtil을 골라 아래와 같이 수정하였다.

```java
@RequestMapping(value = "/file/download")
public void fileDownload(HttpServletResponse response) throws IOException {
    // Response Header 설정
    // 파일 가져오기 
    ......

    InputStream inputStream = new FileInputStream(file);
    assert inputStream != null;

    StreamUtils.copy(inputStream, response.getOutputStream())

    response.flushBuffer();
    inputStream.close();
}
```

시도해서 성공한 비슷한 copy util들

- `java.nio.file.Files.copy()`
- `org.apache.commons.io.IOUtils.copy()`
- `org.springframework.util.FileCopyUtils` 클래스 활용

StreamUtils 를 포함해서 copy 메소드 내부에서 어떻게 돌아가는지는 아직 뜯어보지 않았지만, 2048바이트를 고정으로 내보내는 내 코드보다는 훨씬 유연할 것이라 생각되므로 최종 반영은 위의 코드로 결정하였다.

## 결론
결국 돌아서 첫 번째 방법을 단순화시킨 것을 활용하게 되었다. 찾다보니 xlsx를 다루기 위한 mimetype도 있다는 것을 알게되었다.(application/vnd.openxmlformats-officedocuments.spreadsheetml.sheel)
돌아가서 나중에 바꾸기로 하고.. 지난 번 Java IO를 정리했음에도 불구하고 아직 IO에 대해 이해를 완벽히 하고있지 않다는 것을 느꼈고, NIO도 한 번은 정리하면서 Non-blocking과 Blocking의 차이를 알아봐야겠다는 생각을 했다. 정답은 없겠지만 항상 어떤 상황에서 무엇이 최선일지 고민하는 자세를 잃지 않아야 할 듯.
