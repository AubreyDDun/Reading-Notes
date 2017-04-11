又叫观察者模式。定义对象间的**一对多**的依赖关系。当一个对象发生状态改变，所有依赖它的对象都会得到通知

## 1. 作用
1. 异步编程中可以代替传递callback func。如订阅ajax请求的error、success，动画每一个关键帧结束后做个操作。在异步运行期间，只需要订阅我们感兴趣的事件发生点。
2. 取代对象之间硬编码的通知机制，一个对象不再显式调用另一对象的接口。发布-订阅模式让对象松耦合地联系，只要约定的事件名没有变，订阅者或者发布者内部改变都不影响之间的通信。


DOM节点绑定事件其实就是发布-订阅模式

## 2. 自定义事件
实现发布-订阅模式的步骤：
1. 指定发布者
2. 给发布者添加缓存列表，存放订阅者的cb func
3. 发布消息时遍历缓存列表，依次触发里面存放的cb func

## 3. 发布-订阅模式的通用实现
```
var event = {
    clientList: [],     // 缓存列表
    // 订阅者订阅
    listen: function(key, fn){
        if (!this.clientList[key]) {
            this.clientList[key] = [];
        }
        this.clientList[key].push(fn);
    },
    // 发布
    trigger: function(){
        var key = Array.prototype.shift.call(arguments),
            fns = this.clientList[key];
        if (!fns || fns.length === 0) { // 如果没有绑定对应的信息
            return false;
        }
        for (var i = 0, fn; fn = fns[i++]) {
            fn.apply(this, arguments);  // 依次触发订阅者的回调
        }
    },
    // 取消订阅
    remove: function(key, fn){
        var fns = clientList[key];
        if (!fns) {
            return false;
        }
        if (!fn) {  // 如果没有传入具体的回调，则取消key对应的所有订阅
            fns && (fns.length = 0);
        } else {
            for (var l = fns.length-1; l > 0; l--) {
                var _fn = fns[l];
                if (_fn === fn) {
                    fns.splice(l, 1);
                }
            }
        }
    },
};

var installEvent = function(obj){   // 可以给所有对象安装发布-订阅功能
    for (var i in event) {
        obj[i] = event[i];
    }
};

// 用在售楼处的例子上面
var salesOffices = {};
installEvent(salesOffices);
salesOffices.listen('squareMeter88', fn1 = function(price){
    console.log('Hi, Jane. Price is : ' + price);
});
salesOffices.listen('squareMeter88', fn2 = function(price){
    console.log('Hi, Aubrey. Price is : ' + price);
});
salesOffices.listen('squareMeter110', fn3 = function(price){
    console.log('Hi, Mac. Price is ' + price);
});
salesOffices.trigger('squareMeter88', 2000000); // Hi,Jane.Price is 2000000. ||  Hi,Aubrey.Price is 2000000.
salesOffices.trigger('squareMeter110', 3000000);// Hi,Mac.Price is 3000000.

salesOffices.remove('squareMeter88', fn1);  // 删除Jane的订阅
salesOffices.trigger('squareMeter88', 2000000); // Hi,Aubrey.Price is 2000000.
```

## 4. 真实的例子——网站登录
header、nav、消息列表、购物车都需要用户的登录信息——ajax异步请求——不知道什么时候会返回。通常我们会使用回调：
```
login.succ(function(data){
    header.setAvatar( data.avatar); // 设置 header 模块的头像
    nav.setAvatar( data.avatar );   // 设置导航模块的头像
    message.refresh();  // 刷新消息列表
    cart.refresh();     // 刷新购物车列表
});
```
问题：
1. 使用登录信息的模块和登录产生了强耦合，以后还有模块（比如收货地址列表）要使用登录信息就要修改succ函数
2. 模块不能随意修改接口的名字，比如header.setAdvatar不能改成header.setTouXiang，head也不能修改为Advatar
3. 这种耦合使程序变得僵硬。**针对具体实现编程是不被赞同的**。

- 使用发布-订阅模式，需要用户信息的模块自行订阅登录成功的消息
- 登录模块只需要在登录成功时发布登录成功的消息
- 登录模块不关心业务方模块需要做什么事，内部实现细节如何

```
// login
pro.__getData = function(){
    var that = this;
    request('/xxx.com?login', {
        method: 'get',
        onload: function(result){
            that.$emit('loginSucc', result);
        }
    });
}
```
各模块监听登录成功的消息
```
var header = (function(){
    login.$on('loginSucc', function(data){
        header.setAdvatar(data.advatar);
    });
    return {
        setAdvatar: function(data){
            console.log('设置header模块的头像');
        }
    }
})();
var nav = (function(){
    login.$on('loginSucc', function(data){
        nav.setAdvatar(data.advatar);
    });
    return {
        setAdvatar: function(data){
            console.log('设置nav模块的头像');
        }
    }
})();
```
方法名可以自由更改，有新的模块需要用户信息可自由添加，不需要对登录模块做修改。

## 5. 全局的发布-订阅对象
在售楼处和登录的例子中，存在两个问题：
1. 我们为发布信息的模块都添加了listen和trigger功能，以及一个缓存列表clientList
2. 购房者和售楼处存在耦合。购房者必须知道售楼处的名字才能订阅楼盘信息；如果购房者想从售楼处B获取信息还得订阅售楼处B

但现实中我们通常会通过中介。房产公司的房源信息交由中介公司发布，购房者只需去中介就可以了解所有楼盘的信息。订阅者和发布者都必须知道这个中介，就可以顺利接收和发布信息。

创建全局的Event对象:
```
var Event = (function(){
    var clientList = {},
        listen,
        trigger,
        remove;

    listen = function(key, fn){
        if(!clientList[key]){
            clientList[key] = [];
        }
        clientList[key].push(fn);
    };

    trigger = function(){
        var key = Array.prototype.shift.call(arguments),
            fns = clientList[key];
            if(!fns || fns.length === 0){
                return false;
            }
            for(var i = 0, fn; fn = fns[i++];){
                fn.apply(this, arguments);
            }
    };

    remove = funciton(key, fn){
        var fns = clientList[key];
        if(!fns){
            return false;
        }
        if(!fn){
            fns && (fns.length = 0);
        } else{
            for (var l = fns.length-1; l>=0; l--){
                var _fn = fns[l];
                if(_fn === fn){
                    fns.splice(l, 1);
                }
            }
        }
    };

    return {
        listen: listen,
        trigger: trigger,
        remove: remove
    }
})();

Event.listen('squareMeter88', funtion(price){   // 订阅
    console.log('面积88平方米，价格：' + price);
});

Event:trigger('squareMeter88', 2000000);        // 发布
```
## 6. 模块间通信
基于全局的Event对象的发布-订阅模式，在两个封装良好的模块间通信，这两个模块可以完全不知道对方的存在。

比如有两个模块，A模块里有一个按钮，点击按钮B模块会显示点击总次数。AB模块保持封装的前提下使用全局发布-订阅模式完成这样的功能
```
<DOCTYPE html>
<html>
<body>
    <button id='btn'>点击</button>
    <div id='show'></div>
</body>

<script>
    var a = (function(){
        var count = 0;
        var button = document.getElementById('btn');
        button.click = function(){
            Event.trigger('add', count++);
        };
    });

    var b = (function(){
        var div = document.getElementById('show');
        Event.listen('add', function(count){
            div.innerHTML = count;
        });
    })();
</script>
</html>
```

全局发布-订阅模式通信的一个缺点是，模块之间的联系被隐藏起来，最终不知道消息来自于哪个模块，或者消息将会流向哪个模块，会给维护带来麻烦。
## 7. 还有两个功能是必须考虑的
### 必须先订阅再发布吗
前面讲到的都是先订阅，再发布。但现实中需要实现先发布再订阅，比如未读的离线信息。离线信息被保存起来，下一次接收人登录上线之后可重新收到这条消息。

我们需要建立一个存放离线事件的堆栈，事件发布，还没有订阅者订阅时，将发布事件的动作包裹在一个函数里，包装函数存入堆栈，等到有订阅者的时候，遍历堆栈并且依次执行这些包装函数（是不是也是一种发布订阅呢？有了订阅者，再依次触发离线事件），重新发布里面的事件。

离线的事件生命周期只有一次，所以重新发布的动作只能有一次。
### 全局事件的命名冲突
clientList存放消息名和回调函数，难免还有命名冲突的问题。使用命名空间解决。

### 代码实现
```
var Event = (function(){
    var Event,
        _default = 'default';

    Event = function(){
        var _listen,    // 订阅
            _trigger,   // 发布
            _remove,    // 取消订阅
            _shift = Array.prototype.shift,
            _unshift = Array.prototype.unshift,
            namespaceCache = {},    // 命名空间对象
            _create,    // 创建命名空间
            each = function(ary, fn){   // 执行堆栈中的函数
                var ret;
                for (var i = 0, l = ary.length; i < l; i++) {
                    var n = ary[i];
                    ret = fn.call(n, i, n);
                }
                return ret;
            };

        _listen = function(key, fn, cache){
            if (!cache[key]) {
                cache[key] = [];
            }
            cache[key].push(fn);
        };

        _remove = function(key, cache, fn){
            if (cache[key]) {
                if (fn) {
                    for (var i = cache[key].length; i >= 0; i--) {
                        if (cache[key[i]] === fn) {
                            cache[key].splice(i, 1);
                        }
                    }
                } else {
                    cache[key] = [];
                }
            }
        };

        _trigger = function(){
            var cache = _shift.call(arguments), // 命名空间
                key = _shift.call(arguments),   // 事件
                args = arguments,
                _self = this,
                ret,
                stack = cache[key];

            if (!stack || !stack.length) {
                return;
            }

            return each(stack, function(){  // 依次触发消息堆栈的包裹消息的函数
                return this.apply(_self, args);
            });
        };

        _create = function(namespace){  // 创建命名空间
            var namespace = namespace || _default;
            var cache = {},
                offlineStack = [],  // 包裹离线消息函数的堆栈
                ret = {
                    listen: function(key, fn, last){
                        _listen(key, fn, cache);
                        if (offlineStack === null) {
                            return;
                        }
                        if (last === 'last') {
                            offlineStack.length && offlineStack.pop()();
                        } else {
                            each(offlineStack, function(){  // 触发离线消息堆栈
                                this();
                            });
                        }

                        offlineStack = null;
                    },
                    one: function(key, fn, last){
                        _remove(key, cache);
                        this.listen(key, fn, last); // TODO 不懂
                    },
                    remove: function(key, fn){
                        _remove(key, cache, fn);
                    },
                    trigger: function(){
                        var fn,
                            args,
                            _self = this;

                        _unshift.call(arguments, cache);
                        args = arguments;
                        fn = function(){
                            return _trigger.apply(_self, args);
                        };

                        if (offlineStack) {
                            return offlineStack.push(fn);
                        }
                        return fn();
                    }
                };
            // 是否有命名空间 默认'default'
            return namespace ?
                (namespaceCache[namespace] ? namespaceCache[namespace] :
                    namespaceCache[namespace] = ret)
                        : ret;
        };

        return {
            create: _create,
            one: function(key, fn, last){
                var event = this.create();
                event.one(key, fn, last);
            },
            remove: function(key, fn){
                var event = this.create();
                event.remove(key, fn);
            },
            listen: function(key, fn, last){
                var event = this.create();
                event.listen(key, fn, last);
            },
            trigger: function(){
                var event = this.create();
                event.trigger.apply(this.arguments);
            }
        }
    }();
    return Event;
})();
```
## 8. 炖的小结
时间上和对象间的解耦是发布-订阅者的明显优点。对于Event的实现尚有不懂的地方。对于处理异步请求或者操作，可以使用这种模式，解耦事件的触发者和关心变化的模块。