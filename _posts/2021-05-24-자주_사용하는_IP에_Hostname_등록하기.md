---
title: 자주 사용하는 IP에 Hostname 등록하기
categories: [Mac]
tags: [Mac, IP, Hostname]
---

터미널에서 `/etc/hosts`를 편집기로 실행한다.

```console
$ sudo vi /etc/hosts
```

다음과 같은 화면이 나오면 localhost 의 내용 밑에 등록하고자 하는 IP와 원하는 호스트이름을 입력한다.

```bash
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1       localhost
255.255.255.255 broadcasthost
::1             localhost

127.0.0.1       samplehost # 새로 추가된 내용
```

변경된 내용이 적용되도록 터미널에서 아래 명령어를 실행.

```console
$ dscacheutil -flushcache
```

이렇게 Host를 등록해주면 맥에서 해당 IP를 입력해야 할때 IP 주소대신 Hostname으로 대신할 수 있다.

```console
$ ping samplehost

> PING samplehost (127.0.0.1): 56 data bytes
> 64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.069 ms
> 64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.073 ms
> 64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.125 ms
> 64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.157 ms
```
