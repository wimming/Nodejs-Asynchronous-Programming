# Nodejs-Asynchronous-Programming
History of the way to programming asynchronously in Node.js.

Node.js异步编程。

我们知道，Node.js编写的应用是单进程的，它通过异步回调的方式来解决并发的问题，这是Node.js最大的特点之一。
所以最初的时候，大家通过回调函数的方式来编写异步程序（以Node.js的fs模块为例）：
```javascript
var fs = require('fs');

fs.readfile('filepath', function(err, data) {
   // do your work here
});
```
一般约定回调函数的第一个参数是异步操作时抛出的异常，第二个参数是异步操作成功后携带的数据。

但是在服务端编程中，往往要嵌套多个异步回调：
```javascript
var fs = require('fs');

fs.readfile('file1', function(err, data) {
   // do your work 1 here
   asyncFun2('file2', function(err, data) {
     // do your work 2 here
     asyncFun3('file3', function(err, data) {
       // do your work 3 here  
     });
   });
});
```
这样的代码让人看起来非常糟糕！这就是著名的callback hell（回调地狱）！

为了解决这个问题，最初的解决方案是使用Promise：
```javascript
var fs = require('fs');

var promise = new Promise(function(resolve, reject) {
   fs.readfile('filepath', function(err, data) {
      if (!err) resolve(data);
      else reject(err);
   });
});

promise.then(function(data) {
 // success
}, function(err) {
 // failure
});
```
后来ecmascript迎来了ECMAScript 2015版本（也称ES6），有大神利用其中新特性generator函数设计了co/yeild的异步解决方案：
```javascript
const co = require('co');
const Promise = require("bluebird");
const fs = Promise.promisifyAll(require('fs'));

...

co.wrap(function* (req, res, next) { 
  try {
    let data = yeild fs.readFileAsync('filepath');
  } catch (err) {
    console.log(err);
  }
});
```
以上代码中，引入了2个模块：co和bluebird。

bluebird：这个模块提供了统一的操作能“promisify”(promise化)所有按照标准编写的回调函数。所谓promise化，就是将
```javascript
let result;
readFile('filepath', function(err, data) {
  if (!err) result = data;
});
```
转化为
```javascript
let result;
let promise = readFileAsync('filepath');

promise.then(function(data) {
 // success
 result = data;
}, function(err) {
 // failure
 throw err;
});
```
函数名从readFile变成readFileAsync，函数返回值变成一个promise。这个promise执行then以后就能得到读取文件的结果。

co：只要将原本要用promise写法的代码放入一个generator函数并且传递给co模块，里面的代码就能够被正确地异步执行。co模块起到了一个generator函数执行器的作用，以分段执行函数的方式完成异步操作：
```javascript
co.wrap(function* (req, res, next) { 
  try {
    let data = yeild fs.readFileAsync('filepath');
  } catch (e) {
    console.log(err);
  }
});
```
利用generator函数分段执行结果的特性，每次执行这个generator函数都在异步操作时中断，并将一个promise返回给co模块。co模块得到promise以后用then注册其回调函数，当回调函数在事件队列中被唤起时，判断其是否成功，成功则继续执行这个generator函数，并向其中传递异步操作结果（本例中的文件读结果），失败则传递异常。

再后来，ecmascript迎来了ECMAScript 2016版本（也称ES7），这个版本内置了目前最主流的异步解决方案：async/await。
```javascript
async function() {
  try {
    let data = await readFileAsync('filepath');
  }
  catch (err) {
    throw err;
  }
}
```
