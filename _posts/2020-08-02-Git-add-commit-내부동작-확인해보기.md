---
title: Git add와 commit 내부동작 확인해보기
categories: [Programming, Git]
tags: [git, add, commit]
---

## 1\. Object

Git은 데이터를 저장할 때 데이터와 헤더로 생성한 SHA-1 체크섬으로 파일 이름을 짓는다. 해시의 처음 두 글자를 따서 디렉토리 이름에 사용하고 나머지 38글자를 파일 이름에 사용하는데 이 파일은 `.git/objects` 경로 아래에 저장된다.

**Git의 Object에는 3가지 타입이 있다.**

-   파일의 내용을 담는 (blob)
-   디렉토리의 파일명과 내용에 해당되는 블록의 정보 (tree)
-   커밋정보 (commit)

### 1.1 Blob

-   파일의 내용을 저장하는 object
-   파일 내용을 SHA-1 알고리즘으로 해시하고 `.git/objects` 경로 아래에 저장한다.

![R1280x0-2.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-2.png)

-   처음 두 글자를 따서 디렉토리 이름에 사용하고 나머지 38글자를 파일 이름에 사용한다.

![R1280x0-3.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-3.png)

-   파일의 추가되거나 변경되면 index 파일에 기록된다.
-   파일의 내용을 해시한 값을 사용하기 때문에 파일 이름이 달라도 파일 내용이 같다면, `같은 오브젝트`를 가르킨다.

![R1280x0-4.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-4.png)

### 1.2 Tree

-   커밋 시점의 Index 정보(stage)를 스냅샷으로 저장한다.

![R1280x0-5.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-5.png)

-   Blob과 마찬가지로 `.git/objects` 경로 아래에 저장된다.

![R1280x0-6.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-6.png)

### 1.3 Commit

-   커밋 정보를 저장한다.

    -   스냅샷 정보 (tree)
    -   이전 커밋 정보 (parent)
    -   커밋 메시지, author, 시간 등

![R1280x0-7.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-7.png)

-   Blob과 마찬가지로 `.git/objects` 경로 아래에 저장된다.

![R1280x0-8.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-8.png)

## 2\. Add의 동작

### 2.1 새로운 파일 등록

`git add [파일명]` 으로 파일을 `stage`에 올리면 파일을 blob 오브젝트로 저장한다. 저장된 파일 정보는 `.git/index` 파일에 `[파일모드] [해시] [파일명]` 형태로 기록된다.

![R1280x0-9.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-9.png)

파일을 하나 더 추가하면 index 파일에 2개의 파일 정보가 기록된 것을 확인 할 수 있다.

![R1280x0-10.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-10.png)

### 2.2 기존 파일 수정

기존의 파일이 변경되면 변경된 파일을 다시 SHA-1 알고리즘으로 해시하고 새로운 blob 오브젝트로 저장한다.

index 파일에 해당 파일의 해시값을 새로운 해시값으로 변경한다.

![R1280x0-11.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-11.png)

파일 이름이 변경된 경우 같은 해시값으로 변경된 파일 정보가 기록된다. (파일의 내용으로 해시를 하기 때문에 이름이 바뀌어도 해시값은 변하지 않는다)

![R1280x0-12.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-12.png)

## 3\. Commit의 동작

### 3.1 **commit**

**1) 스냅샷 생성**

-   `git commit -m "commit message"` 가 실행되면 새로운 tree 오브젝트를 생성하고 커밋된 시점의 index 정보를 저장한다. (스냅샷 생성)
-   생성된 tree 오브젝트를 SHA-1 알고리즘으로 해시하고 `.git/objects` 디렉토리에 저장한다.

**2) 커밋 생성**

-   커밋 정보를 저장하는 **commit object**를 생성한다.
-   commit object를 SHA-1 알고리즘으로 해시하고 `.git/objects` 디렉토리에 저장한다.

![R1280x0-13.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-13.png)

-   두 번째 커밋부터 이전 커밋 정보인 parent 정보가 포함된다.

![R1280x0-14.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-14.png)

Git이 저장하는 데이터의 구조를 전체적으로 보면 아래 그림과 같다.

![R1280x0-15.png](/assets/img/posts/2020-08-02-Git-add-commit-내부동작-확인해보기/R1280x0-15.png)

**참고자료**

[https://opentutorials.org/course/2708/15238](https://opentutorials.org/course/2708/15238)

[https://opentutorials.org/course/2708/15240](https://opentutorials.org/course/2708/15240)

[https://git-scm.com/book/ko/v2/Git의-내부-Git-개체](https://git-scm.com/book/ko/v2/Git%EC%9D%98-%EB%82%B4%EB%B6%80-Git-%EA%B0%9C%EC%B2%B4)
