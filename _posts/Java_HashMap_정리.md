# 1\. HashMap

-   데이터를 Key와 Value로 저장하는 자료구조
-   효율적인 검색을 위해 사용된다
-   Key 값을 해시함수로 해싱하여 해당 데이터가 위치한 버킷의 주소값을 찾을 수 있고 이를 통해 바로 찾으려는 데이터에 접근한다.

## 1.1 자바의 HashMap

현재 사용하고 있는 자바8 버전의 HashMap은 해시 충돌을 체이닝 기법을 통해 회피하고 있었는데, 하나의 해시 버킷에 체이닝 된 데이터의 수에 따라 링크드 리스트와 트리를 변경해가며 사용한다는 점이 흥미로웠다.

**링크드 리스트일때 사용하는 데이터 타입**

```java
class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

            //getter, setter, equals...
}
```

**트리일때 사용하는 데이터 타입**

트리는 레드-블랙트리를 사용한다고 한다. ([https://ko.wikipedia.org/wiki/레드-블랙\_트리](https://ko.wikipedia.org/wiki/%EB%A0%88%EB%93%9C-%EB%B8%94%EB%9E%99_%ED%8A%B8%EB%A6%AC))

```java
class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
                ...

                final void treeify(Node<K,V>[] tab) {
          // 링크드 리스트를 트리로 변환한다.
        }

        final Node<K,V> untreeify(HashMap<K,V> map) {
          // 트리를 링크드 리스트로 변환한다.
        }
}
```

링크드 리스트와 트리를 변경하는 기준은 다음과 같이 설정되어 있다.

```java
//8개 이상의 키-값 쌍이 모이면 트리로 변경
static final int TREEIFY_THRESHOLD = 8;

//6개 이하로 떨어지면 링크드 리스트로 변경
static final int UNTREEIFY_THRESHOLD = 6;
```

또 한가지 흥미로웠던 점은, 버킷의 사이즈를 동적으로 확장하는 부분이었는데, 해시 버킷 개수의 기본 값은 16이고, 데이터의 개수가 임계점에 이를 때마다 해시 버킷 개수의 크기를 두 배씩 증가시킨다.

해시 버킷 크기를 두 배로 확장하는 임계점은 현재의 데이터 개수가 'load factor \* 현재의 해시 버킷 개수'에 이를 때이다. 이 load factor는 0.75 즉 3/4이다. 이 load factor 또한 HashMap의 생성자에서 지정할 수 있다.

```java
final float loadFactor;

public HashMap(int initialCapacity, float loadFactor) {...}
```

하지만 버킷 사이즈를 증가 시킬 때 마다 모든 키-값 데이터를 다시 체이닝하는 과정을 거쳐야 하기 때문에 이 부분도 성능적으로 손해를 볼 수 있는 지점이 될 수 있다.

자바는 List, Map 등의 컬렉션 클래스를 사용할 때 내부적으로 알아서 크기를 조절 해주기 때문에 그동안 별도로 컬렉션의 사이즈에 대한 고민없이 기본 값으로 생성하여 사용해 왔었는데, 미리 데이터의 사이즈를 예측이 가능한 상황이라면 컬렉션 클래스를 사용할 때 적절한 사이즈를 미리 지정해 주어서 불필요한 성능 저하를 피할 수 있다.

그동안 성능에 대한 고민보다 편하게 쓸 수 있는 코드를 사용해 왔다는 부분에서 많은 반성을 하게 되었다.

### **Hash 보조함수**

해시 버킷 크기를 두 배로 확장하는 것에는 결정적인 문제가 있다고 한다.

해시 버킷의 개수 M이 2a 형태가 되기 때문에, index = X.hashCode() % M을 계산할 때 X.hashCode()의 하위 a개의 비트만 사용하게 된다는 것이다. 즉 해시 함수가 32비트 영역을 고르게 사용하도록 만들었다 하더라도 해시 값을 2의 승수로 나누면 해시 충돌이 쉽게 발생할 수 있다.

이러한 문제를 피하기 위해 다음과 같은 보조 해시 함수를 사용한다.

```java
static final int hash(Object key) { 
    int h; 
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); 
}  
```

Java 8 HashMap 보조 해시 함수는 상위 16비트 값을 XOR 연산하는 매우 단순한 형태의 보조 해시 함수를 사용한다.

이유로는 두 가지가 있는데,

첫 번째는 Java 8에서는 해시 충돌이 많이 발생하면 링크드 리스트 대신 트리를 사용하므로 해시 충돌 시 발생할 수 있는 성능 문제가 완화되었기 때문이다.

두 번째로는 최근의 해시 함수는 균등 분포가 잘 되게 만들어지는 경향이 많아, Java 7까지 사용했던 보조 해시 함수의 효과가 크지 않기 때문이다. 두 번째 이유가 좀 더 결정적인 원인이 되어 Java 8에서는 보조 해시 함수의 구현을 바꾸었다.

개념상 해시 버킷 인덱스를 계산할 때에는 index = X.hashCode() % M처럼 나머지 연산을 사용하는 것이 맞지만, M값이 2a일 때는 해시 함수의 하위 a비트 만을 취한 것과 값이 같다. 따라서 나머지 연산 대신 '1 << a – 1' 와 비트 논리곱(AND, &) 연산을 사용하면 수행이 훨씬 더 빠르다.

**참고자료**

[https://d2.naver.com/helloworld/83131](https://d2.naver.com/helloworld/831311)

## 1.2 Map vs HashMap

일반적인 Map 자료구조는 **트리(레드-블랙 트리)**를 이용한 TreeMap 자료구조로 구현된다.

HashMap은 해시 함수를 통해 얻어진 위치에 데이터를 저장하기 때문에 정렬을 할 수 없지만 TreeMap은 트리 구조에서 key 값을 기준으로 정렬된 상태로 데이터를 추가할 수 있다.

자바에서 Key값 정렬의 기준을 정해주려면 TreeMap을 생성할 때 Comparator 인터페이스를 생성자로 넣어준다.

Primitive 타입의 key라면 별도로 지정해주지 않았을 경우 natural order를 사용한다.

```java
public TreeMap() { comparator = null; }

public TreeMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
}
```

# 2\. Hash 함수

![https://upload.wikimedia.org/wikipedia/commons/thumb/7/7d/Hash_table_3_1_1_0_1_0_0_SP.svg/1280px-Hash_table_3_1_1_0_1_0_0_SP.svg.png](https://upload.wikimedia.org/wikipedia/commons/thumb/7/7d/Hash_table_3_1_1_0_1_0_0_SP.svg/1280px-Hash_table_3_1_1_0_1_0_0_SP.svg.png)

[https://ko.wikipedia.org/wiki/해시\_테이블](https://ko.wikipedia.org/wiki/%ED%95%B4%EC%8B%9C_%ED%85%8C%EC%9D%B4%EB%B8%94)

-   임의의 길이의 데이터를 고정길이의 데이터로 매핑하는 함수
-   DNA Sequence 패턴 유사도 검사, 암호화, 데이터 무결성 확인(HMAC) 등의 용도로 사용
    -   암호학적 해시함수(Cryptographic Hash Function)와 비암호학적 해시함수로 구분
    -   암호학적 해시함수의 종류로는 MD5, SHA계열 해시함수가 있으며 비암호학적 해시함수로는 CRC32등이 있다.
-   해싱 결과 값이 중복될 수 있다. (해시 충돌)
-   해시 충돌의 확률이 높을수록 서로 다른 데이터를 구별하기 어려워지고 검색하는 비용이 증가

**좋은 해시 함수의 조건**

-   Determinism : 같은 수를 넣을 경우 해시함수는 항상 같은 값을 리턴해야 함. 즉 외부상황에 따라 값이 달라지지 않고, 순수 내부 로직으로 작동
-   Uniformity : input을 집어 넣었을 때, 어떤 output이 나오는 해시 함수가 있다고 하자.이것은 해시의 충돌 확률을 낮추기 위해서다.
-   충돌이 안 일어날 수록 좋은 해쉬 함수.
-   이 때 output range에 있는 각각의 값들이 나올 확률은 서로 대략적(roughly)으로 같아야 한다.
-   결과값이 Bucket Size를 넘지 않도록 범위를 조정할 것.
