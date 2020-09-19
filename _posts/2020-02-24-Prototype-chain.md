---
title: 프로토타입 체인
date: 2020-02-24 21:53:00 +0900
categories: [Programming, Javascript]
tags: [Javascript, 프로토타입]
---

자바스크립트의 OOP는 프로토타입을 통해 이루어진다. 자바스크립트의 모든 객체는 기본 객체 타입인 Object 객체가 최상위 타입이 되는데 객체를 생성해보면 Object 객체의 프로토타입에 등록되어 있는 함수인 toString, valueOf 등을 사용할 수 있는 것을 확인할 수 있다. 프로토타입 객체는 생성한 각각의 객체에서부터 최상위 객체인 Object의 프로토타입까지 연결되어있는데 이를 프로토타입 체인이라고 한다.

```javascript
function Person(name) {
    this.name = name;
}

const me = new Person('Changhee');

console.log(me.toString()); //[object Object]
console.log(me.valueOf()); //Person { name: 'Changhee' }
```

<br>

프로토타입 체인을 통한 상속의 경우 속성에 접근할 때 해당 객체부터 그 속성을 가지고 있는지 최상위 Object 까지 프로토타입 체인을 따라 동적으로 찾게 된다.

```javascript
var a = {
    attr1: 'a',
};

var b = {
    __proto__: a,
    attr2: 'b',
};

var c = {
    __proto__: b,
    attr3: 'c',
};

c.attr1; // 'a'
```

<br>

위의 코드는 c -> b -> a 로 연결한 상태이며, c.attr1 이라는 속성에 접근하면 아래와 같은 과정을 수행한다.

1. c 객체에서 `attr1` 속성을 찾는다. (X)
2. c 객체에서 `__proto__` 속성이 있는지 찾는다. (O)
3. c 객체의 `__proto__` 속성이 참조하는 객체로 이동한다. (b로 이동)
4. b 객체에서 `attr1` 속성을 찾는다. (X)
5. b 객체에서 `__proto__` 속성이 있는지 찾는다. (O)
6. b 객체의 `__proto__` 속성이 참조하는 객체로 이동한다. (a로 이동)
7. a 객체에서 `attr1` 속성을 찾는다. (O)
8. 찾은 속성의 값을 리턴한다.

<br>

여기서 어떤 객체에도 존재하지 않는 속성인 `attr0` 을 찾게 되면 7번부터 다른 과정을 거치게 된다.

(7번 부터)

1. a 객체에서 `attr0` 속성을 찾는다. (X)
2. a 객체에서 `__proto__` 속성이 있는지 찾는다. (O)
3. a 객체의 `__proto__` 속성이 참조하는 객체로 이동한다. (`Object.prototype` 로 이동)
4. `Object.prototype` 에서 `attr0` 속성을 찾는다. (X)
5. `Object.prototype` 에서 `__proto__` 속성을 찾는다. (X)
6. undefined 리턴

<br>

이런 점을 이용해 메서드 오버라이드를 구현할 수 있다.

```javascript
var a = {
    method1: function () {
        return 'a1';
    },
};

var b = {
    __proto__: a,
    method1: function () {
        return 'b1';
    },
};

var c = {
    __proto__: b,
    method3: function () {
        return 'c3';
    },
};

a.method1(); // 'a1'
c.method1(); // 'b1'
```

<br>

### 프로토타입 연결하기

**1) Object.create()**

위 코드들에서는 `__proto__` 를 통해 프로토타입을 연결했지만 실제로는 사용해선 안될 코드다. `__proto__` 속성은 **private** 속성인 `[[Prototype]]` 이 자바스크립트로 노출된 것인데, 개발 코드에서 직접적으로 접근하는 것은 피해야 한다.

`Object.create()` 을 이용하면 `__proto__` 속성에 직접 접근하지않고 프로토타입 체인을 연결할 수 있다. `Object.create()` 는 객체를 인자로 받아 그 객체와 프로토타입 체인으로 연결되어 있는 새로운 객체를 리턴해준다.

```javascript
function Person(name) {
    this.name = name;
}

Person.prototype.getName = function () {
    return this.name;
};

function Student(name) {
    Person.call(this, name);
    this.job = 'student';
}

Student.prototype = Object.create(Person.prototype);
Student.prototype.getJob = function () {
    return this.job;
};

const st1 = new Student('Changhee');
console.log(st1.getName()); //Changhee
console.log(st1.getJob()); //student
```

<br>

**2) 생성자 함수**

생성자를 이용해 객체를 생성하면 생성된 객체는 생성자의 프로토타입 객체와 프로토타입 체인으로 연결된다.

```javascript
function Person(name) {
    this.name = name;
}

Person.prototype.getName = function () {
    return this.name;
};

const p = new Person('myName');

console.log(p.getName()); //myName
```

<br>

`Person` 생성자가 만든 인스턴스 p 는 name 이라는 속성을 가지고 있고 getName 이라는 프로토타입 함수를 사용할 수 있다. 그 이유는 p 객체의 `__proto__` 속성이 `Person.prototype` 을 가리키고 있기 때문이다.

new 키워드로 생성자 함수를 실행하면 실제로는 아래와 같은 과정이 진행된다.

```javascript
const p = new Person('myName');

// 엔진 내부에서 하는 일
// 새로운 객체를 생성
p = new Object();
// call 함수를 이용해 Person 함수의 this를 p로 대신해서 실행
Person.call(p, 'myName');
// 프로토타입을 연결
p.__proto__ = Person.prototype;
```

<br>

그 외에 ES6 부터 지원하는 Class 를 이용하는 방법도 있지만 Class 는 별도로 분리하여 공부를 진행한다.

<br>

### 참고자료

[https://meetup.toast.com/posts/104](https://meetup.toast.com/posts/104)

[https://developer.mozilla.org/ko/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain](https://developer.mozilla.org/ko/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain)
