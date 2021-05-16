Gradle 3.0 에서 complie 키워드가 implementation 과 api 두 가지로 분리되었다.

두 경우의 차이를 구글링해보면 아래와 같이 설명하고 있다.

**implemetation**

> Gradle은 종속성을 컴파일 클래스 경로에 추가하여 종속성을 빌드 출력에 패키징합니다. 다만 **모듈이 implementation 종속성을 구성하는 경우, 이것은 Gradle에 개발자가 모듈이 컴파일 시 다른 모듈로 유출되는 것을 원치 않는다는 것을 알려줍니다. 즉, 종속성은 런타임 시 다른 모듈에서만 이용할 수 있습니다.**
> 
> api 또는 compile(지원 중단됨) 대신 이 종속성 구성을 사용하면 빌드 시스템이 재컴파일해야 하는 모듈의 수가 줄어들기 때문에 빌드 시간이 상당히 개선될 수 있습니다. 예를 들어, implementation 종속성이 API를 변경하면 Gradle은 해당 종속성과 그 종속성에 직접 연결된 모듈만 다시 컴파일합니다. 대부분의 앱과 테스트 모듈은 이 구성을 사용해야 합니다.

**api**

> Gradle은 컴파일 클래스 경로 및 빌드 출력에 종속성을 추가합니다. **모듈에 api 종속성을 포함하면 다른 모듈에 그 종속성을 과도적으로 내보내기를 원하며 따라서 런타임과 컴파일 시 모두 종속성을 사용할 수 있다는 사실을 Gradle에 알려줄 수 있습니다.**
> 
> 이 구성은 compile(현재 지원 중단됨)과 똑같이 동작합니다. 다만 이것은 주의해서 사용해야 하며 다른 업스트림 소비자에게 일시적으로 내보내는 종속성만 함께 사용해야 합니다. 그 이유는 api 종속성이 외부 API를 변경하면 Gradle이 컴파일 시 해당 종속성에 액세스할 권한이 있는 모듈을 모두 다시 컴파일하기 때문입니다. 그러므로 api 종속성이 많이 있으면 빌드 시간이 상당히 증가합니다. 종속성의 API를 별도의 모듈에 노출시키고 싶은 것이 아니라면 라이브러리 모듈은 implementation 종속성을 대신 사용해야 합니다.

즉, implementation 을 사용하면 import 된 모듈의 종속성을 상위 모듈에서 접근할 수 없으며, 해당 모듈이 변경되더라도 직접 연결된 모듈만 재컴파일 하면 되지만, api 를 사용하면 상위 모듈에서 해당 모듈에 접근할 수 있고, 해당 모듈이 변경되면 의존성을 갖는 모든 모듈을 재컴파일 해야 한다는 것이다.

그동안 implementation 만 사용하더라도 큰 문제가 없었고 글로만 봐서는 정확히 어떻게 구분해서 써야하는지 이해하기가 어려웠었다.

그런데 이번에 프로젝트를 멀티 모듈로 변경하는 과정에서 api가 필요한 경우를 직접 체험해볼 수 있었고, 글로 남기려고 한다.

먼저, 도메인 엔티티에서 발생하는 공통 속성을 BaseEntity 클래스에 모으고 domain-core 라는 모듈로 분리시켜 놓았다.

```java
//domain-core
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name="CREATE_AT", nullable = false, updatable = false)
    @CreatedDate
    private LocalDateTime createAt;

    @Column(name="UPDATE_AT", nullable = false)
    @LastModifiedDate
    private LocalDateTime updateAt;
}
```

그리고, 도메인 모듈에서 기존에 사용했던 방법대로 implementation 을 사용하여 domain-core 모듈을 추가하고, BaseEntity 를 상속받아 필요한 엔티티를 구현한다.

```groovy
//build.gradle
bootJar { enabled = false }
jar { enabled = true }

dependencies {
    implementation project(':domain-core')
}

//User Entity
public class User extends BaseEntity {...}
```

그 다음, 도메인 모듈에 의존성을 갖는 서비스 모듈을 구현하면 아래와 같은 오류를 만날 수 있다.

[##_Image|kage@dc7TDN/btqDnwVRS8M/6E9NvXPSUveoKVrDavmAm0/img.png|alignCenter|data-origin-width="629" data-origin-height="274" data-ke-mobilestyle="widthOrigin"|||_##]

여기서 모듈들의 의존 관계는 아래와 같다.

```
domain-core <= domain-user <= domain-account-service
```

그런데, domain-user 모듈에서 domain-core 모듈을 implementation 으로 추가했기 때문에 domain-account-service 모듈에서는 domain-core 모듈에 접근할 수 없게 된다.

오류를 해결하기 위해서는 domain-account-service 모듈에도 domain-core 모듈을 추가해주는 방법과, implemetation 대신 api 를 사용하여 상위 모듈에서도 접근할 수 있도록 해주는 방법이 있다.

api 로 변경하려면 domain-core 모듈의 변경이 빈번한지에 대해 생각해보아야 하는데, domain-core 모듈이 수정되는 일은 거의 없기 때문에 api로 변경하더라도 큰 문제가 되진 않아보였다.

하지만, 명시적으로 추가할 경우 domain-account-service 모듈에서 domain-core 모듈을 직접적으로 사용하지 않음에도 불구하고, domain-core 모듈과 domain-user 모듈 사이의 의존성에 관여하게 되는 것처럼 보여서 고민이 되었고, 명시적으로 추가하면 결국 api 와 똑같이 재컴파일 대상에 포함되기 때문에 결론적으로는 api 로 변경하는 것으로 결정하게 되었다.

api 키워드를 사용하기 위해서는 java-library 플러그인을 추가해주어야 한다.

```groovy
//루트 build.gradle
subprojects {
    apply plugin: 'java-library' //추가
        ...
}


//도메인 모듈 build.gradle
dependencies {
    api project(':domain-core') //api로 변경
      ...
}
```

이번 같은 경우는 변경 가능성이 매우 낮은 모듈에 대해서 api 키워드를 사용하는 경우지만, 변경이 빈번하게 일어나는 모듈에 대해서 api 키워드를 사용한다면 프로젝트 규모가 커지고 모듈들의 의존 관계가 길어질 경우 빌드타임이 굉장히 길어지는 문제가 발생하게 될 것이다.

한 [블로그](https://medium.com/mindorks/implementation-vs-api-in-gradle-3-0-494c817a6fa)의 저자가 얘기하는 것처럼 대부분의 경우에는 implementation 을 사용하여 의존 관계를 최대한 줄이는 방향으로 사용하는 것이 좋을 것 같다.

> Just replace all `compile` with the `implementation` and try to build the project. If it builds successfully, well and good. otherwise look for any leaking dependency you might be using and import those libraries using `api` keyword.
> 
> 모든 `compile`을 `implementation`으로 바꾸고 프로젝트를 빌드해보세요.  
> 만일 여러분이 성공적으로 잘 된다면 훌륭한 프로젝트 입니다.  
> 그렇지 않으면 종속성이 있는지 찾아보고 `api`키워드를 사용하여 해당 라이브러리를 사용합시다.
