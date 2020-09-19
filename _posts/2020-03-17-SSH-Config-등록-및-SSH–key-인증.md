---
title: SSH Config 등록 및 SSH key 인증
date: 2020-03-17 00:48:00 +0900
categories: [Mac]
tags: [Mac, ssh, ssh-key]
---

## SSH 접속 정보 Config 등록

SSH 접속 정보를 config 로 등록해두면 IP를 매번 입력하지 않고 접속할 수 있다.

```
$ sudo nano ~/.ssh/config

>> 내용 입력
Host <접속시 사용할 이름>
HostName <ip 주소>
User <계정 이름>

//예시
Host my-remote-server
HostName 123.123.123.123
User changhee
```

config 등록 후 `ssh <Host>` 로 접속한다.

```
$ ssh my-remote-server

password:
```

## SSH key 인증

맥의 터미널에서 라즈베리파이에 접속할 때 매번 비밀번호를 입력하는 대신 SSH 키를 이용하여 인증하도록 설정할 수 있다.

### 기존 키 확인

맥의 터미널에서 아래 명령어를 실행 했을 때 **id_rsa.pub** 또는 **id_dsa.pub** 파일이 존재한다면 새로운 키를 생성하지 않아도 된다.

```
ls ~/.ssh
```

### 새로운 키 생성

다음 명령어를 실행하여 키 생성 작업을 시작한다.

```
$ ssh-keygen
```

명령어를 실행하면 키를 저장할 위치를 물어본다. 엔터를 입력하면 기본 위치(~/.ssh) 에 저장된다.

다음으로는 패스워드를 등록할 것인지 묻는데 패스워드를 등록한 다음 엔터를 눌러 넘어간다. (패스워드를 사용하지 않으려면 입력하지 않고 바로 엔터를 눌러 넘어간다.)

여기까지 완료하면 키 생성이 시작되고 ~/.ssh 디렉토리에 **id_rsa** 파일과 **id_rsa.pub** 파일이 생성된다.

### 공개키 등록하기

ssh-copy-id 명령어를 사용하여 라즈베리파이에 공개키를 등록한다.

username 과 ip-address 는 라즈베리파이의 계정 id와 아이피 주소를 사용한다.

```
$ ssh-copy-id <USERNAME>@<IP-ADDRESS>
```

만약 ssh-copy-id 명령어를 사용할 수 없다면 직접 등록해주어야 한다.

```
$ cat ~/.ssh/id_rsa.pub | ssh <USERNAME>@<IP-ADDRESS> 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'
```

키 등록을 완료하면 ssh 접속을 다시 시도하여 비밀번호를 물어보는지 확인한다.
