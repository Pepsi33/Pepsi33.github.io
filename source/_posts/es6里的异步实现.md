---
title: es6里的异步实现
date: 2019-03-14 12:53:36
tags: ["javascript"]
---

## promise

#### Promise 是什么？
[阮一峰: Promise 对象](http://es6.ruanyifeng.com/#docs/promise) 
> Promise 是异步编程的一种解决方案，比传统的解决方案——回调函数和事件——更合理和更强大。它由社区最早提出和实现，ES6 将其写进了语言标准，统一了用法，原生提供了Promise对象。


#### Promise 的特点？
> 首先我们知道 promise 对象有三个状态，分别被为pending（等待）、fulfilled（完成）、rejected（失败）  
> 状态只可以从pending -> fulfilled 或者 pending -> rejected

- 状态只能由它内部改变
- 状态不可逆

<!--more-->

1. 内部最好写异步操作，毕竟它是异步编程的一种解决方案 
> 可以很方便的解决回调地狱这种问题

```javascript
$.ajax('url', params, funtion (res) {

  $.ajax('url', res , funtion (res1) {
  
    $.ajax('url', res1, funtion (res2) {
            console.log(res2)
        })
        
    })  
    
})

fucntion ajax (url, params) {

    return new Promise(function (resolve) {
    
         // 异步操作 才有意义
        $.ajax(url, params, funtion (res) {
            resolve(res);
        })
        
    })
    
}

    ajax('u1', 'p1').then((res) => {
         return ajax('u2', res); // 将 ajax 返回的 promise中的值传递给 then返回的promise。 
    }).then((res1) => {
        return ajax('u3', res1);
    }).then((res2) => {
        console.log(2)
    })
    
    
```

2. Promise 构造函数的基本实现
- 再给实例化的promise中传递的回调中无法用return 代替 resolve 
```javascript
// promise 其实就是一个状态机 通过不同的状态执行不同的操作
      const pending = 'pending';
      const resolved = 'resolved';
      const rejected = 'rejected';

      // MyPromise 模拟 promise构造函数 简易的实现
      function MyPromise (fn) { // ‘func’ 方便this指向清晰
          const _this = this; // 缓存当前 promise 实例对象
          _this.state = pending; // 初始状态
          _this.value = undefined; // promise 中存的值

          /*
          * 当promise在pending的时候
          * 会将我们写在.then(() => {}, () => {})中的回调传给下边的俩个数组
          * */
          _this.resovleCallbacks = [];
          _this.rejectCallbacks = [];

          // 为什么resolve 加setTimeout?
          // 实践中要确保 onFulfilled 和 onRejected 方法异步执行，且应该在 then 
          // 方法被调用的那一轮事件循环之后的新执行栈中执行。

          // 我在平常的时候 以为在回调中 resolve了 状态就会立即改变 然而 并不是
          _this.resolve = function (val) { // 当我们在回调中 调用它的时候就会触发 异步代码
              if (val instanceof MyPromise) { // 检测 resolve(val) 的值是不是 promise对象
                return val.then(_this.resolve, _this.reject); // 通过递归 将 val(promise) 的值 传给最外层的promise 
              }

              if (_this.state !== resolved) { // 第一次调用 状态是pending 所以为 true
                  setTimeout(function () { // 这里异步 then 中回调就会传到 callbacks = []
                      // 一个 promise 状态只可以改变一次
                    _this.state = resolved;
                    _this.value = val;
                    _this.resovleCallbacks.forEach(fn => fn());
                  })
              }
          };

          this.reject = function (err) { // 可以传递promise 但会报错
            if (_this.state === rejected) {
                setTimeout(function () {
                    _this.state = rejected;
                    _this.value = err;
                    _this.rejectCallbacks.forEach(fn => fn());
                })
            }
          };

          try {
            fn(_this.resolve, _this.reject); // 调用传入的回调函数
          } catch (e) {
            _this.reject(e); // 捕获 回调内部 报错
          }
      }
      
      
```


3. then 方法
> Promise.prototype 上的一个属性，所以每一个promise实例都可以用它  
> then 方法必须返回一个 promise 对象   
> then 方法可以被同一个 promise 调用多次

```javascript
new Promise.then() 会生成一个新的promise实例

    
    MyPromise.prototype.then = function (onResolved, onRejected) {
          const self = this;
          let newPromise = null;
          
          // 如果类型不是函数需要忽略，同时也实现了透传
          // Promise.resolve(4).then().then((value) => console.log(value))
          
         // onFulfilled 和 onRejected 必须被作为函数调用
          onResolved = typeof onResolved === 'function' ? onResolved : v => v;
          onRejected = typeof onRejected === 'function' ? onRejected : r => {throw r};
          
          if (self.state === resolved) {
              newPromise = new MyPromise(function (resolve, reject) {
                  setTimeout(function () { // 保证 onResolved 和 onRejected 异步 // 该回调会在 当前promise实例的状态改变之后调用
                      try { // 防止报错
                          const x = onResolved(self.value);
                          resolvePromise(newPromise, x, resolve, reject); //  判断 x 的类型执行对应操作
                      } catch (err) {
                          reject(err);
                      }
                  });

              });

              return newPromise;
          }
          if (self.state === pending) {
              newPromise = new MyPromise(function (resolve, reject) {
                  self.resovleCallbacks.push(() => { // 该回调会在 当前promise实例的状态改变之前push进去
                      try {
                          const x = onResolved(self.value);
                          resolvePromise(newPromise, x, resolve, reject);
                      } catch (e) {
                          reject(e);
                      }
                  });

                  self.rejectCallbacks.push(() => {
                      try {
                          const x = onResolved(self.value);
                          resolvePromise(newPromise, x, resolve, reject);
                      } catch (e) {
                          reject(e);
                      }
                  });
              });

              return newPromise;
          }
      };
      
      function resolvePromise (promise2, x, resolve, reject) { // 对内部生成的 newPromise 进行resolve 给下一个then 传值
          if (promise2 === x) {
                throw Error('防止死循环');
          }

          if (x instanceof MyPromise) { // 把 x 的值传给当前的 newPromise
              if (x === pending) {
                  x.then(function (val) {
                      resolvePromise(promise2, val, resolve, reject); // 检测val的类型，如果合适就传给promise2
                  }, reject)
              } else {
                  x.then(resolve, reject); // 直接将x的值 传给promise2
              }
          }

          let called = false; // 防止thenable内的方法多次调用 例如 resolve 多次调用
          //
          if (x !== null && (typeof x === 'object' || typeof x === 'function')) { // 判断当前 x thenable （函数或对象内具有then属性的方法）
            try {
               let then = x.then; // 获取 then 的值
               if (typeof then === 'function') {
                   then.call(
                       x, // 绑定当前 x 对象
                       y => { // 将 then 中resolve的 值传递给 promise2 
                           if (called) return; 
                           called = true;
                           resolvePromise(promise2, y, resolve, reject); // 验证 y 的类型
                       },
                        reason => {
                            if (called) return;
                            called = true;
                            reject(reason);
                        }
                       )
               } else {
                   resolve(x);
               }
            } catch (e) {
                if (called) return;
                called = true;
                reject(e);
            }
          } else { // 普通值
              resolve(x);
          }

      }
    
```
4. promise 错误处理

promise 内部即使报错，它也不会强制导致代码停止运行


```javascript
// promise内部代码都是在try...catch内部运行的
try {
    new Promise((resolve, reject) => {
        // throw Error('错误');
        reject('错误');
    })
} catch (e) {
    console.log('并不会捕获到');
}


// catch() === then(null, () => {}) catch其实就是then的另一种实现方式 

let rejected  = Promise.reject('报错');

rejected.catch(val => {
    console.log(val);
})

```



5. Promise.race([]) 将可以迭代对象（数组）中最先被改变的promise的值 传给return promies 

```javascript
function timerPromise(delay) {
          return new Promise(function (resolve, reject) {
              setTimeout(function () {
                  if (delay === 40) {
                      reject(delay);
                  } else {
                      resolve(delay);
                  }
              }, delay);
          });
      }

      Promise.race([ // 如果数组中的 promise 的状态改变了当前的promise状态也会随之改变
          timerPromise(40),
          timerPromise(20),
          timerPromise(30)
      ]).then(values => {
          console.log(values);
      });
      
MyPromise.race = function (promises) { // 内部实现
    return new Promise((resolve, reject) => {
        promises.forEach(p => {  
            p.then(resolve, reject);  // 如果 p 的状态不是pending 了 resolve这个回调就会调用
        })                            // resolve 中具有 return promise对象 的this指针     
                                      // p 通过then 可以把自己的值传递给 return promise
                                      // 状态改变逻辑不能重复调用
    });
}      


```

6. Promise.all([]); 

> Promise.all可以将多个Promise实例包装成一个新的Promise实例。  
同时，成功和失败的返回值是不同的，成功的时候返回的是一个结果数组，  
而失败的时候则返回最先被reject失败状态的值。

- 在处理多个异步处理时非常有用

```javascript
Promise.all = function (promises) {
    return new Promise((resolve, reject) => { // 返回一个promise实例
        const length = promises.length;
        let result = gen(length, resolve)
        promises.forEach((p, i) => {
            // 将所有的 promise 遍利出来
            p.then(val => {
                result(i, val)
            }, reject)
        }) // 如果有一个promise 被拒绝 就改变return promise 的状态
    });
    
    function gen (length, resolve) {
        let count = 0; // 记录循环次数
        let values = [];
        return function () {
            values[i] = value;
            if (++count === length) { // 遍历完成时改变 return promise 的状态
                resolve(values);
            }
        }
    }
}
```

---


## Iterator (迭代器、遍历器)

> 为各种不同的数据结构提供统一的访问机制。任何数据结构只要部署 Iterator 接口，就可以完成遍历操作


#### 迭代器的组成接口

```TypeScript
interface IteratorResult { // 迭代器结果
    done: boolean,
    value: any
}
interface Iterator { // 迭代器
    next(): IteratorResult 
}
interface Iterable { // 可迭代对象
    [Symbol.iterator]: Iterator  // 通过 Symbol.iterator(迭代器生成函数) 生成迭代器
}
```

什么是迭代器？

> 迭代器是一个对象，它一定会有一个 ==next()== 方法，每次调用 ==next()== 方法，
就会返回一个迭代器结果。

什么是迭代器结果？

> 迭代器结也是一个对象，这个对象有两个属性：done和value，  
其中done是一个布尔值，false表示迭代器迭代的序列没有结束；  
true表示迭代器迭代的序列结束了。而value就是迭代器每次迭代真正返回的值。 
（它们反应了当前元素和当前状态）

什么是可迭代对象？

> 具有 ==\[Symbol.iterator\]()== 这个接口的数据结构就叫可迭代对象

> es6中原生具备这个接口的数据结构如下：   
Array、Map、Set、String、函数内部的Arguments对象、NodeList、TypedArray

> 对象是非线性的数据结构所以没必要部署迭代器接口，因为迭代器是一种线性（有顺序的）处理，  
而且对象实际上被当作 Map 结构使用，es6原生提供了，
如果非要可迭代可以自己添加 ==\[Symbol.iterator\]()== 这个属性。

#### 用于操作可迭代对象的语法：

- for ... of 
- [...iterable] (扩展运算符)
- Array.from(iterable)

```javascript
const arr = ['a', 'b', 'c', 'd'];

    const sequence = {
        [Symbol.iterator]() {
            let i = 0;
            return {
                next() {
                    const value = arr[i];
                    i++;
                    const done = i > arr.length;
                    return {
                        value,
                        done
                    }
                }
            }
        }
    };
    
    for (const val of sequence) {
        console.log(val) // 'a' 'b' 'c' 'd'
    }
    
    console.log([...sequence]) // ["a", "b", "c", "d"]
    console.log(Array.from(sequence)) // ["a", "b", "c", "d"]
    
```

#### 迭代器中的状态

> 如果迭代器中没有设置终止状态，可以通过for ... of 来手动终止

```javascript
const random = { // 如果不改变迭代器结果的状态
      [Symbol.iterator]: () => ({
        next: () => ({ value: Math.random() }) 
      })
    }
    
    [...random] // 就会死循环
    
    for (const val of random) { // for ... of可以通过 break 来退出循环
        if (val > 0.6) {
            break;
        }
        console.log(val);
    }
```



## Generator

> generator函数是 ES6 提供的一种异步编程解决方案，语法行为与传统函数完全不同。

```typescript
interface Generator extends Iterator {
    next(value?: any): IteratorResult; // 迭代器具有的特征
    [Symbol.iterator](): Iterator; // 可迭代对象具有的特征
    throw(exception: any);
}

```

生成器是什么？

> 生成器函数返回的是一个生成器，它内部既有迭代器具有的特征，也有可迭代对象具有的特征，所以说，它既是一个迭代器，又是一个可迭代对象。  

> 因为生成器还提供了一个 yield 关键字，它返回的序列值会自动包装在一个IteratorResult（迭代器结果）对象中,  
所以生成器又是迭代器的“加强版”。

验证一下
```javascript
function *gen() {
  yield 'a'
  yield 'b'
  return 'c'
}

const g = gen();

// 生成器具有可迭代对象的特征
typeof g[Symbol.iterator] === 'function' // g 是个可迭代对象

// 说明它可以被 操作可迭代对象的语法 来操纵
for (const val of g) {
    console.log(val) // a, b
}
// for ... of 在 done 为 true 时就停止运行，所以 return 的返回值并不会被遍历出来。


typeof g.next === 'function'  // g是迭代器

g.next() // {value: 'a', done: false} // 迭代器是可以直接调用 next 方法的

```
### yield 

> ==yield== 关键字 它可以使生成器函数 ++执行暂停++，yield关键字后面的表达式的值 返回给生成器的调用者。

> yield关键字 返回一个IteratorResult对象，它有两个属性，value和done。value属性是对yield表达式求值的结果，而done是迭代完成的状态，一直都是false。（迭代完成或者return会改变done的状态）

#### 生成器中的yield 与 return
> 它们有些作用是相似的，都可以返回函数中的值，都可以暂停函数的执行。但是yield只是将函数暂时的挂起，而return则表示函数运行结束；

```javascript
function* gen (x) {
    const y = x * (yield);
    return y;
}

const it = gen(3);

//启动生成器 
it.next(); // {value: undefined, done: false} 

it.next(4); // {value: 12, done: true}
```
> 生成器的独到之处，就是在于它的 ==yiled== 关键字，它的有俩个神奇之处：
1. 它是生成器函数暂停和恢复执行的分界点；
2. 它是向外和向内传值（包括错误/异常）的媒介。

#### 生成器的单向执行且不可逆

```javascript
function *gen() {
  yield 'a'
  yield 'b'
  return 'c'
}

const g = gen();
[...g] // ['a', 'b']
[...g] // []

```

####  导致生成器暂停的情况还有两种
1. 到达生成器底部也会停止，生成器执行完成
2. 生成器内部有throw语句它也会导致生成器完全停止执行

#### 错误处理

> 生成器的错误可以‘由内而外’也可以‘由外而内再由外’，具体表现如下：

```javascript
// 由内而外
function* testErr () {
    const x = yiled 'hi';
    yield x.toLowerCase(); // 内部报错,返回异常
}

const it = main();
it.next().value // 'hi'

try{
    it.next(3); // 传入导致数值类型错误，外部接收异常
} catch(e) { 
    console.error(e) // TypeError
}

// 由外而内再由外
function* main() {
  var x = yield 'hi';
  console.log('never gets here'); 
}

const it = main();
it.next().value; // 'hi'
try {
// .throw 会给 yield 传递异常信息 和 .next() 传值相似
  it.throw('报错'); //导致生成器终止运行
                    // 生成器接受到异常，又回抛出来
} catch (err) {
  console.error(err); // 报错
}

```
### 异步迭代生成器

> 生成器的异步在于 ==yield== ,因为它不是++必须++同步等待 .next(val) 来给它传值的，而是可以在异步操作中来调用 .next(val) 把值传给它，所以 yield 是可以等待一个异步操作结果的

> 利用生成器，在生成器内部以同步的方式来写异步代码

```javascript
// 1 先封一个基于promise的http请求

funtion get(url) {
    return new Promise((resolve) => {
        $.post(url, function (data) {
            if (!data.isErr) {
                resolve(data);
            } else {
                reject(data);
            }
            
        })
    })
}


// 2. 在生成器中请求数据
function* foo () {
    const data =  yield get('url');
    console.log(data); 
}

const f = foo();
const p = f.next().value; // 获取到get中的promise

p.then(val => {
    f.next(val) // 获取到promise的值，再通过next返给yield，从而代码恢复执行，输出data
}, err => {
    f.throw(err)
})

```

#### generator中 yiled 与 next
```javascript
function* test() {
    let a = 1 + 2;
    let b = yield 2;
    const c = yield b;
    yield c;
    return;
}
let b = test();
console.log(b.next());
console.log(b.next(1));
console.log(b.next(2));
console.log(b.next());

// es5
var _marked = /*#__PURE__*/regeneratorRuntime.mark(test);

function test() {
    var a, b, c;
    return regeneratorRuntime.wrap(function test$(_context) {
        // 可以发现通过 yield 将代码分割成几块
        // 每次执行 next 函数就执行一块代码
        // 并且表明下次需要执行哪块代码
        while (1) {
            switch (_context.prev = _context.next) {
                case 0:
                    a = 1 + 2;
                    _context.next = 3;
                    return 2;

                case 3:
                    b = _context.sent;
                    _context.next = 6;
                    return b;

                case 6:
                    c = _context.sent;
                    _context.next = 9;
                    return c;

                case 9:
                case "end":
                    return _context.stop();
            }
        }
    }, _marked, this);
}
var b = test();
console.log(b.next());
console.log(b.next(1));
console.log(b.next(2));
console.log(b.next());
```

## async/await 
> async 函数是 Generator 函数的语法糖。使用 关键字 async 来表示，在函数内部使用 await 来表示异步。

### async/await 可以让异步代码以同步的方式来编写 
```javascript
get(url, () => {
    get(url2, () => {
    
    })
})


const getData = async () => { // 改善了嵌套的问题
    const g1 = await get(url1);  
    const g2 = await get(url2);
}

// async 中的await 等的就是Promise 
// 它可以将promise中resolve或reject（所以也可以在async用try...catch来捕获primise拒绝的信息）的值返回出来

// 它也可以等待原始类型的值（Number，string，boolean，但这时等同于同步操作）
// 但是这并没有多大意义

```
### 谨慎使用async/await

> 如果将并发的请求中写在同一个 async 函数中会造成性能损失

```javascript
get(url1, () => {
    get(url2, () => {
    
    })
})
get(url3, () => {
    get(url4, () => {
    
    })
})

const getData = async () => { // 因为await会等待promise状态改变才会执行它下面的代码
    const g1 = await get(url1);
    const g2 = await get(url2);
    const g3 = await get(url3);
    const g4 = await get(url4);
    
}
// 所以这样的写法在运行的过程中，其实是将 g3 并发的请求也嵌套了进去
这就会加长请求的时间，影响性能
get(url1, () => {
    get(url2, () => {
        get(url3, () => {
            get(url4, () => {
            
            })
})
    })
})

```

### Async 中 return 和 return await ;

> 函数前面加了async 该函数会默认返回一个promise

```javascript
function test () {
    reutrn Promise.resolve(); // promise 的值默认为 undefined
}

async function test () {
    return 'valeu'; // return返回的值 实际上是被包在了 promise 中
}
```
> Async 中的 await 后边的表达式是一个promise 才有有意义，promise 中值可以通过 await 来返回
> Async 中 return 和 return await 只在 try...catch 中才有区别

```javascript
async function test () {
    throw new Error('错错错'); // reject(new Error('错错错'));
}
// 类似与
const test = new Promise(() => {
    throw new Error('错错错');
});

```

- Async 中的 return；
```javascript
(async function () {
    try {
        return test(); // 重点关注
    }catch(err) {
        console.log('这个并不会捕获到错误')
    }
})().then(() => {
    console.log('resolved');
}, (err) => {
    console.log('rejected'); 
});

// 输出：rejected
```

- Async 中的 return await；
```javascript
(async function () {
    try {
        return await test(); // 重点关注
    }catch(err) {
        console.log('会捕获到这个错误');
    }
})().then((res) => {
    console.log('resolved');
}, () => {
    console.log('rejected');
});

// 输出：
// 会捕获到这个错误
// resolved
```

### Async 是 Generator的语法糖

> async 内部的语法逻辑 可以通过Generator加一个运行器来实现

#### 运行器

```javascript
function run(generator) {
    // 返回一个promise
  return new Promise((resolve, reject) => {
  
    const it = generator() // 返回生成器
    
    step(() => it.next())
    
    function step(nextFn) {
      const result = runNext(nextFn) // 得到IteratorResult
      if (result.done) { // done 为 true说明 return语句 运行结束 
        resolve(result.value) // 将请求返回的值转给当前的promise
        return
      }
      Promise
        .resolve(result.value) // 获取生成器中的promise
        .then(                      
          value => step(() => it.next(value)), // 将promise中的值传给生成器中yield
          err => step(() => it.throw(err))
        )
    }
    
    function runNext(nextFn) { // 错误处理
      try {
        return nextFn()
      } catch (err) {
        reject(err)
      }
    }
    
  })
  
}

// 通过生成器运行程序控制异步代码

function example() {
    return run(function* () {
    
        const r1 = yield new Promise(resolve => {
            setTimeout(resolve, 500, 'r1value');
        });
        
         const r2 = yield new Promise(resolve => {
            setTimeout(resolve, 200, 'r2value');
        });
        
        return [r1, r2];
    })
}

example.then(val => console.log(val)); // ['r1value', 'r2value']

// async/await 来控制异步代码

async function example() {
    // 内置了运行器函数
    const r1 = yield new Promise(resolve => {
            setTimeout(resolve, 500, 'r1value');
        });
        
    const r2 = yield new Promise(resolve => {
            setTimeout(resolve, 200, 'r2value');
        });
        
    return [r1, r2];
}

example.then(val => console.log(val)); // ['r1value', 'r2value']

```

>  async/await 其实是基于promise、iterator、generator的‘语法糖’




