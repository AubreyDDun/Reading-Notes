**保证一个类只有一个实例，并提供一个访问它的全局访问点**

## 1. 实现单例模式
用一个**变量标记**当前是否已为某个类创建过对象，如果是，在下一次获取该类实例的时候直接**返回之前创建的对象**
```
var Singleton = function(name){
    this.name = name;
};
Singleton.prototype.getName = function(){
    alert(this.name);
};
Singleton.getInstance = (function(){
    var instance = null;
    return function(name){
        if (!this.instance) {
            this.instance = new Singleton(name);
        }
        return this.instance;    
    }
})();
var a = Singleton.getInstance('a');
var b = Singleton.getInstance('b');
console.log(a === b);   // true
```
## 2. 透明的单例模式
上面的代码中，用户需要知道这是一个单例，用getInstance获取对象而不是new一个。用户需要一个透明的单例类，使用这个类跟使用普通类一样

## 3. 用代理实现单例模式
普通类+代理管理单例
```
var CreateDiv = function(html){
    this.html = html;
    this.init();
};
CreateDiv.prototype.init = function(){
    var div = document.createElement('div');
    div.innerHTML = this.html;
    document.appendChild(div);
};
var ProxySingletonCreateDiv = (function(){
    var instance = null;
    return function(html){
        if (!instance) {
            return instance = new CreateDiv(html);
        }
        return instance;
    }
})();
var a = new ProxySingletonCreateDiv('a');
var b = new ProxySingletonCreateDiv('b');
```

## 4. JS中的单例模式
全局对象符合单例的两个定义——1.独一无二；2.全局可访问。但是全局变量会造成命名空间污染。应该尽可能避免使用全局变量。

1. 使用命名空间
    最简单的方法是用对象字面量
    ```
    var namespace1 = {
        a: function(){
            alert(1);
        },
        b: function(){
            alert(2);
        },
    };
    ```
    还可以动态地创建NS
    ```
    var myAPP = {};
    myAPP.namespace = function(name){
        var parts = name.split('.');
        var current = myAPP;
        for (var i in parts) {
            if (!current[parts[i]]) {
                current[parts[i]] = {};
            }
            current = current[parts[i]];
        }
    };

    MyApp.namespace( 'event' );
    MyApp.namespace( 'dom.style' );
    // 上述代码等价于：
    var MyApp = {
        event: {},
        dom: {
            style: {}
        }
    };
    ```
2. 使用闭包封装私有变量
    只暴露API
    ```
    var user = (function(){
        var __name = 'Aubrey';
        var __hobby = 'pool';
        return {
            getUserInfo: function(){
                return __name + "'s hobby is " + __hobby;
            }
        }
    })();
    ```
## 5. 通用的惰性单例
- **需要的时候才创建**
- **创建和管理的逻辑分离**

如果我们需要创建其他唯一的对象，比如iframe、script，就要重复实现单例。应该提取不变的部分——比如用一个变量标志是否创建过对象，将管理单例的逻辑实现出来
```
var getSingle = function(fn){
    var result;
    return function(){
        return result || (result = fn.apply(this, arguments));
    }
};
```
1. 比如要实现创建唯一的登录弹窗
    ```
    createLoginLayer = function(){
        ...
    };
    createSingleLoginLayer = getSingle(createLoginLayer);
    ```
2. 渲染完一个列表之后对其绑定click事件，我们希望的是第一次渲染出来的时候绑定事件。下面虽然render了三次，但是只输出了一次'bind'
    ```
    var bindEvent = getSingle(function(){
        document.getElementById('list').onclick = function(){
            console('bind');
        };
        return true;
    });

    var render = function(){
        console.log('renderring list');
        bindEvent();
    };
    render();
    render();
    render();
    ```
## 6. 炖的小结
- 单例保证了对象的实例的唯一性，一旦创建出实例，全局引用的都是这一份实例。
- 应该把创建对象和管理单例的逻辑分开，使得创建普通实例对用户透明，管理单例逻辑可复用。