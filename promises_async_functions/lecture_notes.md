# Асинхронность в JavaScript: Promises & Async functions

_Примечение: нам не известен устойчивый перевод на русский или украинский слов "resolve", "fullfill", "reject" в контексте промисов, поэтому за неимением лучшего будем писать "выполниться", "выполниться успешно", "выполниться с ошибкой", но иногда придется писать "зарезолвиться" и "зареджектиться". Нам тоже не нравится._

Вспомогательные функции (будут использоваться дальше в примерах кода):
`wait(ms)` возвращает промис, который успешно выполняется(резолвится) через указанное количество миллисекунд:

```javascript
function wait(ms) {
  return new Promise((r) => setTimeout(r, ms));
}
```

## Промис

**Промис** - это сущность, которая представляет результат асинхронной операции. Промис умеет выполнять только одно действие - изменить свое состояние и вызвать обработчики, зарегистрированные на такое изменение состояния. С промисом мы можем взаимодействовать только при помощи обработчиков, а значение промиса доступно нам только внутри зарегистрированных через метод `.then` обработчиков (исключение - асинхронные функции, о них позже).

Когда промис изменит свое состояние? Нам это неизвестно. Если вы работаете с API, которое возвращает промисы (например, [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)), то вам не нужно об этом заботиться. Вам нужно только зарегистировать свои обработчики на полученном промисе. Если вы создаете промис самостоятельно, то вы определите этот момент вызовом колбеков в конструкторе промиса.

```javascript
const myPromise = new Promise((resolve, reject) => {
  // any code here
  if (isOk) {
    resolve(data); // тут мы определяем, когда промис успешно завершится
  } else {
    reject(error); // тут мы определяем, когда промис завершится с ошибкой
  }
});
```

### Состояние промиса

Промис может находиться в одном из трех состояний. При создании промис находится в состоянии `pending` (в ожидании). Из состояния `pending` он переходит в состояние `fullfilled` (выполнился успешно) либо `rejected` (выполнился с ошибкой). Состояния `fullfilled` и `rejected` вместе называются `settled` (завершен или выполнился).

Promise => **pending** => settlement => **fullfilled**/**rejected**

Промис может изменить состояние (выполниться) только один раз, больше он свое состояние не меняет.

Внимание :point_up: в коде мы не можем получить доступ к состоянию промиса, но обычно нам это и не нужно. Об изменении состояния мы можем узнать косвенно, по вызову зарегистрированных колбеков. Однако в консоли браузера мы можем увидеть состояние промиса благодаря дополнительным возможностям инструментов разработчика. Это удобно при отладке. 

Попробуйте:

```javascript
// создаем промис, который успешно выполнится через 5 секунд
const myPromise = new Promise((resolve, reject) => setTimeout(resolve, 5000));
myPromise;
/* выводим промис в консоль сразу после создания. Ожидаемый результат - Promise {<pending>} (промис в состоянии ожидания)
ждем 5 секунд и снова выводим промис в консоль */
myPromise;
/* ожидаемый результат: Promise {<resolved>: undefined} (успешно выполнившийся промис со значением undefined) */
```

**Реакции на промис**- это колбеки, которые регистрируются на промисе при помощи метода `then`. Эти колбеки будут вызваны, когда промис войдет в нужное состояние (успешно выполнится или завершится с ошибкой).

```javascript
// создадим промис, который успешно выполнится через 5 секунд
const myPromise = new Promise((resolve, reject) => {
  setTimeout(() => resolve(42), 5000);
});

// Добавим реакции (колбеки на изменение состояния промиса)

myPromise.then(console.log); // этот колбек выполнится через приблизительно 5 секунд и выведет в консоль 42
myPromise.then(console.log); // и этот тоже
```

Если обработчики регистрируются на промисе, который уже завершился, они будут вызваны немедленно с тем значением, с которым завершился промис.

```javascript
// создадим успешно завершившийся промис

const myResolvedPromise = Promise.resolve(5);

// Добавим реакции (колбеки на изменение состояния промиса)

myResolvedPromise.then(console.log); // этот колбек выполнится сразу и выведет в консоль 5
myResolvedPromise.then(console.log); // и этот тоже


// Создадим промис, завершившийся с ошибкой

const myRejectedPromise = Promise.reject("oops");

// Добавим реакции (колбеки на изменение состояния промиса)

myRejectedPromise.then(
  (value) => console.log(value),
  (err) => console.log(err) // этот колбек выполнится сразу и выведет 'oops' в консоль
);

myRejectedPromise.catch(console.log); // и этот тоже
```

А вот этот промис никогда не завершится, а зарегистрированные на нем обработчики никогда не выполнятся:

```javascript
const myPromise = new Promise((resolve, reject) => {});
myPromise.then(console.log);
```

:question: :question: :question: Почему этот промис никогда не выполнится?

### Цепочка промисов

Каждый вызов метода `.then` на промисе возвращает новый промис, у которого есть свое собственное состояние. У этого промиса также есть метод `.then`, с помощью которого мы можем зарегистрировать на нем свои обработчики, которые выполнятся, когда второй промис изменит свое состояние. Вызов метода `.then` также вернет новый промис, и так далее. Таким образом мы объединяем промисы и можем составлять из них цепочку асинхронных операций, которые должны выполняться последовательно. На что нужно обратить внимание:

:point_up: Каждый вызов `.then` возвращает новый промис:

```javascript
const promise1 = new Promise((r) => r(1));
const promise2 = promise1.then((x) => x + 1);
const promise3 = promise2.then((x) => x + 1);
promise1; //Promise {<resolved>: 1}
promise2; // Promise {<resolved>: 2}
promise3; // Promise {<resolved>: 3}
promise1 === promise2; // false
promise2 === promise3; //false
```

:point_up: Остановить выполнение цепочки промисов невозможно. Если промис изменил свое состояние, зарегистрированные на нем обработчики неизбежно выполнятся. Это нужно учитывать при формировании цепочки помисов. Например, нужно предусмотреть такой сценарий в последующих обработчиках, передавать вместе с ответом статус выполнения запроса и не выполнять последующие запросы на сервер, если из предыдущего колбека пришло сообщение об ошибке. В упрощенном варианте цепочка запросов может выглядеть так:

```javascript
getUserIdFromApi(token).then(({id, error}) => {
  if( === null) {
    return getUserDataFromApi(id)
  } esle {
    return handleError(error)
  }
}).then(sendResponse);
```

### Методы промисов

#### Promise.resolve()

`Promise.resolve()` возвращает промис, успешно выполнившийся (зарезолвленный) с переданным значением.

```javascript
const value = 42;
Promise.resolve(value).then(console.log); // в консоль выведено 42
```

Если в качестве аргумента `Promise.resolve` передан промис, этот промис будет возвращен.

```javascript
const myPromise = new Promise((r) => r());
Promise.resolve(myPromise) === myPromise; // true;
```

Внимание :point_up: `Promise.resolve(somePromise)` никак не влияет на состояние промиса `somePromise`, переданного в качестве аргумента. `somePromise` имеет свое собственное состояние и изменить его снаружи невозможно.

```javascript
const myPromise = new Promise((r) => {}); // создаем промис, который никогда не выполнится
myPromise; // Promise {<pending>} (помис в состоянии ожидания)
const newPromise = Promise.resolve(myPromise);
newPromise === myPromise; // true
newPromise; // Promise {<pending>}, состояние промиса не изменилось
```

:question: :question: :question: Что будет, если передать `Promise.resolve` в качестве аргумента промис, завершившийся с ошибкой? Проверьте свое предположение в браузерной консоли.

```javascript
const myRejectedPromise = Promise.reject('oops');
Promise.resolve(myRejectedPromise); // ???
```

`Promise.resolve` может пригодиться, если нужно сделать значение асинхронным. Если дальнейший код ожидает промис, то это также удобный способ сделать некое значение гарантированно промисом: если значение было изначально промисом, с ним ничего не случится, если нет - значение будет обернуто в промис.

#### Promise.reject()

Возвращает промис, зареджектившийся (выполненный с ошибкой) с переданным значением.

```javascript
const myPromise = Promise.reject("oops");
myPromise.catch(console.log); // oops;
```

#### Promise.all

Если нужно выполнить несколько промисов параллельно, берите Promise.all. Promise.all принимает в качестве аргумента массив промисов и возвращает промис, который выполнится, когда выполнятся все переданные промисы.

Важные замечения:

:point_up: Порядок результатов гарантирован - в каком порядке переданы промисы в `Promise.all`, в том же порядке придут результаты  
:point_up: Возвращаемый `Promise.all` промис зарезолвится только тогда, когда зарезолвятся все переданные ему в массиве промисы.

```javascript
Promise.all([wait(2000), wait(10000), wait(1000), wait(0)]).then(() =>
  console.log("done")
); // "done" будет выведено не ранее, чем через 10 секунд (время выполнения самого долгого промиса из переданных)
```

Если хотя бы один из переданных промисов выполнится с ошибкой, `Promise.all` вернет промис, реджектнутый со значением первого неуспешного промиса:

```javascript
Promise.all([
  Promise.resolve("RESULT 1"),
  Promise.reject("ERROR 1"),
  Promise.resolve("ERROR 2"),
  Promise.resolve("RESULT 2"),
])
  .then((value) => console.log("it went well:", value)) // не выполнится никогда
  .catch((err) => console.log("it went bad:", err)); // it went bad: ERROR 1
```

#### Promise.any

Метод еще не попал в стандарт, на момент написания находится на [Cтадии 3](https://github.com/tc39/proposal-promise-any) процесса рассмотрения комитетом TC39.
Метод принимает массив промисов и возвращает промис, которые успешно завершится, как только завершится первый из массива промисов. Если ни один из промисов не завершится успешно, возвращенный промис завершится с ошибкой. В отличие от `Promise.all`, `Promise.any` не будет ждать завершения всех переданных промисов, если хотя бы один завершится успешно.

#### Promise.allSettled

Метод принимает массив промисов и возвращает промис, которые завершится после того, как завершатся все переданные ему промисы (как успешно, так и с ошибкой), и вернет массив результатов в качестве значения.

:question: :question: :question: Сформулируйте разницу между `Promise.all` и `Promise.allSettled`

#### Promise.prototype.finally

_Обратите внимание: метод находится на прототипе промиса, а не на классе Promise. Это значит, что он доступен на экземплярах класса Promise, в отличие от предыдущих методов, которые являются статическими и находятся прямо на Promise._

Колбек, переданные этому методу, будет вызван, как только промис будет выполнится, то есть, перейдет в состояние fullfilled или rejected. Этот метод можно использовать, чтобы не дублировать код, которые нужно выполнить при любом исходе - успешном выполнении или ошибке. Без `Promise.prototype.finally` этот код нужно было бы написать дважды - в `.then` и `.catch.`

Важные отличия:
`Promise.prototype.finally` не получит аргументов, так как неизвестно, как именно завершился предыдущий промис - успешно или с ошибкой. Поэтому `Promise.prototype.finally` стоит использовать в тех случаях, когда это не имеет значения.
Если после `.finally` есть обработчики, они получат это значение:

```javascript
Promise.resolve(2)
  .finally(() => {})
  .then(console.log); // 2
Promise.reject(3)
  .finally(() => {})
  .then(console.log); // 3
```

Пример использования - если после загрузки данных нужно убрать индикатор загрузки независимо от того, успешно ли загрузились данные:

```javascript
function hideLoader() {
  // some code to hide loading indicator
}
getDataFromApi().then(hideLoader, hideLoader);
getDataFromApi().then(hideLoader).catch(hideLoader);
getDataFromApi().finally(hideLoader);
```


## Async functions

Получить значение промиса можно только внутри колбека, зарегистрированного на промисе через метод `.then`. Это не очень удобно. Хотелось бы обращаться со значениями промисов так, как с обычными значениями. Помогают в этом асинхронные функции, которые были добавлены в JavaScript после промисов.
Асинхронные функции позволяют писать асинхронный код более удобным и привычным способом - так, как будто он синхронный, но при этом не блокирует поток исполнения. В асинхронных функциях мы можем получить доступ к значению промисов так, как к любому другому значение, не передавая колбеков в `.then`.

Для создания асинхронной функции нужно перед определением функции поставить ключевое слово async. Теперь значением функции будет промис, успешно завершенный со значением,которое возвращает функция, либо с ошибкой, которая была брошена в теле функции.

```javascript
async function foo() {
  return "ok";
}
foo().then(console.log); // 'ok'
```

Если в теле асинхронной функции используются промисы и перед ними стоит ключевое слово `await`, код функции будет дожидаться, пока промис выполнится, и сможет получить доступ к его значению.

```javascript
async function foo() {
  const result = await new Promise((r) => setTimeout(() => r("ok"), 5000));
  console.log(result);
}
foo(); // 'ok' спустя 5 секунд

async function getUserData() {
  const response = await fetch("https://randomuser.me/api/");
  const userData = await response.json();
  const user = userData.results[0];
  console.log(user);
  return user;
}

getUserData(); // random user profile
```

Асинхронные функции интересны тем, что внутри них можно использовать ключевое слово `await`. Если после ключевого слова `await` стоит промис, `await` останавливает исполнение функии до момента, когда промис выполнится, и возвращает значение промиса.

```javascript
function getDataFromApi() {
  // some code which returns a Promise with data
}
async function getData() {
  const result = await getDataFromApi();
  return result;
}

getData().then((result) => console.log(result));
```

Код в предыдущем примере выполнится следующим образом: интерпретатор дойдет до ключевого слова `await`, затем дождется исполнения промиса и только после этого выполнит присваивание значения промиса в переменную `result`. Затем функция вернет значение `result`.

После `await` можно ставить не только промис:

```javascript
async function uselessAsync() {
  const result = await 42;
  return result;
}

uselessAsync().then(console.log); // 42
```

Переданное значение, если оно не промис и не промисо-образный объект, то есть, не реализует метод `then`, оборачивается в `Promise.resolve()`. Таким образом, получится промис, который выполнится сразу.

Значение, возвращаемое `await`, можно не сохранять. Например, так можно реализовать ожидание:

```javascript
// wait ms milliseconds
function wait(ms) {
  return new Promise((r) => setTimeout(r, ms));
}

async function hello() {
  await wait(500);
  return "world";
}
```

### Блокирующий и неблокирующий код

Async/await создает впечатление, что асинхронный код исполняется синхронно. С точки зрения кода внутри асинхронной функции исполнение промиса после `await` блокирующее, но остальной код не блокируется. Другие операции будут выполняться, но оставшийся после `await` код функции заблокирован до исполнения промиса.

### Ошибки в асинхронных функциях

Промис может завершиться не только успешно, но и с ошибкой. Если промис после `await` завершится с ошибкой, `await` бросит эту ошибку и функция вернет реджектнутый промис.

```javascript
async function withError() {
  const result = await Promise.reject("oops");
}

withError().catch(console.log); // oops
```

А вот так промис будет выполнен успешно и ошибка съестся. Только `await` бросает ошибки. Неуспешно выполнившийся промис синхронную ошибку не бросает, они съедаются. Не забывайте ставить `await`!

```javascript
async function withError() {
  Promise.reject("oops");
}

withError().catch(console.log); // в консоль ничего не выведется, но консоль покажет ошибку Uncaught in promise
```

Если в асинхронной функции происходит любая другая ошибка не из промиса, это тоже приведет к возврату зареджекшенного промиса:

```javascript
async function withError() {
  sdsgdfgsg;
}
withError().catch(console.log); // ReferenceError: sdsgdfgsg is not defined
```
Для управления ошибками в асинхронных функциях мы может воспользоваться try/catch:

```javascript
async function getValueSafely() {
  try {
    const value = await getValue();
    return {
      value,
      error: null
    }
  } catch(e) {
    return {
      value: null,
      error: e.message
    }
  }
}
```

Если есть несколько асинхронных операций и нужно обрабатывать ошибки точечно, можно возпользоваться методом `.catch`промисов, хотя в целом смешивание промисов и `await` в теле одной функции не приветствуется.

```javascript
async function getValueSafely() {
  const value1 = await getValue1().catch(value1ErrorHandler);
  const value2 = await getValue2().catch(value2ErrorHandler);
  return { value1, value2 };
}
```

### Последовательное и параллельное выполнение

В асинхронных функциях можно ошибиться, выполняя последовательно действия, которые могут выполняться параллельно, и, соответственно, быстрее.

Сравните эти две функции:

```javascript
async function series() {
  await wait(500); // Ждем 500 мс...
  await wait(500); // ...и еще 500 мы
  return "done!";
}
```

```javascript
async function parallel() {
  const wait1 = wait(500); // Асинхронно запускаем первый таймер на 500 мс...
  const wait2 = wait(500); // ...и второй таймер, который будет отсчитывать время параллельно
  await Promise.all([wait1, wait2]);
  return "done!";
}
```

## Дополнительные материалы

- [Promises/A+](https://promisesaplus.com/) Specification - the source of truth as per Promise behavior
- [We have a problem with promises](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html) - mistakes, which even experienced developers make with Promises. Check yourself!
- [Обещание бургерной вечеринки (Russian translation)](https://medium.com/web-standards/%D0%BE%D0%B1%D0%B5%D1%89%D0%B0%D0%BD%D0%B8%D0%B5-%D0%B1%D1%83%D1%80%D0%B3%D0%B5%D1%80%D0%BD%D0%BE%D0%B9-%D0%B2%D0%B5%D1%87%D0%B5%D1%80%D0%B8%D0%BD%D0%BA%D0%B8-b0ed209809ab) - введение в промисы от Марико Косака с отличными иллюстрациями
- [Async functions - making promises friendly](https://developers.google.com/web/fundamentals/primers/async-functions) - Introduction to async functions by Jake Archibald
- [Promise API Workshop - DIY Promise static methods](https://github.com/kottans/promise-api-workshop) - Workshop held by Yevhen Orlov during Kottans Frontend Course 2018-19. Write your own implementations of Promise static methods to get deeper understanding of them.

If you prefer video format, take a look at these videos by Mattias Petter Johansson aka MPJ on FunFunFunction channel:
- [Promises](https://www.youtube.com/watch?v=2d7s3spWAzo)
- [async/await in JavaScript](https://www.youtube.com/watch?v=568g8hxJJp4)
- [Error handling Promises in JavaScript ](https://www.youtube.com/watch?v=f8IgdnYIwOU)