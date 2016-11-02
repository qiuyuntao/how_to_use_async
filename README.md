# 使用Async函数处理异步

### 回调函数

基本上，我们处理异步的时候，都会使用callback来进行处理。

例如，发起一个ajax请求的时候

```js
$.ajax({
  url,
  success(res) {
      console.log(res);
  }
})
```

当然，callback其实是很好理解的，在收到请求之后，我们执行这个函数。但是，问题在与，如果我们要发起多次的异步请求，那就麻烦了

```js
$.ajax({
  url,
  success(res) {
    console.log(res);
    $.ajax({
      url,
      success(res) {
        console.log(res);
      }
    })
  }
})
```

像这个，有两个callback的时候，代码就会变的非常的恶心。假如我们变成3个、4个。。。 那简直就无法进行维护了。看着头都会很大，造成回调地狱


### Promise函数

Promise函数可以解决函数嵌套的问题。它可以将callback的嵌套改成链式的调用。

```js
function getData() {
  return new Promise((resolve, reject) => {
    $.ajax({
      url: './package.json',
      success(res) {
        resolve(res);
      },
      error(err) {
        reject(err);
      }
    });
  });
}

getData()
  .then((val) => {
    console.log(val);
  })
  .then(() => {
    return getData();
  })
  .then((val) => {
    console.log(123, val);
  })
  .catch((err) => {
    console.error(err);
  });
```

使用Promise来处理异步，你会发现有一个问题。整个代码看上去全是then，而且每个then在处理什么问题，你需要痕仔细的去查看。代码的可维护性也不是特别的高。

### Generator函数

#### 简介

```js
function *gen(x) {
  let y = yield x + 2;
  return y;
}

let g = gen(1);
g.next(); // { value: 3, done: false}
g.next(); // { value: undefined, done: true}
```

* 遇到yield，函数就会暂停，执行next之后，函数继续执行

#### 利用Promise对象自动执行

* 首先把异步函数包装成promise函数

```js
function getData(url) {
  return new Promise((resolve, reject) => {
    $.ajax({
      url,
      success(res) {
        resolve(res);
      }
    })
  });
}

var gen = function* (){
  var f1 = yield getData('./package.json');
  var f2 = yield getData('./package.json');
  console.log(f1);
  console.log(f2);
};

var g = gen();

g.next().value.then(function(data){
  g.next(data).value.then(function(data){
    g.next(data);
  });
});
```

* 我们可以看到，如果我们用promise来进行操作的话，会需要人肉不断的嵌套回调。我们可以编写一个自动执行Generator的执行器。

```js
function run(gen) {
  let g = gen();

  function next(data) {
    let res = g.next(data); // 直接自动执行
    if (!res.done) { // 如果没有执行完，那就自动执行
      res.value.then((val) => { // 执行完的promise对象，继续执行
        next(val);
      });
    }
  }

  next();
}

run(gen);
```

* 只要Generator函数还没执行到最后一步，next函数就调用自身，以此实现自动执行。
* 传递多个promise，可以让并发同时执行

```js
function run(gen) {
  let g = gen();

  function onFulfilled(data) {
    let res = g.next(data); // 直接自动执行
    next(res);
  }

  function next(res) {
    if (res.done) return;
    let promise = res.value;

    if (!isPromise(res.value)) promise = Promise.all(res.value);
    promise.then((val) => {
      onFulfilled(val);
    })
  }

  function isPromise(obj) {
    return 'function' == typeof obj.then;
  }

  onFulfilled();
}

run(function *() {
  let a = yield [
    Promise.resolve(1),
    Promise.resolve(2),
    Promise.resolve(3)
  ]
  console.log(a);

  let b = yield Promise.resolve(12312);
  console.log(b);
});

```


### Async函数

#### 简介

 * async函数返回的是一个promise对象，函数里的return将会作为then方法调用的回调

```js
async function f() {
  return 'hello world';
}

f().then(v => console.log(v))
// "hello world"
```

* 只有当async函数内部的所有内容都执行完了之后，才会执行then方法中的回调
* await后面一个是一个promise对象。如果不是的话，会被转成一个立即执行的resolve的promise对象

```js
async function f() {
  return await 123;
}

f().then(v => console.log(v)); //123
```

* 如果promise被设置为reject的时候，async会catch住错误，可以用以下两种方式进行处理
    * 一种方法是在async中加入try..catch区块

    ```js
    // method 1
    async function f() {
      try {
        await Promise.reject('get error');
      } catch (err) {
        console.log('error', err);
      }
    }

    f();
    ```

    * 第二种方法是在async执行之后的promise对象会被设置为reject状态，在catch中进行回调

    ```js
    // method 2
    async function f() {
      await Promise.reject('get error');
    }

    f()
      .catch(err, () => {
        console.log('error', err);
      });
    ```

#### 使用方式

* 函数表达式

```js
async function f() {
  await 123;
}

const f = async function() {
  await 123;
}
```

* 对象的方法

```js
let obj = {
  async f() {
    await 123;
  }
}
```

* Class 方法

```js
class C {
  constructor（() {
    ...
  }

  async f() {
    await 123;
  }
}

let c = new C();
c.f();
```

* 箭头函数

```js
const foo = async () => {
  await 123;
}

[1, 2, 3].map(async (d) => {
  let a = await d;
})
```

#### 执行多个异步操作

* 队列执行多个异步操作，第一个resolve之后，才会执行第二个异步操作

    ```js
    async function f() {
      let a = await getData('./a.json');
      let b = await getData('./a.json');
    }
    ```

* 同时执行多个异步操作

```js
// method 1
async function f() {
  let [a, b] = await Promise.all([getData('./a.json'), getData('./b.json')]);
  console.log('两个都返回后，执行到这里', a, b);
}

// method 2
async function f() {
  let a = getData('./a.json');
  let b = getData('./b.json')

  let dataA = await a;
  let dataB = await b;
}
```




