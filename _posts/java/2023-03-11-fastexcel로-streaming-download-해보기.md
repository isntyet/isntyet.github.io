---
title: "fastexcel로 streaming download 해보기"
date: 2023-03-11T15:30:30-16:00
categories:
  - java
tags:
  - java
  - excel
  - fastexcel
  - download
  - spring boot
comments: true
---

개발하고있는 서비스에 대량의 데이터를 엑셀로 다운로드해야하는 API가 필요하여 알아보던 중 아래와 같은 목표를 가지고 방법을 찾게되었다.

- S3와 같은 별도의 저장소를 사용하지 않기
- 다운로드시 임시 파일을 생성하지 않기
- 엑셀 파일을 생성할 때 전체 데이터를 통째로 메모리에 올려서 엑셀 파일을 생성하지 않기
- 다운로드 클릭시 즉시 다운로드가 시작되게 하기 (파일 다 만들고 다운X)

기존에 사용하던 다운로드 방식은 비동기 다운로드 방식으로  
사용자가 다운로드 버튼을 누르면 s3에 파일을 비동기로 저장하게 되고  
다운로드 요청 목록을 확인 할 수 있는 화면으로 이동하여 s3 다운로드 링크를 이용하여 다운로드 하는 방식인데  
사용성이 매우 떨어졌다

위와 같은 이유 등등 때문에 방법을  알아보던중  
**[[SpringBoot + Fastexcel] 대용량 엑셀 생성 및 다운로드](https://jaimemin.tistory.com/2191?category=1084044)**   
블로그에서 아주 잘 설명이 되어있고 내 목적에 딱 맞아서 따라해보기로 했다

---

### 구현

- Sample Code ([github](https://github.com/isntyet/java-practice/commit/8d5b845eccc279c622da8c6d5fe0064a9fb19826))
  - dependency
      ```groovy
      implementation 'org.dhatim:fastexcel:0.14.18'
      ```

  - Controller
      ```java
      // get 버전 (간단 버전)
      @GetMapping("/download")
      public void downloadGet(HttpServletResponse response) throws IOException {
          response.setContentType("application/vnd.ms-excel");
          response.setCharacterEncoding("utf-8");
          String fileNameUtf8 = URLEncoder.encode("FAST_EXCEL", StandardCharsets.UTF_8);
          response.setHeader("Content-Disposition", "attachment; filename=" + fileNameUtf8 + ".xlsx");
      
          try (OutputStream os = response.getOutputStream()) {
              Workbook wb = new Workbook(os, "PracticeApplication", "1.0");
      
              Worksheet ws = wb.newWorksheet("home");
              for (int i = 0; i < 500000; i++) {
                  ws.value(i, 0, "No.");
                  ws.value(i, 1, "첫번 째 칼럼 " + i);
                  ws.value(i, 2, "두번 째 칼럼");
                  ws.value(i, 3, i + "세번 째 칼럼");
      
                  if (i % 100 == 0) {
                      System.out.println("flush 중 " + i);
                      ws.flush();
                  }
              }
      
              ws.flush();
              ws.finish();
      
              wb.finish();
          } catch (IOException e) {
              log.error("[fastexcel] ERROR {}", e.getMessage());
          }
      }
      
      // post 버전
      @PostMapping(value = "/download")
      public void downloadPost(
              HttpServletResponse response,
              HomeFilter filter
      ) throws IOException {
          response.setContentType("application/vnd.ms-excel");
          response.setCharacterEncoding("utf-8");
          String fileNameUtf8 = URLEncoder.encode("FAST_EXCEL", StandardCharsets.UTF_8);
          response.setHeader("Content-Disposition", "attachment; filename=" + fileNameUtf8 + ".xlsx");
      
          try (OutputStream os = response.getOutputStream()) {
              this.homeService.excelDownload(os, filter);
          } catch (IOException e) {
              log.error("error {}", e.getMessage());
          }
      }
      ```

  - Service
      ```java
      @Transactional(readOnly = true)
      public void excelDownload(OutputStream os, HomeFilter filter) throws IOException {
          final int FLUSH_SIZE = 100;
          Workbook wb = new Workbook(os, "PracticeApplication", "1.0");
      
          Worksheet ws = wb.newWorksheet("home");
          ws.value(0, 0, "No.");
          ws.value(0, 1, "이름");
          ws.value(0, 2, "주소");
          ws.value(0, 3, "가격");
      
          int row = 1;
          int page = 0;
      
          while (true) {
              var homes = homeRepository.findAllHomesByFilterWithPage(
                      filter,
                      PageRequest.of(page, 10000)
              );
      
              if (homes.getContent().size() == 0) {
                  break;
              }
      
              for (Home home : homes) {
                  ws.value(row, 0, row);
                  ws.value(row, 1, home.getName());
                  ws.value(row, 2, home.getAddress());
                  ws.value(row, 3, home.getPrice());
      
                  if (++row % FLUSH_SIZE == 0) {
                      ws.flush();
                  }
              }
      
              ws.flush();
              ws.finish();
      
              page++;
          }
      
          wb.finish();
      }
      ```

  - Test HTML
      ```html
      <!DOCTYPE html>
      <html lang="en">
      <head>
          <meta charset="UTF-8">
          <title>EXCEL DOWNLOAD</title>
      </head>
      <body>
      
      <div class="contents_wrap">
      
          <div>
              <input type="button" value="fastexcel get windowopen" onclick="fastexcelWindowOpen()">
          </div>
          <br/>
      
          <form action="http://localhost:8080/home/download" method="post">
              <input type="text" name="name" value="name">
              <input type="submit" value="fastexcel post form with filter">
          </form>
          <br/>
      </div>
      
      <script
              src="https://code.jquery.com/jquery-3.6.1.js"
              integrity="sha256-3zlB5s2uwoUzrXK3BT7AX3FyvojsraNFxCc2vC/7pNI="
              crossorigin="anonymous"></script>
      <script type="text/javascript">
      
          function fastexcelWindowOpen() {
              var excelDownloadUrl = "http://localhost:8080/home/download";
              window.open(excelDownloadUrl, "_self", 'width=200, height=200, left=2000, top=2000');
          }
      
      </script>
      </body>
      </html>
      ```


다른 자세한 코드는 [여기](https://github.com/isntyet/java-practice/commit/8d5b845eccc279c622da8c6d5fe0064a9fb19826)를 확인 하면 된다

![fastexcel로 streaming download 해보기/0.png](/assets/images/fastexcel로 streaming download 해보기/0.png)

---

## 다운로드 해보기

이제 다운로드 버튼을 눌러 실행해보면 실행과 동시에 다운로드가 즉시 진행되며,  
쿼리도 내가 정한 사이즈만큼 실행되어 즉시 cell을 만들고 지정한 flush 사이즈 만큼씩 내보내는것을 확인 할 수 있다

- 다운로드 버튼을 누르는 즉시 다운로드 진행하는 모습
  ![fastexcel로 streaming download 해보기/1.png](/assets/images/fastexcel로 streaming download 해보기/1.png)


- 다운로드 중간에 paging 만큼 계속 쿼리가 실행되는 모습
  ![fastexcel로 streaming download 해보기/2.png](/assets/images/fastexcel로 streaming download 해보기/2.png)


---

## 추가 문제

해당 코드를 실제 운영에서 반영하는데 다운로드가 즉시 진행되지 않고,  
파일이 다 만들어지고 나서 다운로드가 진행되는 현상이 나타났다

- 즉시 다운로드되는 버전의 Response Header
  ![fastexcel로 streaming download 해보기/3.png](/assets/images/fastexcel로 streaming download 해보기/3.png)


- 즉시 다운로드 되지 않는 버전의 Response Header
  ![fastexcel로 streaming download 해보기/4.png](/assets/images/fastexcel로 streaming download 해보기/4.png)


즉시 다운로드 되는 버전의 Response Header를 보면 Transfer-Encoding가 chunked고  
안되는 버전은 Transfer-Encoding이 없는 대신에 Content-Length가 있다

안되는 버전에서 코드를 소거해보면서 확인한 결과 안되는 버전에서만 있던 아래 코드가 문제였다

- CacheRequestFilter
    ```java
    @Component
    public class CacheRequestFilter extends OncePerRequestFilter {
        @Override
        protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
            ContentCachingRequestWrapper wrappingRequest = new ContentCachingRequestWrapper(request);
            ContentCachingResponseWrapper wrappingResponse = new ContentCachingResponseWrapper(response);
    
            filterChain.doFilter(wrappingRequest, wrappingResponse);
    
            wrappingResponse.copyBodyToResponse();
        }
    }
    ```


Request와 Response 로깅을 위해 캐싱하려고 만든 RequestFilter가 문제였던 것이다

`ContentCachingResponseWrapper` 에서 response를 wrapping하는데 이때 OutPutStream을 `ResponseServletOutputStream` 을 사용한다

- ResponseServletOutputStream
    ```java
    private class ResponseServletOutputStream extends ServletOutputStream {
    
    		private final ServletOutputStream os;
    
    		public ResponseServletOutputStream(ServletOutputStream os) {
    			this.os = os;
    		}
    
    		@Override
    		public void write(int b) throws IOException {
    			content.write(b);
    		}
    
    		@Override
    		public void write(byte[] b, int off, int len) throws IOException {
    			content.write(b, off, len);
    		}
    		 
    		...
    	}
    ```


뭔가 content에 write하고 있는데 ContentCachingResponseWrapper 자체가 content 캐싱을 위한거니 content가 완성 되어야 캐싱이 가능하기 때문에 모아서 content가 완성된 후에 내보내기 때문에 즉시 streming이 안된 것으로 생각된다

해당 문제를 해결하기 위해 OutputStream을 기존 HttpServletResponse의 OutputStream을 가져오게 Controller를 수정하였다 (어차피 파일 다운로드는 content 캐싱이 필요 없으니까)

- Controller
    ```java
    @PostMapping(value = "/download")
    public void downloadPost(
            HttpServletResponse response,
            HomeFilter filter
    ) throws IOException {
        response.setContentType("application/vnd.ms-excel");
        response.setCharacterEncoding("utf-8");
        String fileNameUtf8 = URLEncoder.encode("FAST_EXCEL", StandardCharsets.UTF_8);
        response.setHeader("Content-Disposition", "attachment; filename=" + fileNameUtf8 + ".xlsx");
    
    //  try (OutputStream os = response.getOutputStream()) { // 기존 코드
    // 아래는 새로운 코드
        try (OutputStream os = ((ContentCachingResponseWrapper) response).getResponse().getOutputStream()) { 
            this.homeService.excelDownload(os, filter);
        } catch (IOException e) {
            log.error("error {}", e.getMessage());
        }
    }
    ```


나 같이 ContentCachingResponseWrapper를 사용하는 곳에서만 사용하도록 주의하자  
(casting 에러 나니까)

위와 같이 즉시 다운로드 진행이 되지 않는다면  
중간에 response를 조작하는 코드가 없는지 확인하도록 하자

---

## Reference

- fastexcel - [https://github.com/dhatim/fastexcel](https://github.com/dhatim/fastexcel)
- fastexcel 사용기 - [https://jaimemin.tistory.com/2191?category=1084044](https://jaimemin.tistory.com/2191?category=1084044)
- fastexcel 사용 예제 - [https://github.dev/jaimemin/SampleExcelDownloadProject](https://github.dev/jaimemin/SampleExcelDownloadProject)
- node로 streaming download 구현 - [https://d2.naver.com/helloworld/9423440](https://d2.naver.com/helloworld/9423440)

---

## 끝.
