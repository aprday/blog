---
title: 深入解析Vue源码
date: 2017-08-03 11:37:25
tags: Javascript
---

### Vue简介
响应式编程 路由
组件化 稳定性
模块化 动画

数据绑定

    /**
    *假设有这么两种东西
    **/
    //数据
    var object = {
      message: 'Hello World!'
    }
    //DOM
    <div id="example">
      {{ message }}
    </div>

    /**
    *我们可以这么写
    **/


    new Vue({
      el: '#example',
      data: object
    })

    /**
    * 如果有个数据
    **/

    var object1 = {
      message: 'Hello World!'
    }

    var object2 = {
      message: 'Hello World!'
    }

    //DOM
    <div id="example1">
      {{ message }}
    </div>

    <div id="example2">
      {{ message }}
    </div>

    /**
    *我们还可以这么写
    **/

    var vm1 = new Vue({el: '#example1',data: object})
    //改变vm1的数据DOM随之改变
    vm2.message = 'oliver'

    var vm2 = new Vue({el: '#example2',data: object})

    vm2.message = 'lisa'

组件化
    var Example = Vue.extend({
    template: '<div>{{ message }}</div>',
    data: function () {
    return {
      message: 'Hello Vue.js!'
    }
    }
    })

    // 将该组件注册为 <example> 标签
    Vue.component('example', Example)

    Vue 在组件化上和 React 类似：一切都是组件。
    组件使用上也和React一致:

    <example></example>

组件之间数据传递:
    1.用 props 来定义如何接收外部数据;
    Vue.component('child', {
      // 声明 props
      props: ['msg'],
      // prop 可以用在模板内
      // 可以用 `this.msg` 设置
      template: '<span>{{ msg }}</span>'
    })
    <child msg="hello!"></child>

    2.用自定义事件来向外传递消息;
    使用 $on() 监听事件；
    使用 $emit() 在它上面触发事件；
    使用 $dispatch() 派发事件，事件沿着父链冒泡；
    使用 $broadcast() 广播事件，事件向下传导给所有的后代。

    3.用 <slot> API 来将外部动态传入的内容（其他组件或是 HTML）和自身模板进行组合;

模块化

    Webpack 或者 Browserify，然后再加上 ES2015配合 vue-loader 或是 vueify，就可以把Vue的每一个组件变成
    Web Components

    <!-- MyComponent.vue -->

    <!-- css -->
    <style>
    .message {
      color: red;
    }
    </style>

    <!-- template -->
    <template>
      <div class="message">{{ message }}</div>
    </template>

    <!-- js -->
    <script>
    export default {
      props: ['message'],
      created() {
        console.log('MyComponent created!')
      }
    }
    </script>

路由

    使用Vue重构的Angular项目

    www.songxuemeng.com/diary

    个人感觉vue-router烦的问题是组件之间的数据交互,rootRouter的数据很难向其他组件传递.

    /**
    *解决方法
    **/
    var app = Vue.extend({
      data:function(){
          return {
              data:'',
          };
      },
    });
    router.map({
          '/': {
              component:  Vue.extend({
                                mixins: [calendar.mixin],
                                data:function(){
                                    return {
                                        data:data
                                    }
                                }
                          })
          },
      })
    router.start(app, '#app');

###Vue源码分析

http://img2.tbcdn.cn/L1/461/1/8142ef3fc2055839f1a93a933d80e17694b4f76b

Vue.js是一个典型的MVVM的程序结构，程序大体可以分为：
全局设计：包括全局接口、默认选项等；
vm实例设计：包括接口(vm原形)、实例初始化过程(vm构造函数)

下面是构造函数最核心的工作内容。

http://img3.tbcdn.cn/L1/461/1/00049a09def4aff8d80f3bb7229e3f6d395426fb

整个实例初始化的过程中，重中之重就是把数据 (Model) 和视图 (View) 建立起关联关系。Vue.js 和诸多 MVVM 的思路是类似的，主要做了三件事：

通过 observer 对 data 进行了监听，并且提供订阅某个数据项的变化的能力
把 template 解析成一段 document fragment，然后解析其中的 directive，得到每一个 directive 所依赖的数据项及其更新方法。比如 v-text="message" 被解析之后；
所依赖的数据项 this.$data.message，以及
相应的视图更新方法 node.textContent = this.$data.message
通过 watcher 把上述两部分结合起来，即把 directive 中的数据依赖订阅在对应数据的 observer 上，这样当数据变化的时候，就会触发 observer，进而触发相关依赖对应的视图更新方法，最后达到模板原本的关联效果。
所以整个 vm 的核心，就是如何实现 observer, directive (parser), watcher 这三样东西

####vue文件结构

http://img4.tbcdn.cn/L1/461/1/cb73a147451157e52500734c0d31665a9540adae

####数据列表的更新

视图更新效率的焦点问题主要在于大列表的更新和深层数据更新这两方面.

但是工作中经常用的主要是前者

首先 diff(data, oldVms) 这个函数的注释对整个比对更新机制做了个简要的阐述，大概意思是先比较新旧两个列表的 vm 的数据的状态，然后差量更新 DOM。


第一步：便利新列表里的每一项，如果该项的 vm 之前就存在，则打一个 _reused 的标，如果不存在对应的 vm，则创建一个新的。

    for (i = 0, l = data.length; i < l; i++) {
            item = data[i];
            key = convertedFromObject ? item.$key : null;
            value = convertedFromObject ? item.$value : item;
            primitive = !isObject(value);
            frag = !init && this.getCachedFrag(value, i, key);
            if (frag) {
              // reusable fragment如果存在打上usered
              frag.reused = true;
              // update $index
              frag.scope.$index = i;
              // update $key
              if (key) {
                frag.scope.$key = key;
              }
              // update iterator
              if (iterator) {
                frag.scope[iterator] = key !== null ? key : i;
              }
              // update data for track-by, object repeat &
              // primitive values.
              if (trackByKey || convertedFromObject || primitive) {
                frag.scope[alias] = value;
              }
            } else {
              // new isntance如果不存在就新建一个
              frag = this.create(value, alias, i, key);
              frag.fresh = !init;
            }
            frags[i] = frag;
            if (init) {
              frag.before(end);
            }
          }

第二步：便利旧列表里的每一项，如果 _reused 的标没有被打上，则说明新列表里已经没有它了，就地销毁该 vm。

    for (i = 0, l = oldFrags.length; i < l; i++) {
        frag = oldFrags[i];
        if (!frag.reused) {
    //如果没有used说明不存在,就地销毁
          this.deleteCachedFrag(frag);
          this.remove(frag, removalIndex++, totalRemoved, inDocument);
        }
      }

第三步：整理新的 vm 在视图里的顺序，同时还原之前打上的 _reused 标。就此列表更新完成

    for (i = 0, l = frags.length; i < l; i++) {
            frag = frags[i];
            // this is the frag that we should be after
            targetPrev = frags[i - 1];
            prevEl = targetPrev ? targetPrev.staggerCb ? targetPrev.staggerAnchor : targetPrev.end || targetPrev.node : start;
            if (frag.reused && !frag.staggerCb) {
              currentPrev = findPrevFrag(frag, start, this.id);
              if (currentPrev !== targetPrev && (!currentPrev ||
              // optimization for moving a single item.
              // thanks to suggestions by @livoras in #1807
              findPrevFrag(currentPrev, start, this.id) !== targetPrev)) {
                this.move(frag, prevEl);
              }
            } else {
              // new instance, or still in stagger.
              // insert with updated stagger index.
              this.insert(frag, insertionIndex++, prevEl, inDocument);
            }
    //还原打上的used
            frag.reused = frag.fresh = false;
          }

keep-alive
          Vue.js 为其组件设计了一个 [keep-alive] 的特性，如果这个特性存在，那么在组件被重复创建的时候，会通过缓存机制快速创建组件，以提升视图更新的性能。

              bind: function bind() {
          if (!this.el.__vue__) {
            // keep-alive cache
            this.keepAlive = this.params.keepAlive;
            if (this.keepAlive) {
              this.cache = {};
            }
    .....
    }

### 数据监听机制

#### 对象数据监听
'Vue'使用'Object.defineProperty'这个'API'为想要监听的属性增加了对应的'getter'和'setter',每次数据改变的时候在setter中触发函数'dep.notify()',来达到数据监听的效果

    //对要监听的属性使用Object.defineProperty重写get和set函数,增加setter和getter方法
      Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            get: function reactiveGetter() {
              //增加getter
              var value = getter ? getter.call(obj) : val;
              if (Dep.target) {
              dep.depend();
              if (childOb) {
                childOb.dep.depend();
              }
              if (isArray(value)) {
                for (var e, i = 0, l = value.length; i < l; i++) {
                  e = value[i];
                  e && e.__ob__ && e.__ob__.dep.depend();
                }
              }
            }
              return value;
            },
            set: function reactiveSetter(newVal) {
              var value = getter ? getter.call(obj) : val;
              //在属性set value的时候调用!!!
              if (newVal === value) {
                return;
              }
              //增加setter
              if (setter) {
                setter.call(obj, newVal);
              } else {
                val = newVal;
              }
              childOb = observe(newVal);
              //最后调用一个自己的函数
              dep.notify();
            }
          });

          然后dep.notify()都做了什么呢?
          ```
            Dep.prototype.notify = function () {
              // stablize the subscriber list first
              var subs = toArray(this.subs)
              for (var i = 0, l = subs.length; i < l; i++) {
                //对相应的数据进行更新
                subs[i].update()
              }
            }
          ```
          dep在文档里面定义是:
          ```
            //A dep is an observable that can have multiple
            //directives subscribing to it.
            export default function Dep () {
              this.id = uid++
              this.subs = []
            }
          ```
'dep'是维护数据的一个数组,对应着一个'watcher'对象

所以整个数据监听的完成是靠set给属性提供一个setter然后当数据更新时,dep会触发watcher对象,返回新值.

之后会有更详细解释

数组可能会有点麻烦，Vue.js 采取的是对几乎每一个可能改变数据的方法进行 prototype 更改：

    ;['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse'].forEach(function (method) {
        // cache original method
        var original = arrayProto[method];
        def(arrayMethods, method, function mutator() {
          // avoid leaking arguments:
          // http://jsperf.com/closure-with-arguments
          var i = arguments.length;
          var args = new Array(i);
          while (i--) {
            args[i] = arguments[i];
          }
          var result = original.apply(this, args);
          var ob = this.__ob__;
          var inserted;
          switch (method) {
            case 'push':
              inserted = args;
              break;
            case 'unshift':
              inserted = args;
              break;
            case 'splice':
              inserted = args.slice(2);
              break;
          }
          if (inserted) ob.observeArray(inserted);
          // notify change
          ob.dep.notify();
          return result;
        });
      });


同时 Vue.js 提供了两个额外的“糖方法” $set 和 $remove 来弥补这方面限制带来的不便。

但这个策略主要面临两个问题：

无法监听数据的 length，导致 arr.length 这样的数据改变无法被监听
通过角标更改数据，即类似 arr[2] = 1 这样的赋值操作，也无法被监听

为此 Vue.js 在文档中明确提示不建议直接角标修改数据

"实例计算属性。getter 和 setter 的 this 自动地绑定到实例。"

举个栗子:


    var vm = new Vue({
      data: { a: 1 },
      computed: {
        // 仅读取，值只须为函数
        b: function () {
          return this.a * 2
        },
        // 读取和设置
        c: {
          get: function () {
            return this.a + 1
          },
          set: function (v) {
            this.a = v - 1
          }
        }
      }
      })

可以看出来computed可以提供自定义一个属性c的getter和setter/b的getter,问题是c和b怎么维护和a的关系

下面是computed怎么提供属性setter和getter的代码:

    ```
      //初始化computed
      ...
      var userDef = computed[key];
      //userDef指的是computed属性,this -> computed
      def.get = makeComputedGetter(userDef, this);
      //或者makeComputedGetter(userDef.get, this)
      ...
      function makeComputedGetter(getter, owner) {
          var watcher = new Watcher(owner, getter, null, {
            lazy: true
          });
          return function computedGetter() {
            if (watcher.dirty) {
              watcher.evaluate();
            }
            if (Dep.target) {
              watcher.depend();
            }
            return watcher.value;
          };
        }
    ```

computed在建立的时候绑定一个对应的 watcher 对象，在计算过程中它把属性记录为依赖。之后当依赖的 setter 被调用时，会触发 watcher 重新计算 ，也就会导致它的关联指令更新 DOM。

### 视图解析过程

### 解析器

parsers/path.js 主要的职责是可以把一个 JSON 数据里的某一个“路径”下的数据取出来，比如：

    var path = 'a.b[1].v'
    var obj = {
      a: {
        b: [
          {v: 1},
          {v: 2},
          {v: 3}
        ]
      }
    }
    parse(obj, path) // 2

    var pathStateMachine = []

    pathStateMachine[BEFORE_PATH] = {
      'ws': [BEFORE_PATH],
      'ident': [IN_IDENT, APPEND],
      '[': [IN_SUB_PATH],
      'eof': [AFTER_PATH]
    }

    pathStateMachine[IN_PATH] = {
      'ws': [IN_PATH],
      '.': [BEFORE_IDENT],
      '[': [IN_SUB_PATH],
      'eof': [AFTER_PATH]
    }

    pathStateMachine[BEFORE_IDENT] = {
      'ws': [BEFORE_IDENT],
      'ident': [IN_IDENT, APPEND]
    }

    pathStateMachine[IN_IDENT] = {
      'ident': [IN_IDENT, APPEND],
      '0': [IN_IDENT, APPEND],
      'number': [IN_IDENT, APPEND],
      'ws': [IN_PATH, PUSH],
      '.': [BEFORE_IDENT, PUSH],
      '[': [IN_SUB_PATH, PUSH],
      'eof': [AFTER_PATH, PUSH]
    }

    pathStateMachine[IN_SUB_PATH] = {
      "'": [IN_SINGLE_QUOTE, APPEND],
      '"': [IN_DOUBLE_QUOTE, APPEND],
      '[': [IN_SUB_PATH, INC_SUB_PATH_DEPTH],
      ']': [IN_PATH, PUSH_SUB_PATH],
      'eof': ERROR,
      'else': [IN_SUB_PATH, APPEND]
    }

    pathStateMachine[IN_SINGLE_QUOTE] = {
      "'": [IN_SUB_PATH, APPEND],
      'eof': ERROR,
      'else': [IN_SINGLE_QUOTE, APPEND]
    }

    pathStateMachine[IN_DOUBLE_QUOTE] = {
      '"': [IN_SUB_PATH, APPEND],
      'eof': ERROR,
      'else': [IN_DOUBLE_QUOTE, APPEND]
    }

状态机可以完成

    1.dom结构中{{data.someObj}}的解析;
    2.对字符型json的取值;

可惜大学里面的编译原理我给忘记了,否则可以给大家解析一下.

#### 视图解析过程

视图的解析过程，Vue.js 的策略是把 element 或 template string 先统一转换成 document fragment，然后再分解和解析其中的子组件和 directives。

相比React的visual DOM有一定的性能优化空间，毕竟 DOM 操作相比纯 JavaScript 运算还是会慢一些。

### Vue扩展

#### Mixin

Mixin (混入) 是一种可以在多个 Vue 组件之间灵活复用特性的机制。你可以像写一个普通 Vue 组件的选项对象一样编写一个 mixin：

    module.exports = {
      created: function () {
        this.hello()
      },
      methods: {
        hello: function () {
          console.log('hello from mixin!')
        }
      }
    }



    // test.js
    var myMixin = require('./mixin')
    var Component = Vue.extend({
      mixins: [myMixin]
    })
    var component = new Component() // -> "hello from mixin!"

#### Vue插件
    Vue插件类型分为以下几种:

    1.添加一个或几个全局方法。比如 vue-element
    2.添加一个或几个全局资源：指令、过滤器、动画效果等。比如
    vue-touch
    3.通过绑定到 Vue.prototype 的方式添加一些 Vue 实例方法。这里有个约定，就是 Vue 的实例方法应该带有 $ 前缀，这样就不会和用户的数据和方法产生冲突了。

##### 开发Vue插件

    MyPlugin.install = function (Vue, options) {
    // 1. 添加全局方法或属性
    Vue.myGlobalMethod = ...
    // 2. 添加全局资源
    Vue.directive('my-directive', {})
    // 3. 添加实例方法
    Vue.prototype.$myMethod = ...
    }

##### 使用Vue插件

    var vueTouch = require('vue-touch')
    // use the plugin globally
    Vue.use(vueTouch)
    你也可以向插件里传递额外的选项：

    Vue.use(require('my-plugin'), {
    /* pass in additional options */
    })

    全局方法:
    Vue.fun()
    局部方法:
    vm.$fun()

#### Vue指令

Vue.js 允许注册自定义指令，实质上是开放 Vue 一些技巧：怎样将数据的变化映射到 DOM 的行为。你可以使用 Vue.directive(id, definition) 的方法传入指令 id 和定义对象来注册一个全局自定义指令。定义对象需要提供一些钩子函数：
bind： 仅调用一次，当指令第一次绑定元素的时候。
update： 第一次是紧跟在 bind 之后调用，获得的参数是绑定的初始值；以后每当绑定的值发生变化就会被调用，获得新值与旧值两个参数。
unbind：仅调用一次，当指令解绑元素的时候。

一旦注册好自定义指令，你就可以在 Vue.js 模板中像这样来使用它（需要添加 Vue.js 的指令前缀，默认为 `v-`）：

`<div v-my-directive="someValue"></div>`

如果你只需要 `update` 函数，你可以只传入一个函数，而不用传定义对象：

```Vue.directive('my-directive', function (value) {
  // 这个函数会被作为 update() 函数使用
})```

所有的钩子函数会被复制到实际的**指令对象**中，而这个指令对象将会是所有钩子函数的 `this` 上下文环境。指令对象上暴露了一些有用的公开属性：

- **el**： 指令绑定的元素
- **vm**： 拥有该指令的上下文 ViewModel
- **expression**： 指令的表达式，不包括参数和过滤器
- **arg**： 指令的参数
- **raw**： 未被解析的原始表达式
- **name**： 不带前缀的指令名

>这些属性是只读的，不要修改它们。你也可以给指令对象附加自定义的属性，但是注意不要覆盖已有的内部属性。

使用指令对象属性的示例：

`<div id="demo" v-demo="LightSlateGray : msg"></div>`

```Vue.directive('demo', {
  bind: function () {
    this.el.style.color = '#fff'
    this.el.style.backgroundColor = this.arg
  },
  update: function (value) {
    this.el.innerHTML =
      'name - '       + this.name + '<br>' +
      'raw - '        + this.raw + '<br>' +
      'expression - ' + this.expression + '<br>' +
      'argument - '   + this.arg + '<br>' +
      'value - '      + value
  }
})
var demo = new Vue({
  el: '#demo',
  data: {
    msg: 'hello!'
  }
})```

**Result**

- name - demo
- raw - LightSlateGray：msg
- expression - msg
- argument - LightSlateGray
- value - hello!

### 多重从句

同一个特性内部，逗号分隔的多个从句将被绑定为多个指令实例。在下面的例子中，指令会被创建和调用两次：

`<div v-demo="color: 'white', text: 'hello!'"></div>`

如果想要用单个指令实例处理多个参数，可以利用字面量对象作为表达式：

`<div v-demo="{color: 'white', text: 'hello!'}"></div>`

```Vue.directive('demo', function (value) {
  console.log(value) // Object {color: 'white', text: 'hello!'}
})```

## 字面指令

如果在创建自定义指令的时候传入 `isLiteral: true`，那么特性值就会被看成直接字符串，并被赋值给该指令的 `expression`。字面指令不会试图建立数据监视。

**Example**：

`<div v-literal-dir="foo"></div>`

```Vue.directive('literal-dir', {
  isLiteral: true,
  bind: function () {
    console.log(this.expression) // 'foo'
  }
})```

### 动态字面指令

然而，在字面指令含有 `Mustache` 标签的情形下，指令的行为如下：

- 指令实例会有一个属性，`this._isDynamicLiteral` 被设为 `true`；

- 如果没有提供 `update` 函数，`Mustache` 表达式只会被求值一次，并将该值赋给 `this.expression` 。不会对表达式进行数据监视。

- 如果提供了 `update` 函数，指令将会为表达式建立一个数据监视，并且在计算结果变化的时候调用 `update`。

## 双向指令

如果你的指令想向 Vue 实例写回数据，你需要传入 `twoWay: true` 。该选项允许在指令中使用 `this.set(value)`。

```Vue.directive('example', {
  twoWay: true,
  bind: function () {
    this.handler = function () {
      // 把数据写回 vm
      // 如果指令这样绑定 v-example="a.b.c",
      // 这里将会给 `vm.a.b.c` 赋值
      this.set(this.el.value)
    }.bind(this)
    this.el.addEventListener('input', this.handler)
  },
  unbind: function () {
    this.el.removeEventListener('input', this.handler)
  }
})```

## 内联语句

传入 `acceptStatement: true` 可以让自定义指令像 `v-on` 一样接受内联语句：

`<div v-my-directive="a++"></div>`

```Vue.directive('my-directive', {
  acceptStatement: true,
  update: function (fn) {
    // the passed in value is a function which when called,
    // will execute the "a++" statement in the owner vm's
    // scope.
  }
})```

但是请明智地使用此功能，因为通常我们希望避免在模板中产生副作用。

## 深度数据观察

如果你希望在一个对象上使用自定义指令，并且当对象内部嵌套的属性发生变化时也能够触发指令的 `update` 函数，那么你就要在指令的定义中传入 `deep: true`。

`<div v-my-directive="obj"></div>`

```Vue.directive('my-directive', {
  deep: true,
  update: function (obj) {
    // 当 obj 内部嵌套的属性变化时也会调用此函数
  }
})```

## 指令优先级

你可以选择给指令提供一个优先级数（默认是 0）。同一个元素上优先级越高的指令会比其他的指令处理得早一些。优先级一样的指令会按照其在元素特性列表中出现的顺序依次处理，但是不能保证这个顺序在不同的浏览器中是一致的。

通常来说作为用户，你并不需要关心内置指令的优先级，如果你感兴趣的话，可以参阅源码。逻辑控制指令 `v-repeat`， `v-if` 被视为 “终结性指令”，它们在编译过程中始终拥有最高的优先级。

## 元素指令

有时候，我们可能想要我们的指令可以以自定义元素的形式被使用，而不是作为一个特性。这与 `Angular` 的 `E` 类指令的概念非常相似。元素指令可以看做是一个轻量的自定义组件（后面会讲到）。你可以像下面这样注册一个自定义的元素指令：

```Vue.elementDirective('my-directive', {
  // 和普通指令的 API 一致
  bind: function () {
    // 对 this.el 进行操作...
  }
})

### Vue扩展

#### Mixin

Mixin (混入) 是一种可以在多个 Vue 组件之间灵活复用特性的机制。你可以像写一个普通 Vue 组件的选项对象一样编写一个 mixin：

    module.exports = {
      created: function () {
        this.hello()
      },
      methods: {
        hello: function () {
          console.log('hello from mixin!')
        }
      }
    }



    // test.js
    var myMixin = require('./mixin')
    var Component = Vue.extend({
      mixins: [myMixin]
    })
    var component = new Component() // -> "hello from mixin!"

#### Vue插件
    Vue插件类型分为以下几种:

    1.添加一个或几个全局方法。比如 vue-element
    2.添加一个或几个全局资源：指令、过滤器、动画效果等。比如
    vue-touch
    3.通过绑定到 Vue.prototype 的方式添加一些 Vue 实例方法。这里有个约定，就是 Vue 的实例方法应该带有 $ 前缀，这样就不会和用户的数据和方法产生冲突了。

##### 开发Vue插件

    MyPlugin.install = function (Vue, options) {
    // 1. 添加全局方法或属性
    Vue.myGlobalMethod = ...
    // 2. 添加全局资源
    Vue.directive('my-directive', {})
    // 3. 添加实例方法
    Vue.prototype.$myMethod = ...
    }

##### 使用Vue插件

    var vueTouch = require('vue-touch')
    // use the plugin globally
    Vue.use(vueTouch)
    你也可以向插件里传递额外的选项：

    Vue.use(require('my-plugin'), {
    /* pass in additional options */
    })

    全局方法:
    Vue.fun()
    局部方法:
    vm.$fun()

#### Vue指令

Vue.js 允许注册自定义指令，实质上是开放 Vue 一些技巧：怎样将数据的变化映射到 DOM 的行为。你可以使用 Vue.directive(id, definition) 的方法传入指令 id 和定义对象来注册一个全局自定义指令。定义对象需要提供一些钩子函数：
bind： 仅调用一次，当指令第一次绑定元素的时候。
update： 第一次是紧跟在 bind 之后调用，获得的参数是绑定的初始值；以后每当绑定的值发生变化就会被调用，获得新值与旧值两个参数。
unbind：仅调用一次，当指令解绑元素的时候。

一旦注册好自定义指令，你就可以在 Vue.js 模板中像这样来使用它（需要添加 Vue.js 的指令前缀，默认为 `v-`）：

`<div v-my-directive="someValue"></div>`

如果你只需要 `update` 函数，你可以只传入一个函数，而不用传定义对象：

```Vue.directive('my-directive', function (value) {
  // 这个函数会被作为 update() 函数使用
})```

所有的钩子函数会被复制到实际的**指令对象**中，而这个指令对象将会是所有钩子函数的 `this` 上下文环境。指令对象上暴露了一些有用的公开属性：

- **el**： 指令绑定的元素
- **vm**： 拥有该指令的上下文 ViewModel
- **expression**： 指令的表达式，不包括参数和过滤器
- **arg**： 指令的参数
- **raw**： 未被解析的原始表达式
- **name**： 不带前缀的指令名

>这些属性是只读的，不要修改它们。你也可以给指令对象附加自定义的属性，但是注意不要覆盖已有的内部属性。

使用指令对象属性的示例：

`<div id="demo" v-demo="LightSlateGray : msg"></div>`

```Vue.directive('demo', {
  bind: function () {
    this.el.style.color = '#fff'
    this.el.style.backgroundColor = this.arg
  },
  update: function (value) {
    this.el.innerHTML =
      'name - '       + this.name + '<br>' +
      'raw - '        + this.raw + '<br>' +
      'expression - ' + this.expression + '<br>' +
      'argument - '   + this.arg + '<br>' +
      'value - '      + value
  }
})
var demo = new Vue({
  el: '#demo',
  data: {
    msg: 'hello!'
  }
})```

**Result**

- name - demo
- raw - LightSlateGray：msg
- expression - msg
- argument - LightSlateGray
- value - hello!

### 多重从句

同一个特性内部，逗号分隔的多个从句将被绑定为多个指令实例。在下面的例子中，指令会被创建和调用两次：

`<div v-demo="color: 'white', text: 'hello!'"></div>`

如果想要用单个指令实例处理多个参数，可以利用字面量对象作为表达式：

`<div v-demo="{color: 'white', text: 'hello!'}"></div>`

```Vue.directive('demo', function (value) {
  console.log(value) // Object {color: 'white', text: 'hello!'}
})```

## 字面指令

如果在创建自定义指令的时候传入 `isLiteral: true`，那么特性值就会被看成直接字符串，并被赋值给该指令的 `expression`。字面指令不会试图建立数据监视。

**Example**：

`<div v-literal-dir="foo"></div>`

```Vue.directive('literal-dir', {
  isLiteral: true,
  bind: function () {
    console.log(this.expression) // 'foo'
  }
})```

### 动态字面指令

然而，在字面指令含有 `Mustache` 标签的情形下，指令的行为如下：

- 指令实例会有一个属性，`this._isDynamicLiteral` 被设为 `true`；

- 如果没有提供 `update` 函数，`Mustache` 表达式只会被求值一次，并将该值赋给 `this.expression` 。不会对表达式进行数据监视。

- 如果提供了 `update` 函数，指令将会为表达式建立一个数据监视，并且在计算结果变化的时候调用 `update`。

## 双向指令

如果你的指令想向 Vue 实例写回数据，你需要传入 `twoWay: true` 。该选项允许在指令中使用 `this.set(value)`。

```Vue.directive('example', {
  twoWay: true,
  bind: function () {
    this.handler = function () {
      // 把数据写回 vm
      // 如果指令这样绑定 v-example="a.b.c",
      // 这里将会给 `vm.a.b.c` 赋值
      this.set(this.el.value)
    }.bind(this)
    this.el.addEventListener('input', this.handler)
  },
  unbind: function () {
    this.el.removeEventListener('input', this.handler)
  }
})```

## 内联语句

传入 `acceptStatement: true` 可以让自定义指令像 `v-on` 一样接受内联语句：

`<div v-my-directive="a++"></div>`

```Vue.directive('my-directive', {
  acceptStatement: true,
  update: function (fn) {
    // the passed in value is a function which when called,
    // will execute the "a++" statement in the owner vm's
    // scope.
  }
})```

但是请明智地使用此功能，因为通常我们希望避免在模板中产生副作用。

## 深度数据观察

如果你希望在一个对象上使用自定义指令，并且当对象内部嵌套的属性发生变化时也能够触发指令的 `update` 函数，那么你就要在指令的定义中传入 `deep: true`。

`<div v-my-directive="obj"></div>`

```Vue.directive('my-directive', {
  deep: true,
  update: function (obj) {
    // 当 obj 内部嵌套的属性变化时也会调用此函数
  }
})```

## 指令优先级

你可以选择给指令提供一个优先级数（默认是 0）。同一个元素上优先级越高的指令会比其他的指令处理得早一些。优先级一样的指令会按照其在元素特性列表中出现的顺序依次处理，但是不能保证这个顺序在不同的浏览器中是一致的。

通常来说作为用户，你并不需要关心内置指令的优先级，如果你感兴趣的话，可以参阅源码。逻辑控制指令 `v-repeat`， `v-if` 被视为 “终结性指令”，它们在编译过程中始终拥有最高的优先级。

## 元素指令

有时候，我们可能想要我们的指令可以以自定义元素的形式被使用，而不是作为一个特性。这与 `Angular` 的 `E` 类指令的概念非常相似。元素指令可以看做是一个轻量的自定义组件（后面会讲到）。你可以像下面这样注册一个自定义的元素指令：

```Vue.elementDirective('my-directive', {
  // 和普通指令的 API 一致
  bind: function () {
    // 对 this.el 进行操作...
  }
})

### vuejs vs angularjs
    Angular Modules
    angular.module('myModule', [...]);
    Components
    Vue.extend({
      data: function(){ return {...} },
      created: function() {...},
      ready: function() {...},
      components: {...},
      methods: {...},

总体来说
对于Angular来说module就是一个容器,而对Vue来说一个component里面会有逻辑代码
在Vue里面会放进许多代码细节,并且有固定的属性

    Directives
    Angular
    myModule.directive('directiveName', function (injectables) {
      return {
        restrict: 'A',
        template: '<div></div>',
        controller: function() { ... },
        compile: function() {...},
        link: function() { ... }
        //(other props excluded)
      };
    });
    Vue
    Vue.directive('my-directive', {
      bind: function () {...},
      update: function (newValue, oldValue) {...},
      unbind: function () {...}
    });

Vue的指令比Angular的简单,而Angular的指令类似Vue的component

    Filters
    Angular
    myModule.angular.module(‘filterName', [])
    .filter('reverse', function() {
    return function(input) {...};
    });
    Vue
    Vue.filter('reverse', function (value) {
    return function(value){...};
    });


filters都是类似的,但是Vue提供了read/wirte功能

    Templating
    Interpolation
    {{myVariable}}
    Interpolation
    {{myVariable}}


当输出是一个对象的时候
Vue:[Object]
Angular :{[attr:value]}
Vue可以使用filters得到正常输出 {{someObject|json}}

    Model binding
    Angular
    <input type="text" ng-model="myVar">
    <p ng-bind="myVar"></p>
    Vue
    <input type="text" v-model="myVar">
    <p v-model="myVar"></p>

    Loops
    Angular
    <li ng-repeat="item in items" class="item-{{$index}}">
      {{item.myProperty}}
    </li>
    Vue
    <li v-for="items" class="item-{{$index}}">
      {{myProperty}}
    </li>

    Conditionals
    Angular
    <div ng-if="myVar"></div>
    <div ng-show="myVar"></div>
    Vue
    <div v-if="myVar"></div>
    <div v-show="myVar"></div>

    Conditional classes
    Angular
    <div ng-class="{‘active’: myVar}"></div>
    Vue
    <div v-class="active: myVar"></div>

Vue也可以这样写v-repeat='item: items'

    Event binding
    Angular
    <div ng-click="myMethod($event)"></div>
    Vue
    <div v-on="click: myMethod($event)"></div>

通用v-on指令使事件更加一致

#### 脏值检查
一个电话列表应用的例子，在其中我们会将一个phones数组中的值（在JavaScript中定义）绑定到一个列表项目中以便于我们的数据和UI保持同步：

    <code><html ng-app>
      <head>
    ...
    <script src="angular.js"></script>
    <script src="controller.js"></script>
      </head>
      <body ng-controller="PhoneListCtrl">
    <ul>
      <li ng-repeat="phone in phones">
    {{phone.name}}
    <p>{{phone.snippet}}</p>
      </li>
    </ul>
      </body>
    </html>
    </code>

    <code>var phonecatApp = angular.module('phonecatApp', []);

    phonecatApp.controller('PhoneListCtrl', function($scope) {
      $scope.phones = [
    {'name': 'Nexus S',
     'snippet': 'Fast just got faster with Nexus S.'},
    {'name': 'Motorola XOOM with Wi-Fi',
     'snippet': 'The Next, Next Generation tablet.'},
    {'name': 'MOTOROLA XOOM',
     'snippet': 'The Next, Next Generation tablet.'}
      ];
    });  
    </code>

任何时候只要是底层的model数据发生了变化，我们在DOM中的列表也会跟着更新。

脏值检查的基本原理就是只要任何时候数据发生了变化，这个库都会通过一个digest或者change cycle去检查变化是否发生了。在Angular中，一个digest循环意味着所有所有被监视的表达式都会被循环一遍以便查看其中是否有变化发生。它智斗一个模型之前的值因此当变化发生时，一个change事件将会被触发。对于开发者来说，这带来的一大好处就是你可以使用原生的JavaScript对象数据，它易于使用及整合。下面的图片展示的是一个非常糟糕的算法，它的开销非常大。

这个操作的开销和被监视的对象的数量是成正比的。我可能需要做很多的脏治检查。同时当数据发生改变时,我也需要一种方式去触发脏值检查.

相比Angular的脏值检查,Vue的setter/getter方案使数据和DOM更新的时间复杂度降低,数据的更新只发生在数据发生改变时,数据更新的时间复杂度只和数据的观察者有关,"它们拥有一些存取器去获取数据并且能够在你设置或者获取对象时捕获到这些行为并在内部进行广播".

#### vue的约束的模型系统

而且相比Object.observer()[在es7标准中],Vue的存取方式可以做到比较好的兼容性.

### Vue实现简单的watcher

    1.实现observer
    2.Vue消息-订阅器
    3.Watcher的实现
    4.实现一个Vue

实现一个 $wacth

    ```
    const v = new Vue({
      data:{
        a:1,
        b:2
      }
    })
    v.$watch("a",()=>console.log("哈哈，$watch成功"))
    setTimeout(()=>{
      v.a = 5
    },2000) //打印 哈哈，$watch成功
    ```
为了帮助大家理清思路。。我们就做最简单的实现。。只考虑对象不考虑数组

##### 实现obserer

将要observe的对象， 通过递归，将它所有的属性，包括子属性的属性，都给加上set和get， 这样的话，给这个对象的某个属性赋值，就会触发set。就给每个属性（包括子属性）都加上get/set， 这样的话，这个对象的，有任何赋值，就会触发set方法。

    export default class  Observer{
      constructor(value) {
        this.value = value
        this.walk(value)
      }
      //递归。。让每个字属性可以observe
      walk(value){
        Object.keys(value).forEach(key=>this.convert(key,value[key]))
      }
      convert(key, val){
        defineReactive(this.value, key, val)
      }
    }


    export function defineReactive (obj, key, val) {
      var childOb = observe(val)
      Object.defineProperty(obj, key, {
        enumerable: true,
        configurable: true,
        get: ()=>val,
        set:newVal=> {      
         childOb = observe(newVal)//如果新赋值的值是个复杂类型。再递归它，加上set/get。。
         }
      })
    }


    export function observe (value, vm) {
      if (!value || typeof value !== 'object') {
        return
      }
      return new Observer(value)
    }

##### 消息－订阅器

维护一个数组，，这个数组，就放订阅着，一旦触发notify， 订阅者就调用自己的update方法

    export default class Dep {
      constructor() {
        this.subs = []
      }
      addSub(sub){
        this.subs.push(sub)
      }
      notify(){
        this.subs.forEach(sub=>sub.update())
      }
    }

每次set函数，调用的时候，我们是不是应该，触发notify，对吧。所以 我们把代码补充完整

    export function defineReactive (obj, key, val) {
          var dep = new Dep()
          var childOb = observe(val)
          Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            get: ()=>val,
            set:newVal=> {
              var value =  val
              if (newVal === value) {
                return
              }
              val = newVal
              childOb = observe(newVal)
              dep.notify()
            }
          })
        }

##### 实现一个Watcher

         v.$watch("a",()=>console.log("哈哈，$watch成功"))

我们想象这个Watcher，应该用什么东西。update方法，嗯这个毋庸置疑， 还有呢，
对表达式（就是那个“a”） 和 回调函数，这是最基本的，所以我们简单写写

    export default class Watcher {
      constructor(vm, expOrFn, cb) {
        this.cb = cb
        this.vm = vm
        //此处简化.要区分fuction还是expression,只考虑最简单的expression
        this.expOrFn = expOrFn
        this.value = this.get()
      }
      update(){
        this.run()
      }
      run(){
        const  value = this.get()
        if(value !==this.value){
          this.value = value
          this.cb.call(this.vm)
        }
      }
      get(){
        //此处简化。。要区分fuction还是expression
        const value = this.vm._data[this.expOrFn]
        return value
      }
    }

怎样将通过addSub(),将Watcher加进去呢。 我们发现var dep = new Dep() 处于闭包当中， 我们又发现Watcher的构造函数里会调用this.get 所以，我们可以在上面动动手脚， 修改一下Object.defineProperty的get要调用的函数， 判断是不是Watcher的构造函数调用，如果是，说明他就是这个属性的订阅者 果断将他addSub()中去，那问题来了， 我怎样判断他是Watcher的this.get调用的，而不是我们普通调用的呢

    export default class Watcher {
      ....省略未改动代码....
      get(){
        Dep.target = this
        //此处简化。。要区分fuction还是expression
        const value = this.vm._data[this.expOrFn]
        Dep.target = null
        return value
      }
    }

这样的话，我们只需要在Object.defineProperty的get要调用的函数里， 判断有没有值，就知道到底是Watcher 在get，还是我们自己在查看赋值，如果 是Watcher的话就addSub(),代码补充一下

    export function defineReactive (obj, key, val) {
    var dep = new Dep()
    var childOb = observe(val)

    Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: ()=>{
      // 说明这是watch 引起的
      if(Dep.target){
        dep.addSub(Dep.target)
      }
      return val
    },
    set:newVal=> {
      var value =  val
      if (newVal === value) {
        return
      }
      val = newVal
      childOb = observe(newVal)
      dep.notify()
    }
    })
    }

最后不要忘记，在Dep.js中加上这么一句

    Dep.target = null

##### 实现一个Vue

我们要把以上代码配合Vue的$watch方法来用， 要watch Vue实例的属性:

    import Watcher from '../watcher'
    import {observe} from "../observer"

    export default class Vue {
      constructor (options={}) {
        //这里简化了。。其实要merge
        this.$options=options
        //这里简化了。。其实要区分的
        let data = this._data=this.$options.data
        Object.keys(data).forEach(key=>this._proxy(key))
        observe(data,this)
      }


      $watch(expOrFn, cb, options){
        new Watcher(this, expOrFn, cb)
      }

      _proxy(key) {
        var self = this
        Object.defineProperty(self, key, {
          configurable: true,
          enumerable: true,
          get: function proxyGetter () {
            return self._data[key]
          },
          set: function proxySetter (val) {
            self._data[key] = val
          }
        })
      }
    }

两件事，observe自己的data，代理自己的data， 使访问自己的属性，就是访问子data的属性。
