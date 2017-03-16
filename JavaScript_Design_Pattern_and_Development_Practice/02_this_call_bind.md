## 1. this
### 1.1 this的指向
1. 作为对象的方法调用
    ```
    var obj = {
        a: 1,
        getA: function(){
            console.log(this === obj);  // true, this指向对象
            console.log(this.a);    // 1
        },
    };
    ```

2. 作为普通函数调用

    函数不作为对象的属性时是普通函数，this指向全局对象window。
    ```
    <html>
        <body>
            <div id="div1">我是一个div</div>
        </body>
        <script>
            window.id = 'window';
            document.getElementById( 'div1' ).onclick = function(){
                alert ( this.id ); // 'div1'
                var callback = function(){
                    alert ( this.id ); // 'window', callback函数是普通函数 指向全局变量
                }
                callback();

                var that = this;    // 用that变量保留div1节点
                var callback1 = function(){
                    alert ( that.id ); // 'div1'
                }
                callback1();
            };
        </script>
    </html>
    ```
    在ES5的++strict++模式下，普通函数中的this不再指向全局对象，而是++undefined++

3. 构造器调用
    - 大部分js函数可以当作构造器，和普通函数看起来一样。当用new调用函数时总是会返回一个对象，通常构造器里的this指向返回的这个对象。
        ```
        var MyClass = function(){
            this.name = 'sven';
        };
        var obj = new MyClass();
        alert ( obj.name ); // sven, this指向obj
        ```
    - 如果构造器显式返回一个object类型的对象
        ```
        var MyClass = function(){
            this.name = 'sven';
            return {    // 显式地返回一个对象
                name: 'anne'
            }
        };
        var obj = new MyClass();    // obj = {name: 'anne'}
        alert ( obj.name ); // anne
        ```
    - 只要构造器不显式返回一个object对象，就不会有上面的情况
        ```
        var MyClass = function(){
            this.name = 'sven'
            return 'anne'; // 返回string 类型
        };
        var obj = new MyClass();
        alert ( obj.name ); // sven
        ```

4. Function.prototype.call 或Function.prototype.apply 调用

    动态改变传入函数的this
    ```
    var obj1 = {
        name: 'sven',
        getName: function(){
            return this.name;
        }
    };
    var obj2 = {
        name: 'anne'
    };
    console.log( obj1.getName() ); // sven
    console.log( obj1.getName.call( obj2 ) ); // anne,this变成了obj1
    ```
### 1.2 丢失的this
    ```
    // example1
    var obj = {
        myName: 'sven',
        getName: function(){
            return this.myName;
        }
    };
    var getName2 = obj.getName;
    console.log( getName2() ); // undefined, getName2是普通函数, this指向window
    ```

## 2. call、apply
### 2.1 区别
- 作用一样
- 传参形式不一样
1. apply [arg1, arg2]
    - arg1: 指定函数体内this指向
    - arg2: 带下标的集合（数组、类数组），集合中的元素作为参数传给被调用的函数
2. call [arg1, [arg2, ...]]
    - arg1: 指定函数体内this指向
    - 从第二个参数开始依次传入函数
```
var func = function( a, b, c ){
    alert ( [ a, b, c ] ); // 输出 [ 1, 2, 3 ]
};
func.apply( null, [ 1, 2, 3 ] );    // 函数体内this指向默认宿主对象，浏览器中是window，严格模式下是null
func.call( null, 1, 2, 3 );
```
### 2.2 用途
1. 改变this指向
    ```
    // 简写document.getElementById方法, getElementById方法的内部实现上有用到this指向document
    document.getElementById = (function(func){
        retuen function(){
            return func.apply(document, arguments);
        }
    })(document.getElementById);
    var getId = document.getElementById;
    var div = getId('div1');
    console.log(div.id);    // div1
    ```
2. Function.prototype.bind
    大部分高级浏览器都实现了Function.prototype.bind指定函数内部的this指向。可模拟原生的方法：
    ```
    Function.prototype.bind = function(context){
        var self = this;    // 保留原函数
        return function(){  // 返回一个新函数
            return self.apply(context, arguments);  // 执行新的函数的时候，会把之前传入的context当做新函数体内的this
        }
    };
    var obj = {
        name: 'Aubrey'
    };
    var func = function(){
        alert(this.name);   // 输出：Aubrey
    }.bind(obj);    // this指向obj
    func();
    ```
    可以在func函数中预填一些参数
    ```
    Function.prototype.bind = function(){
        var self = this,
            context = [].shift.call(arguments), // 绑定的上下文
            args = [].slice.call(arguments),    // arguments 现在是绑定的第二个及后面的参数
        return function(){
            return self.apply(context, [].concat.call(args, [].slice.call(arguments)));
            // 执行新的函数的时候，会把之前传入的context当做新函数体内的this
            // 组合预先填入的参数和调用func时传入的函数，作为新函数的参数
        }
    };
    var obj = {
        name: 'Aubrey'
    };
    var func = function(a, b, c, d){
        alert(this.name);   // 输出：Aubrey
        alert([a, b, c, d]);    // 输出： [1, 2, 3, 4]
    }.bind(obj, 1, 2);
    func(3, 4);
    ```
3. 借用其他对象的方法
    1. 借用构造函数，实现类似继承的效果
    ```
    var A = function( name ){
        this.name = name;
    };
    var B = function(){
        A.apply( this, arguments ); // 类似于B继承A
    };
    B.prototype.getName = function(){
        return this.name;
    };
    var b = new B( 'sven' );
    console.log( b.getName() ); // 输出： 'sven'
    ```
    2. 操作arguments
    ```
    (function(){
        Array.prototype.push.call( arguments, 3 );
        console.log ( arguments ); // 输出[1,2,3]
    })( 1, 2 );
    ```