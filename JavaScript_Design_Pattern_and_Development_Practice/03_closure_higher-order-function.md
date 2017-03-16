## 1. 闭包（作用域+生存周期）
### 1.1 变量的作用域
1. 函数内不用var定义的变量是全局的
2. 变量搜索++从内到外++

### 1.2 变量的生存周期
```
var func = function(){
    var a = 0;
    return function(){
        a++;
        console.log(a);
    }
};
var f = func();
f();    // 1
f();    // 2
```
执行函数f，f返回一个匿名函数的引用，可以访问到内部函数func()被调用时产生的环境，局部的a在这个环境里 -> 局部变量的所在环境能被访问到，局部变量不被销毁。

1. 经典使用1 - 点击body中的div，弹出其对应的下标
    ```
    <html>
        <body>
        <div>1</div>
        <div>2</div>
        <div>3</div>
        <div>4</div>
        <div>5</div>
        // 五个节点，点击时弹出对应索引
        <script>
            var nodes = document.getElementsByTagName( 'div' );
            for ( var i = 0, len = nodes.length; i < len; i++ ){
                nodes[ i ].onclick = function(){    // onclick事件异步触发，for循环执行完了才触发事件
                    alert ( i );    // i已经变成5，不论点击那个节点都弹出5
                }
            };
        </script>
        // 用闭包把局部变量和其调用环境保留下来
        <script>
            var nodes = document.getElementsByTagName( 'div' );
            for ( var i = 0, len = nodes.length; i < len; i++ ){
                (function(i){
                    nodes[ i ].onclick = function(){
                        alert ( i );
                    }
                })(i);
            };
        </script>
        </body>
    </html>
    ```
2. 经典使用2 - 模拟判断变量类型的方法
    ```
    var Type = {};
    for(var i = 0, type; type = ['String', 'Array', 'Number'][i++]){
        (function(type){
            Type['is' + type] = function(obj){
                retuen Object.prototype.toString.call(obj) === '[object ' + type + ']';
            };
        })(type);
    };
    Type.isArray([]);   // true
    Type.isString('str');   // true
    ```
### 1.3 更多作用
1. 封装变量
    将变量封装成“私有变量”
    ```
    var mult = function(){
        var a = 1;
        for(var i = 0, l = arguments.length; i < l; i++){
            a = a * arguments[i];
        }
        return a;
    };
    ```
    mult函数接受number型参数进行相乘计算，如果参数相同（顺序可不同），都执行mult显然是无意义的行为，可以加入缓存
    ```
    var cache = {};
    var mult = function(){
        arguments = Array.prototype.sort.call(arguments);
        var args = Array.prototype.sort.join.call(arguments, ',');
        if (args in cache){
            return cache[args];
        }
        var a = 1;
        for(var i = 0, l = arguments.length; i < l; i++){
            a = a * arguments[i];
        }
        return cache[args] = a;
    }
    ```
    cache仅在mult函数中使用，不必也不宜暴露在全局环境中，加入闭包
    ```
    var mult = (function(){
        var cache = {};
        return function(){
            arguments = Array.prototype.sort.call(arguments);
            var args = Array.prototype.sort.join.call(arguments, ',');
            if (args in cache){
                return cache[args];
            }
            var a = 1;
            for(var i = 0, l = arguments.length; i < l; i++){
                a = a * arguments[i];
            }
            return cache[args] = a;
        };
    })();
    ```
    提炼函数——大块代码中有可独立代码，封装出小函数，有利于复用，如果不需要在其他地方用到这些小函数，用闭包封闭起来。
    ```
    var mult = (function(){
        var cache = {};
        var calc = function(){  // 将乘法计算封装起来，只创建一次，可重复使用
            var a = 1;
            for(var i = 0, l = arguments.length; i < l; i++){
                a = a * arguments[i];
            }
            return a;
        };
        return function(){
            arguments = Array.prototype.sort.call(arguments);
            var args = Array.prototype.sort.join.call(arguments, ',');
            if (args in cache){
                return cache[args];
            }
            return cache[args] = calc.apply(null, arguments);
        };
    })();
    ```
2. 延续局部变量的寿命（？？）
    ```
    var report = (function(){
        var imgs = [];
        return function( src ){
            var img = new Image();
            imgs.push( img );
            img.src = src;
        }
    })();
    ```
### 1.4 闭包和面向对象设计
过程与数据结合是形容面向对象中的“对象”常用的表达。

- | 包含 | 形式
---|---|---
对象 | 过程 | 方法
闭包 | 数据 | 环境

### 1.5 用闭包实现命令模式
命令模式是把请求封装为对象，分离请求的发起者和执行者之间的耦合。在命令执行之前，可以预先往命令呢对象中植入执行者。
```
var TV = function(){
    open: function(){
        console.log('open TV');
    },
    close: function(){
        console.log('close TV');
    },
};

var createCommand = function(receiver){
//先在对象中植入命令的执行者TV，让闭包来做这件事。在闭包版的命令模式中，执行者会被封闭在闭包形成的环境中
    var execute = function(){
        return receiver.open();
    };
    var undo = function(){
        return receiver.close();
    };
    return {
        execute: execute,
        undo: undo
    }
};

var setCommand = function(command){
    document.getElementById('execute').onclick = function(){
        command.execute();
    };
    document.getElementById('undo').onclick = function(){
        command.undo();
    };
};

setCommand(createCommand(TV));  // 函数作为一等对象可以四处传播
```

## 2. 高阶函数
指满足下面条件之一的函数：
- 函数可以作为参数被传递
- 函数可以作为返回值输出

### 2.1 函数作为参数传递
1. 回调函数
- 常用在异步请求，不知道请求什么时候会返回，所以可以把返回之后要做的事情作为callback放在请求函数里
- 把函数作为参数委托给另一个函数执行

2. Array.prototype.sort

    接受一个函数当作参数，该函数封装了数组元素的排序规则。排序是不变的，如何排序是可变的；封装可变的部分在函数参数中使这个函数更加灵活
    Array.prototype.sort(function(a,b){
        return a-b; // 从小到大排序，从大到小则是return b-a;
    });
### 2.2 函数作为返回值返回
1. 判断事件的类型
    ```
    var isType = function(type){
        return function(obj){
            return Object.prototype.toString.call(obj) === '[object ' + type + ']';
        }
    };
    var isString = isType('String');
    var isArray = isType('Array');
    var isNumber = isType('Number');
    ```
    还可以用循环语句来批量注册
    ```
    var Type = {};
    for(var i = 0, type; type = ['String', 'Array', 'Number'][i++];){
        (function(type){
            Type['is' + type] = function(obj){
                return Object.prototype.toString.call(obj) === '[object ' + type + ']';
            };
        })(type)
    }
    Type.isArray([]);
    Type.isString('str');
    ```
2. getSingle

// TODO 实现单例模式，代码暂时还没有看懂。

### 2.3 实现AOP

面向切面编程：把跟核心业务无关的功能抽离出来，使核心逻辑模块纯净和高内聚，其他抽离出来的功能可以复用。
// TODO 看15章的装饰者模式

### 2.4 其他应用
#### 1. currying
函数柯里化，部分求值。接受一些参数不立即求值，返回另一个函数，所传参数被闭包保存起来。待函数真正被需要求值时，之前传入参数会被一次性用于求值。

```
var currying = function(fn){
    var args = [];
    return function(){
        if(arguments.length === 0){
            return fn.apply(this, args);    // 执行函数
        } else {
            [].push.apply(args, arguments); // 明确带上了参数说明不真正求值，保存参数在args
            return arguments.callee;    // 返回传参的函数
        }
    }
};

var cost = (function(){
    var money = 0;
    return function(){
        for(var i = 0, l = arguments.length; i < l; i++){
            money += arguments[i];
        }
        return money;
    }
})();

var cost = currying(cost);  // 转成柯里化函数
cost(100);
cost(200);
cost(300);
cost(); // 求值并输出 600
```
#### 2. uncurrying
提取泛化this的过程
```
Function.prototype.uncurrying = function(){
    var self = this;    //self: Array.prototype.push
    return function(){
        var obj = Array.prototype.shift.call(arguments);
        return self.apply(obj, arguments);  // Array.prototype.push.apply(obj, arguments)
    }
};

// 类数组对象arguments借用Array.prototype的方法
// 先把Array.prototype.push.call转化为通用的push函数
var push = Array.prototype.push.uncurrying();
(function(){
    push(arguments, 4);
    console.log(arguments); // 输出:[1, 2, 3, 4]
})(1, 2, 3);

// 闭包的那几行代码也可以这样看
var obj = {
    "length": 3,
    "0": 1,
    "1": 2,
    "2": 3,
};
push(obj, 4);
```
#### 3. 函数节流
函数被频繁调用的场景
- window.onresize
- mousemove 拖拽
- 上传进度

// TODO 看不太懂
```
var throttle = function(fn, interval){
    var _self = fn,
        timer,
        firstTime = true;
    return function(){
        var args = arguments,
            _me = this;
        if (firstTime) {    // 如果是第一次执行则不需要延时
            _self.apply(_me, args);
            return firstTime = false;
        }
        if (timer) {        // 如果定时器还在，说明上一次的延迟还没有结束
            return false;
        }
        timer = setTimeout(function(){
            clearTimeout(timer);
            timer = null;
            _self.apply(_me, args);
        }, interval || 500);
    };
}
window.onresize = throttle(function(){
    console.log(1);
}, 500)
```

#### 4. 分时函数
一次性执行完太过阻塞，影响页面性能。下面的例子是在页面中创建大量DOM节点，每200秒创建8个。
```
var timeChunk = function(ary, fn, count){   // ary创建节点需要的数据 fn创建节点的逻辑 count每一次创建的节点数
    var obj, t;
    var len = ary.length;
    var start = function(){
        for (var i = 0; i < Math.min(count | 1, ary.length); i++) {
            var obj = ary.shift();
            fn(obj);
        }
    };
    return function(){
        t = setInterval(function(){
            if (ary.length === 0) {     // 如果全部的节点都已经创建好
                return clearInterval(t);
            }
            start();
        }, 200);
    }
};

var ary = [];
for (var i = 0; i <= 1000; i++) {
    ary.push(i);
}
var renderFriendList = timeChunk(ary, function(n){
    var div = document.createElement('div');
    div.innerHTML = n;
    document.body.appendChild(div);
}, 8);
renderFriendList();
```
#### 5. 惰性加载函数
实现各个浏览器能够通用的addEvent函数
- 需要使用的时候创建
- 只创建一次，只写一次条件分支语句


第一次进入条件分支语句 函数内部会重写addEvent，重写之后的函数就是当前浏览器支持的事件绑定函数（real-addEvent）。再一次进入addEvent(real-addEvent)的时候里面就不包含条件分支语句了。
```
<html>
    <body>
        <div id="div1">点击绑定事件</div>
    </body>
    <script type="text/javascript">
        var addEvent = function(elem, type, handler){
            if (window.attachEvent) {
                addEvent = function(elem, type, handler){
                    elem.attachEvent('on' + type, handler);
                };
            } else if (window.addEventListener) {
                addEvent = function(elem, type, handler){
                    elem.addEventListener(type, handler, false);
                };
            }
            addEvent(elem, type, handler);
        };

        var div = document.getElementById('div1');
        addEvent(div, 'click', function(){
            console.log(1);
        });
        addEvent(div, 'click', function(){
            console.log(2);
        });
    </script>
</html>
```

#### <炖的小结>
闭包还是需要有实际使用才能理解生命周期这回事，以及高阶。都是重要的概念，温故而知新。希望一个星期后可以再回来看一遍。20170308