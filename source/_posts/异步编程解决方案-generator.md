---
title: 异步编程解决方案 - generator
date: 2017-08-03 11:32:43
tags: Javascript
---


## 异步编程难点

#### 异常处理

在处理异常时经常用try/catch/final语句块进行异常捕获，但是这种异常捕获对异步编程并不是用

    function async(callback) {
        process.nextTick(callback);
    }

    try {
        async(function () {
            console.log(a);
        });
    } catch (err) {
        // TODO
    }


异步代码分为两个过程，提交请求和处理结果，其中代码在异步处理完成之前返回，而异常不一定在这个过程中发生，所以try、catch不会有任何作用，调用async时，callback被暂时挂起，等到代码执行完毕才会执行，try只能捕获当前事件循环的异常，对下一次的事件循环无法处理（nodejs异步时间做了约定，异常一定被当成第一个参数传回，在调用callback时先判断是否有异常发生）

    function async(callback) {
        process.nextTick(function () {
            if (err) {
                return callback(err);
            }
            callback(null);

        });
    }

    try {
        async(function (err) {
            if (!err) {
                console.log(a);
            }
        });
    } catch (err) {
        // TODO
    }

#### 函数嵌套过深

对于Node和agax调用而言，有时会存在多个异步调用嵌套的场景，比如一个文件目录的遍历操作：

    fs.readdir(path.join(__dirname, '..'), function (err, file) {
        files.forEach(function (filename, index) {
            fs.readFile(filename, 'utf8), function (err, file) {
                // TODO
            }
        });
    });

或者一个网页渲染操作：

    $(selector).click(function (e) {
        $ajax({
            data: '',
            success: function (data) {
                template.init(data, function (tpl) {
                    // TODO
                });
            }
        });
    });

上面的代码逻辑上是没有问题的，但是并没有利用好异步I/O带来的优势，这是异步编程的典型问题。

#### 多线程编程

如果是多核CPU，单个Node进程实际没有充分利用多核CPU，浏览器提出了Web workers，通过将javascrit执行与UI渲染分离，可以良好的利用多核CPU。因为前端浏览器对标准的滞后，Web workers并没有广泛应用起来。

#### 异步转同步

习惯同步编程的同学，并不能从容面对异步编程带来的副产品，比如嵌套回调、业务分散。Node 提供了绝大部分异步 API 却很少有同步 API，往往出现同步需求会无所适从，虽然 Node 试图异步转同步但是并没有原生的支持，需要借助库或者编译实现，对于异步编程通过良好的流程控制，还是可以降落几梳理成顺序的形式。

### 异步编程解决方案

#### 事件发布/订阅模式

    // 订阅 
    emiiiter.on('event'， function(message) {
        console.log(message);
    })
    // 发布
    emitter.emit('event', 'i am a message');

事件监听是一种高阶函数的应用，通过事件可以把内部数据传递给外部的调用者，编程者可以不用关心组件内部如何执行，只需关注在需要的事件点上即可。注意： 

* 如果事件的监听器过多可能出现过度占用cup的结果。
* 如果运行期间触发了error事件，解释器会检查是否对error监听了事件，如果有就交给监听器处理，如果没有则将错误抛出。所以应该对error事件做监听。

利用事件可以解决雪崩问题：当大量的访问同时发生时，服务器无法对所有的访问做处理，可以在第一个回调添加状态锁控制服务器的访问数量，同时使用事件（once）把所有请求压入队列中。

#### promise/deferred模式

promise/A 规定了三种状态，未完成态、完成态和失败态，未完成态向其他两种状态转化，不能逆转；

pedding  -> resolved
         -> rejected


    function call(state, fn, err, arg) {
        if  (state === 'pendding') {
            fn(arg);
        } else {
            fn(err);
        }

    }
    new Promise = function (fn) {
        this.state = 'pendding';
        this.fn = function() {};
        return fn(this.resolve, this.reject);
    }
    Promise.prototype.then = function (fn) {
        this.fn = fn;
        return this;
    }
    Promise.prototype.resolve = function (arg) {
        this.state = 'resolved';
        call(this.state, this.fn, null, arg);
        return this;
    }
    Promise.prototype.reject = function () {
        this.state = 'rejected';
        var err = "err opened";
        call(this.state, this.fn, err);
        return this;
    }
    new Promise(function (resolve, reject) {
        setTimeout(function () {
            var value = 'abc';
            resolve(value);
        }, 100);
    }).then(function (result) {
        console.log(result);
    });

#### 流程控制库

##### 中间件

使用connect存储中间件手动调用执行的方式，例如next，通常叫做尾触发，尾触发在jquery中非常常见，比如

    $get('/get').success().error();

这种方式首先注册中间件，每个中间件包括传递请求对象，响应对象和尾触发函数，通过队列行程一个处理流，最简单的中间例如：

    function (req, res, err) {
        // 中间件
    }

connect核心代码：

    function creatServer() {
        function app(req, res) {
            app.handle(req,res);
        }
        app.stack = [];
        for (var i = 0; i < arguments.length; ++i) {
            app.use(arguments[i]);
        }
        return app;
    }

app.use: 

    app.use = function(router, fn) {
        this.stack.push(fn);
        return this;
    }

next:

    function handle = function() {
        // ...
        next();
    }
    function next() {
        // ... next callback ...
        layer = this.stack[index++];
        layer.handle(req, res, next);
    }

##### async
    
异步的串行执行

    async.series([function (callback) {
        callback();
    },function (callback) {
        callback();
    }], function (err, result) {})

等价于：

    function (callback) {
        function (callback) {
            callback();
        }
        callback();
    }

异步的并行执行：

    async.parallel([function (callback) {
        callback();
    }, function (callback) {
        callback();
    }], function (err, results) {

    });

等价于：

    var counter = 2;
    var results = [];
    var done = function (index, value) {
        results[index] = value;
        if (!--conuter) {
            callback(null, results);
        }
    }

    function (callback) {
        // var value = ...
        callback();
        done(0, value);
    }
    function (callback) {
        // var value = ...
        callback();
        done(1, value);
    }

依赖处理

    当前一个异步的结果是后一个异步的输入时，async使用waterfall方式处理

    async.waterfall([function (callback) {
        callback();
    }, function (arg1, callback) {
        callback();
    }, function (arg2, callback) {
        callback();
    }], function (err, results) {

    });

当存在很多依赖关系，有同步有异步时，async使用auto()实现复杂的处理

    async.waterfall({
        fun1:function (callback) {
            callback();
        },
        fun2: ['fun1', function (arg1, callback) {
            callback();
        }, function (arg2, callback) {
            callback();
        }]}, function (err, results) {

        });

##### step

step接受任意数量的任务，所有任务会串行执行：

    step(task1, task2, task3);

step使用next把上一步的结果传递给下一步作为参数
在执行多个异步任务时，调用代码如下：

    step(function () {
        fn1(this.parallel());
        fn2(this.parallel());
    }, function (err, result1, result2) {

    });

##### wind

wind旨在控制异步流程的逻辑控制，其作用类似generator：

    eval(Wind.compile('async', funtion () {
        $await(Wind.Async.sleep(20)); //延迟20ms
        console.log('hello world');
    }));



### generator

#### generaor函数

    function * maker(){
        var index = 0;
        while (index < 10) {
            yield index++;
        }
    }

    var g = maker();
    // 输出结果
    console.log(g.next().value); // 0
    console.log(g.next().value); // 1
    console.log(g.next().value); // 2

#### yeild关键字

yield 关键字用来暂停和恢复一个生成器函数

    [rv] = yield [expression];

    yield [[expression]];

rv 返回传递给生成器的 next() 方法的可选值，以恢复其执行。

#### Regenerator

上面这段代码等价下面代码：

    var _marked = [maker].map(regeneratorRuntime.mark);

    function maker() {
        var index;
        return regeneratorRuntime.wrap(function maker$(_context) {
            while (1) {
                switch (_context.prev = _context.next) {
                    case 0:
                        index = 0;

                    case 1:
                        if (!(index < 10)) {
                            _context.next = 6;
                            break;
                        }

                        _context.next = 4;
                        return index++;

                    case 4:
                        _context.next = 1;
                        break;

                    case 6:
                    case "end":
                        return _context.stop();
                }
            }
        }, _marked[0], this);
    }

    var g = maker();

    console.log(g.next().value); // 0
    console.log(g.next().value); // 1
    console.log(g.next().value); // 2

编译机制造了一个状态机，通过_context.next状态的装换完成代码执行的挂起。
假设状态是0 -> n（n是最后一个状态）
0运行第一个yield之前的所有代码，n运行最后一个yield函数之后的所有代码，generator的next尾调用通过一个while循环实现，如果_context.next到达最后一个case就退出循环，等待下一次next调用

regenerator是用来生成generetor函数并返回一个迭代器供外界调用的高阶函数，功能主要是

* regenerator-transform： 重写generator函数把yield重写成switch case，并且创建_context.next保存上下文环境；
* 包装generator函数被返回一个迭代器对象；

经过wrap返回的迭代器：

    GeneratorFunctionPrototype {
        _invoke: function invoke(method, arg) { … }
        __proto__: GeneratorFunctionPrototype {
            constructor: function GeneratorFunctionPrototype() {},
            next: function (arg) { … },
            throw: function (arg) { … } 
            …
        }
    }

当调用迭代器对象iter.next()方法时，因为有如下代码，所以会执行_invoke方法，而根据前面wrap方法代码可知，最终是调用了迭代器对象的 `makeInvokeMethod (innerFn, self, context);` 方法


makeInvokeMethod方法内容较多，这里选取部分分析。

    function makeInvokeMethod(innerFn, self, context) {
        var state = GenStateSuspendedStart;
        return function invoke(method, arg) {

makeInvokeMethod返回invoke函数，当我们执行.next方法时，实际调用的是invoke方法中的下面语句

    var record = tryCatch(innerFn, self, context);

这里tryCatch方法中fn为经过转换后的example$方法，arg为上下文对象context,因为invoke函数内部对context的引用形成闭包引用，所以context上下文得以在迭代期间一直保持。

    function tryCatch(fn, obj, arg) {
        try {
            return { type: "normal", arg: fn.call(obj, arg) };
        } catch (err) {
            return { type: "throw", arg: err };
        }
    }

tryCatch方法会实际调用 `example$` 方法，进入转换后的switch case,执行代码逻辑。如果得到的结果是一个普通类型的值，我们将它包装成一个可迭代对象格式，并且更新生成器状态至GenStateCompleted或者GenStateSuspendedYield

    var record = tryCatch(innerFn, self, context);
            if (record.type === "normal") {
            // If an exception is thrown from innerFn, we leave state ===
            // GenStateExecuting and loop back for another invocation.
            state = context.done
                ? GenStateCompleted
                : GenStateSuspendedYield;
    
            var info = {
                value: record.arg,
                done: context.done
            };

伪代码：

    function wrap(innerFn, outerFn, self, tryLocsList) {
        var protoGenerator = outerFn && outerFn.prototype instanceof Generator
            ? outerFn
            : Generator;
        var generator = Object.create(protoGenerator.prototype);
        var context = new Context(tryLocsList || []);

        generator._invoke = makeInvokeMethod(innerFn, self, context);

        return generator;
    }

    function makeInvokeMethod(innerFn, self, context) {
        var obj = this;
        return function invoke(method, arg) {
            context.method = method;
            // 把next带入的arg参数赋值给sent
            if (context.method === "next") {
                context.sent = context._sent = context.arg;
            }
            // 实际上调用了mark$，并且带入了context
            var record = {
                arg: innerFn.call(obj, context)
            };
            // 返回一个可以迭代的对象
            return {value: record.arg, done: context.done};
        };
    }

    // 用一个next调用invoke， 如果要进行下一步就传入next
    generator.next = next(arg) {
        generator._invoke('next', arg);
    }

#### cojs处理generator过程

##### thunk函数

能够得到一个函数的函数叫thunk函数， thunk函数是一个偏函数，它只带一个执行参数

    function getThunk(number) {
        return function (fn) {
            setTimeout(() => {
                if (number) {
                    fn(null, number);
                } else {
                    const err = "error open";
                    fn(err);
                }
            }, number)
        }
    }

##### cojs-generator的自动执行器

    import co from 'co';

    co(function * () {
        var a = yield getThunk(100);
        var b = yield getThunk(1000);
        console.log('a:', a);
        console.log('b:', b);
        return [a, b];
    })

    // 输出
    // a 100
    // b 1000

##### cojs代码解析

    function co2Thunk(fn) {
        return (done) => {
            const ctx = this;
            const g = fn.call(ctx);
            function next(err, res) {
                console.log('next1', res);
                let it = g.next(res);
                if (it.done) {
                    done.call(ctx, err, it.value);
                } else {
                    it.value(next);
                }
            }
            next();
        }
    }

    co2Thunk(function * () {
        var a = yield getThunk(10000);
        var b = yield getThunk(1000);
        // console.log('a:', a);
        // console.log('b:', b);
        return [a, b];
    })(function (err, args) {
        console.log("callback thunk co : ==========");
        // console.log(err, args);
    });

co2Thunk的代码等价于：

    function co2Thunk(fn) {
        return (done) => {
            const ctx = this;
            const g = fn.call(ctx);
            let it0 = g.next();
            it0.value((err, res) => {
                const it1 = g.next(res); // 第一次迭代返回的是getThunk(10000);
                it0.value((err, res) => {
                    const it1 = g.next(res); // 第二次迭代返回的是getThunk(1000);
                    it1.value((err, res) => {
                        const it2 = g.next(data);
                        // ...
                    });
                });
            });
        }
    }

    // it.value 等价于：
    function (fn) {
        setTimeout(() => {
            if (number) {
                fn(null, number);
            } else {
                const err = "error open";
                fn(err);
            }
        }, number)
    }


##### promise版

    function co2Promise(fn) {
        return new Promise((resolve, reject) => {
            const ctx = this;
            const g = fn.call(ctx);
            function next(err, res) {
                let it = g.next(res);
                if (it.done) {
                    resolve(it.value);
                } else {
                    it.value(next);
                }
            }
            next();
        });
    }

    co2Promise(function * () {
        var a = yield getThunk(100);
        var b = yield getThunk(1000);
        console.log('a:', a);
        console.log('b:', b);
        return [a, b];
    }).then(function (args) {
        console.log("callback promise co : ==========");
        console.log(args);
    });

##### thunk升级版

    function co2Thunk(fn) {
        return (done) => {
            const ctx = this;
            const g = fn.call(ctx);
            function next(err, res) {
                let it = g.next(res);
                if (it.done) {
                    done.call(ctx, err, it.value);
                } else {
                    // 增加对其他类型的处理
                    const value = toThunk.call(ctx, it.value);
                    // 对于promise 此处应该是 value.then(next)
                    value(next);
                }
            }
            next();
        }
    }

    co2Thunk(function * () {
        var a = getThunk(100);
        var b = getThunk(1000);
        // console.log('a:', a); console.log('b:', b);
        return yield [a, b];
    })(function (err, args) {
        console.log("callback thunk co : ==========");
        console.log(err, args);
    });

    function toThunk(obj) {
        if (isObject(obj) || isArray(obj)) {
            return objectToThunk(obj);
        }
        if (isPromise(obj)) {
            return promiseToThunk.call(ctx, obj);
        }
        return obj;
    }

    function objectToThunk(obj) {
        return function (done) {
            let keys = Object.keys(obj);
            let length = keys.length;
            let results = new obj.constructor();
            for(let key in keys) {
                const fn = toThunk(obj[key]);
                fn((err, res) => {
                    results[key] = res;
                    --length || done(null, results);
                }, key);
            }
        }
    }

    function promiseToThunk(promise){
        return function(done){
            promise.then(function(err,res){
                done(err,res);
            },done)
        }
    }

    function isObject(obj) {
        return obj && Object == obj.constructor;
    }
    function isArray(obj) {
        return Array.isArray(obj);
    }

    function isPromise(obj) {
    return obj && 'function' == typeof obj.then;
    }

#### async/await

    async function fn(args){
    // ...
    }

等同于

    function fn(args){ 
        return co2Thunk(function*() {
            // ...
        }); 
    }
