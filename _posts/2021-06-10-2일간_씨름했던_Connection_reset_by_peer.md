---
title: 2일간 씨름했던 Connection reset by peer
categories: [개발자로 성장하기]
tags: [회사생활, 개발자, Nginx]
---



현재 담당하고 있는 사내 툴 개발 프로젝트에서 그동안 잘 동작하던 다운로드 API가 오류가 나기 시작했다. 파일의 크기가 약 200KB 이상만 되면 다운로드가 실패하고, 그 이하의 파일들은 다운로드가 정상적으로 동작했다.



<br>



서버 Application에서 찍히는 로그를 보면 `IOException: Connection reset by peer` 익셉션 또는 `IOException: broken pipe` 이 발생하고 있었다. 

```
org.apache.catalina.connector.ClientAbortException: java.io.IOException: Connection reset by peer
        at org.apache.catalina.connector.OutputBuffer.realWriteBytes(OutputBuffer.java:351)
        at org.apache.catalina.connector.OutputBuffer.flushByteBuffer(OutputBuffer.java:776)
        at org.apache.catalina.connector.OutputBuffer.append(OutputBuffer.java:681)
        at org.apache.catalina.connector.OutputBuffer.writeBytes(OutputBuffer.java:386)
        at org.apache.catalina.connector.OutputBuffer.write(OutputBuffer.java:364)
        at org.apache.catalina.connector.CoyoteOutputStream.write(CoyoteOutputStream.java:96)
        at org.apache.commons.compress.utils.IOUtils.copy(IOUtils.java:88)
        at org.apache.commons.compress.utils.IOUtils.copy(IOUtils.java:62)
        .... 생략
```



<br>



`Connection reset by peer` 와 관련된 자료를 찾아보니 해당 Exception이 발생하는 다양한 이유들이 있었다.

- 원격 서버에서 Connection을 reset 처리하거나
- 종료된 커넥션을 재사용하려고 할때
- 클라이언트(브라우저)에서 정지버튼을 누르거나, 브라우저를 종료하거나, 다른 화면으로 이동하는 등의 이유로 서버 측에서 작업 결과를 전달할 곳이 없어졌을 때
- Connection에서 Timeout 발생
- 메모리부족
- 소켓 고갈 등등...



<br>



## 2일간의 삽질 로그...

처음엔 스프링 서버에서 Exception을 띄우고 있었기 때문에 스프링 서버쪽의 문제로 생각하고 몇가지 설정들을 추가하거나 수정해주었고 배포된 서버도 재부팅하여 서버 Resource들을 리셋해주었다. 

- jvm 옵션 추가
  - heap memory : default -> 4g
  - maxConnections : default -> 100
- 조회용 Connection은 ConnectionPool을 사용하는 WebClient를 사용하도록 변경
  - 다운로드 시에는 HttpUrlConnection 클래스를 사용해 스토리지 서버와의 Connection만 생성하고 InputStream을 Response의 OutputStream으로 바로 복사해주고 있다.



<br>



그래도 해결이 되지 않아서 실제 파일을 받아오는 스토리지 서버쪽 이슈가 있는지 개발자분께 문의 진행해보았지만 curl 등의 요청으로는 정상 동작하여 스토리지 서버 문제는 아닌 것으로 확인되었다.

그 외에도 테스트를 해보았지만 특별히 문제가 될 만한 부분은 찾기가 어려웠다.

- 설정된 타임아웃이 지나기도 전에 API가 호출되자마자 실패가 발생하므로 타임아웃 문제 X

- Postman으로 API를 호출했을 때도 동일한 문제가 발생하므로 클라이언트 문제 X



<br>



해결된 건 없는데 너무 많은 시간을 할애하고 있는 것 같아서 일단은 동일한 문제가 발생하지 않는 새로운 서버에 배포하는 방법으로 처리해두었다.

이 문제를 그냥 넘어가기엔 새로운 서버에서도 다시 발생할 수도 있고 찝찝함이 남아 조건을 변경해보며 이리저리 테스트를 해보고 있었는데, Nginx에서 스프링 서버로 proxy 하는 것을 이용하지 않고 직접 스프링 서버로 API를 호출해보니 다운로드가 정상적으로 동작하고 있었다. 🤔

그렇다는 것은 Nginx쪽 문제였다는 것이므로, Nginx의 로그파일을 확인해보았다.

![스크린샷 2021-06-10 오후 11.08.11](https://user-images.githubusercontent.com/17294694/122771596-8d001700-d2e1-11eb-8cc3-545f0c83af09.png)

다운로드 API 호출을 Nginx에서 proxy 할때  `/var/cache/nginx/proxy_temp/...` 라는 경로에 있는 임시파일들을 참조하고 있었는데 권한이 없어 Permission denied 오류가 발생하고 있었다.

해당 경로의 접근 권한을 확인해보니 `nginx:nginx`  로 되어 있었다.

![스크린샷 2021-06-10 오후 11.07.49](https://user-images.githubusercontent.com/17294694/122771633-938e8e80-d2e1-11eb-8509-9dbfd5fe6275.png)

Nginx의 기본 user값은 nginx인데, 배포 설정을 진행하던 당시 Nginx에서 빌드 결과물에 접근 권한을 가질 수 있도록 user를 다른 값으로 변경해놓았던 것이 문제였다.

`/etc/nginx/nginx.conf.d` 파일의 user 값을 다시 원래대로 nginx로 돌려놓고 빌드 결과물을 Nginx 기본 root인 `/usr/share/nginx/html` 하위에 생성되도록 수정했다. (기존 path 그대로 사용하고 group 권한으로 처리하려고 했더니 403이 아닌 500 permission denied 오류가 발생했다.)

그 다음 API를 호출해보니 문제가 드디어 해결되었다.. 😭

2일동안 원인을 찾기위해 이리저리 코드도 수정해보고 jvm 옵션도 추가해보고 리눅스 서버 로그까지 확인했었는데 Nginx쪽 로그는 왜 확인할 생각을 못했을까...



<br>



## Nginx Reverse Proxy를 사용한다면

스프링 서버로의 접속을 스프링 서버에서 직접 받는 것이 아니라 Nginx를 통해 Reverse 프록시 처리를 한다면 주의해야 할 부분이 있다. 

예전에 개인 프로젝트를 진행했을때 비슷한 경험이 있던 것이 떠올랐는데, 업로드 기능을 구현할 때 스프링 서버에 `multipart.maxFileSize` 값을 잘 설정해놓았음에도 불구하고 413 (Payload Too Large) 에러가 발생했었다. 

그 이유는, 스프링 서버로 프록시해주는 Nginx에서 그 크기만큼의 Payload를 받을 수 있도록 설정되어 있지 않았기 때문이었다.

Nginx 설정은 한번 설정해두면 크게 건들일이 없기 때문에 간과하기가 쉬운데, 스프링 서버에서 네트워크 관련된 설정이 적용되는 경우 Nginx에도 적절한 값으로 설정이 되어있는지 확인해 볼 필요가 있다. (네트워크 관련 문제가 생겼을 때도 Nginx를 꼭 같이 확인해보자...)







  
