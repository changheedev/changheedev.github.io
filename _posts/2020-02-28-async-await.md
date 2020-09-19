---
title: async/await
date: 2020-02-28 21:28:00 +0900
categories: [Programming, Javascript]
tags: [Javascript, async-await]
---

콜백 방식의 단점들을 개선하기 위해 Promise 스펙이 추가되었지만 여전히 아래와 같이 복잡한 구조가 발생할 수 있다.

**Promise**

```javascript
doSomething().then((result) => {
    afterDoSomething(result).then((result) => {
        finalDoSomething(result).then((result) => {
            // ...
        });
    });
});
```

ES8 에서 추가된 async / await 를 사용하면, Promise 에 비해 코드 가독성이 향상되고 코드를 동기적 코드처럼 작성할 수 있다.

**async / await**

```javascript
async function getResult() {
    const result1 = await doSomething();
    const result2 = await afterDoSomething(result1);
    const result3 = await finalDoSomething(result2);
    //...
}
```

## 1. async

**async function** 은 **AsyncFunction** 객체를 반환하는 비동기 함수를 정의한다. AsyncFunction 객체는 이벤트 루프를 통해 비동기적으로 동작하며, Promise 를 사용하여 결과를 반환한다.

```javascript
async function asyncFn() {
    console.log('async call');
}

asyncFn().then(() => console.log('async callback'));
console.log('after async');

/*
async call
after async
async callback
*/
```

## 2. await

async 함수에는 await 식이 포함될 수 있다. await 키워드를 사용하면 Promise 에서 resolve 또는 reject 를 호출하여 해결될 때 까지 **기다리는 상태**가 되며, 여러 Promise 의 작업을 동기스럽게 사용할 수 있게 된다.

await 키워드는 async 함수 바로 아래에서만 유효하며 async 함수 내부의 다른 함수에서 호출될 경우 에러가 발생한다.

```javascript
function resolveAfter2Seconds() {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve("resolved");
    }, 2000);
  });
}

async function asyncCall() {
  console.log("calling");
  const result1 = await resolveAfter2Seconds(); //resolve 가 호출될 때까지 2초간 기다린다.
  console.log(result1); //resolved
}


//아래의 경우는 에러
async function asyncCall2() {
  console.log("calling");

  function inner(){
    const result2 = await resolveAfter2Seconds(); //error
    console.log(result2);
  }
  inner();
}
```

만약, await 키워드 다음에 오는 표현식의 값이 Promise가 아닌 경우 해결 된 Promise로 변환된다.

```javascript
async function asyncFn() {
    const nonPromise = await 'non-promise';
    console.log(nonPromise); //non-promise
}
```

## 3. await 사용 방식에 따른 실행 순서 확인

1초의 타이머 이후 resolve 를 호출하는 resolveAfter1Seconds 함수와 2초의 타이머 이후 resolve 를 호출하는 resolveAfter2Seconds 함수가 있다.

```javascript
const resolveAfter1Second = function () {
    console.log('starting fast promise');
    return new Promise((resolve) => {
        setTimeout(function () {
            resolve(10);
            console.log('fast promise is done');
        }, 1000);
    });
};

const resolveAfter2Seconds = function () {
    console.log('starting slow promise');
    return new Promise((resolve) => {
        setTimeout(function () {
            resolve(20);
            console.log('slow promise is done');
        }, 2000);
    });
};
```

아래와 같이 resolveAfter1Seconds 함수와 resolveAfter2Seconds 함수를 호출할 때 await 를 사용하면 resolveAfter2Seconds 함수에서 반환된 Promise 가 resolve 를 호출하고 난 뒤에 resolveAfter1Seconds 함수가 **순차적으로 실행**된다.

```javascript
const sequentialStart = async function () {
    console.log('==SEQUENTIAL START==');

    const slow = await resolveAfter2Seconds();
    console.log(slow);

    const fast = await resolveAfter1Second();
    console.log(fast);
};

sequentialStart();

/*
==SEQUENTIAL START==
starting slow promise
slow promise is done
20
starting fast promise
fast promise is done
10
*/
```

그렇다면, 다음의 경우는 어떤 순서로 실행될까?

```javascript
const concurrentStart = async function () {
    console.log('==CONCURRENT START with await==');
    const slow = resolveAfter2Seconds(); // starts timer immediately
    const fast = resolveAfter1Second();

    console.log(await slow);
    console.log(await fast); // waits for slow to finish, even though fast is already done!
};

concurrentStart();

/*
==CONCURRENT START with await==
starting slow promise
starting fast promise
fast promise is done
slow promise is done
20
10
*/
```

위 예제에서는 resolveAfter2Seconds 함수와 resolveAfter1Seconds 함수를 호출할때는 await 를 사용하지 않았다.

따라서 백그라운드에서 resolveAfter2Seconds 함수의 타이머가 실행되는 동안 resolveAfter1Seconds 함수의 타이머도 **병렬 실행**된다.

slow, fast 에는 각각 resolveAfter2Seconds, resolveAfter1Seconds 에서 반환된 Promise 객체가 저장되어 있고 **resolve 호출을 기다리고 있는 상태**가 된다. 이 상태에서 await 로 resolve 를 기다리지 않고 slow 와 fast 에 저장된 값을 출력해보면 pending 상태의 Promise 객체가 출력된다.

하지만 console.log 에서 await 를 사용하여 resolve 호출이 완료될 때까지 대기하도록 만들었기 때문에 정상적으로 20 과 10이 출력되고, fast 가 먼저 완료 되었더라도, slow 가 출력될 때까지 대기 상태가 되는 것을 볼 수 있다.

두 예제의 실행 시간을 비교해보자.

```javascript
const measureExecutionTimes = async () => {
  console.time("sequentialStart");
  await sequentialStart();
  console.timeEnd("sequentialStart");

  console.time("concurrentStart");
  await concurrentStart();
  console.timeEnd("concurrentStart");
};

measureExecutionTimes();

//두 함수의 실행시간 비교
sequentialStart: 3009.454ms
concurrentStart: 2003.013ms
```

sequentialStart 함수는 2초의 타이머와 1초의 타이머가 순차적으로 실행되어 총 3초의 시간이 걸리지만 concurrentStart 함수는 타이머가 병렬적으로 실행되어 총 2초의 시간이 걸린다.

## 4. Error handling

async / await 의 에러 핸들링은 **try catch** 문으로 감싸서 처리해준다.

```javascript
const rejectAfter1Seconds = function () {
    return new Promise((resolve, reject) => {
        setTimeout(function () {
            reject('Error occurred in rejectAfter1Seconds');
        }, 1000);
    });
};

const asyncFn = async () => {
    try {
        const result1 = await rejectAfter1Second();
        const result2 = await resolveAfter2Second();
    } catch (err) {
        console.error(err);
    }
};

asyncFn();

/*
Error occurred in rejectAfter1Seconds
*/
```

## 5. 주의 사항

async function 이 반환하는 AsyncFunction 객체는 이벤트 루프를 통해 비동기적으로 동작하고, Promise 를 사용하여 결과를 반환한다.

그 말은 async function 으로 정의 된 함수를 호출했을 때에는 Promise 로 처리를 해주어야 한다는 것이다.

sequentialStart 함수에 slow 와 fast 의 결과값을 더한 뒤 반환하는 로직을 추가한 다음, 결과값을 출력해보자. 만약, async 함수의 내부에서 호출되는 Promise 에만 신경쓰게 된다면, 다음과 같은 실수를 할 수 있다.

```javascript
const sequentialStart = async function () {
    console.log('==SEQUENTIAL START==');

    const slow = await resolveAfter2Seconds();
    console.log(slow);

    const fast = await resolveAfter1Second();
    console.log(fast);

    return slow + fast;
};

const result = sequentialStart();
console.log(result);
```

sequentialStart 함수는 비동기적으로 Promise 를 사용하여 결과값을 반환하기 때문에 기대했던 slow + fast 값을 출력하기 위해서는 아래와 같이 변경해주어야 한다.
sequentialStart 함수를 호출하는 부분이 다른 async function 내부라면 await을 사용할 수도 있다.

```javascript
sequentialStart().then((result) => console.log(result));

//또는
const result = await sequentialStart();
```

## 참고 자료

[https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Statements/async_function](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Statements/async_function)

[https://www.zerocho.com/category/ECMAScript/post/58d142d8e6cda10018195f5a](https://www.zerocho.com/category/ECMAScript/post/58d142d8e6cda10018195f5a)

[http://www.itworld.co.kr/t/62086/오픈소스/113430](http://www.itworld.co.kr/t/62086/오픈소스/113430)

[https://mariusschulz.com/blog/measuring-execution-times-in-javascript-with-console-time](https://mariusschulz.com/blog/measuring-execution-times-in-javascript-with-console-time)
