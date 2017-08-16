#### 背景
前段时间刚完成《你不知道的JavaScript》上的总结，主要分享了关于作用域和闭包的相关概念。那么接下来我将为大家带来本书的第二部分：this和对象原型部分的分享。这部分也是我们在开发项目代码的过程中经常会遇到的问题，了解了this和对象原型对于我们深入理解js的原理有很大的帮助。


###### this和对象原型
1.关于this    
前一部分谈到了javascript中的词法作用域，javascript的词法作用域是在声明时就已经确定的，是一种静态的概念。相对应的this算是javascript中的动态概念，同时this也是javascript中比较复杂的机制。那大家肯定会想我们为什么要使用this。
this提供了一种优雅的方式来隐式的“传递”一个对象的引用，可以方便我们简洁的设计API，特别是当结构越来越复杂时，这种优势就能体现出来了。
对于this的指向有常见的两种误区。一种是认为this指向自身；另一种是认为this指向函数的作用域。下面我们通过例子一一说明：
A.this指向自身
function foo(num) {
    console.log(num);
    this.count++;
}
foo.count = 0;
for(var i=0; i < 8; i++) {
    if(i > 3) {
        foo(i);
    }
}
//4,5,6,7
console.log(foo.count); //0
这就说明了其中的道理；同时这里++的count会在全局创建这么一个变量。

B.this指向函数作用域
这种情况在某些情况下是正确的。需要说明的是，作用域跟对象类似，其中的变量就像对象的属性，但是作用域对象无法被js代码引用，它存在于js引擎内部。
function foo(){
    var a = 2;
    this.bar();
}
function bar() { console.log(this.a);}
foo();  //ReferenceError: a is not defined

那this到底是什么呢？this只跟函数的调用位置有关；this的指向取决于函数在哪里被调用。

2.this全面解析  
this的指向只跟调用位置有关，调用位置就在当前正在执行的函数的前一个调用中，跟调用栈密切相关，等于调用栈中的倒数第二个元素。我们可以通过js调试工具设置断点或是在代码中插入debugger来获取函数的调用栈。
this指向的最终对象，跟调用位置以及应用的绑定规则有关。this的绑定规则有四种,分别是：默认绑定、隐式绑定、显示绑定以及new绑定，我们来一一讲解。
A.默认绑定
最常见的绑定，独立函数调用时应用默认绑定。例如：
function foo() {console.log(this.a);}
var a = 2;
foo();  //2

默认绑定中非严格模式下，this会默认绑定到全局对象；严格模式下绑定到undefined；其中所谓的严格模式，是指函数体处于严格模式，而非调用位置；否则还是绑定到全局对象。
【函数体严格】
function foo() {
    "use strict";
    console.log(this.a);
}
var a = 2;
foo();  //TypeError: this is undefined
【调用位置严格】
function foo() {console.log(this.a);}
var a = 2;
(function(){
    "use strict";
    foo();  //2
})();

B.隐式绑定
当函数的调用存在上下文对象，this将会被绑定到这个上下文对象。例：
function foo() {console.log(this.a);}
var obj = {a : 2, foo: foo};
obj.foo();  //2

隐式绑定存在一个问题，被隐式绑定的函数会丢失绑定对象，从而使用默认绑定规则。经常出现在一下两种情况中：
【函数引用传递给其他变量】
function foo() {console.log(this.a);}
var a = "Mistake";
var obj = {a : 2, foo: foo};
var bar = obj.foo;
bar();  //Mistake

【函数引用当做参数传入回调函数】
function foo(){
    console.log(this.a);
}
var obj = {a: 2, foo: foo};
var a = "It is a mistake";
setTimeout(obj.foo, 1000);  //It is a mistake

C.显示绑定
在隐式绑定中有时候this的改变意想不到，这时候我们可以使用apply和call来显示的绑定this的指向。例：
function() {console.log(this.a);}
var obj = {a: 2};
foo.call(obj);  //2

显示绑定有两种应用的场景：硬绑定和API调用的“上下文”；其中硬绑定可以用来创建一个包裹函数，负责接收参数并返回值；并且可以创建一个可以重复使用的辅助函数，类似bind。API调用的“上下文”可以理解为许多第三方库中函数+许多js和宿主环境内置的函数，内部实现了显示绑定的功能，确保使用指定的this。例：
function foo(el) {console.log(el, this.id)};
var obj = {id: "example"};
[1,2,3].forEach(foo, obj);  //1 example 2 example 3 example

D.new绑定
使用new进行绑定时会创建新的对象；使用起来跟面向类的语言一样，其实javascript中创建新对象时调用的构造函数就是普通函数，不存在“构造函数”，它只是对函数的“构造调用”。例：
function foo(a) {
    this.a = a;
}
var bar = new foo(2);
console.log(bar.a);  //2

使用new对函数进行“构造调用”的过程
a.创建（or构造）一个全新对象;
b.对新对象执行[[PROTOTYPE]]连接;
c.将新对象绑定到函数调用的this;
d.返回新对象

当某个函数调用应用了这四种规则中的多条，那么优先级：
new绑定 > 限时绑定 > 隐式绑定 > 默认绑定
通过确定函数的调用位置，判断应用的绑定规则基本可以确定this的指向问题。但总会有一些意外的情况：
【情景一】被忽略的this
当我们将null或undefined作为this绑定对象，传入call、apply、bind，实际使用默认绑定。
有人可能会问为啥要传null或undefined。当我们将null传入apply以后，可以用来展开一个数组，并作为参数传入一个函数。将null传给bind以后，可以对参数进行柯里化，用来预先设置一些参数。例：
function foo(a,b){console.log(a,b);}
foo.apply(null,[2,3]);  //展开数组，a=2,b=3
var bar = foo.bind(null,2);
bar(3);  //2,3

【情景二】简介引用
有意、无意的创建一个函数的“间接引用”，这个函数会使用默认绑定规则。具体跟上述隐式绑定规则绑定丢失情况类似。

【情景三】软绑定
硬绑定太硬了，降低了函数的灵活性。为了达到给默认绑定指定一个除全局对象和undefined之外的值的效果，引出软绑定。它首先会检查调用时的this，如果this绑定到全局对象或undefined，那就将指定的obj绑定到this；否则不会修改this的绑定。

针对上述对于this的理解，看出this是一个动态的概念，这种动态绑定的特性导致this的指向难以理解。ES6中引入了箭头函数来解决这个问题。箭头函数不使用this的四种绑定规则，箭头函数的绑定无法被修改，并且箭头函数中this的指向根据外层作用域决定，箭头函数会继承外层函数调用的this绑定。通过箭头函数实现用更加常见的词法作用域来取代传统的this动态机制。让代码编写起来更加简单、舒畅。
在用react开发项目的过程中，无时无刻不在体会着this变换莫测的特性，以及引入ES6的箭头函数之后的那种从容。

3.对象  
说完this，就不得不说js中的操作主体————对象。可以通过声明形式和构造形式来定义对象。声明形式比较常用，可以一次性为对象添加多个属性；
var person = {name: 'xx', sex: 'male'};
构造形式比较繁琐，比较少见。
var person = new Object();
person.name = 'xx';

javascript中存在6中基本类型：string、number、boolean、null、undefined、object。它们本身不是对象，只是一种基本类型。但当我们在操作这些基本的类型的变量的时候，都会将其转换为对应的对象子类型。
let str = 'lnong story';
console.log(str.length);
js中内置的对象子类型就比较丰富了：String、Number、Boolean、Object、Function、Array、Date、RegExp、Error等等。
其中需要注意的是，虽然typeof(null)为"Object"，但null本身是基本类型。

接下来我们具体的谈一谈对象中的相关操作，主要关于对象中的属性名和属性值。
对象的属性名存储在对象容器内部，指向属性值。而属性值一般不会存储在对象容器内部。如果要访问对象的属性值，可以通过：属性访问or键值访问。
var person = {name: 'xx'};
person.name;  //属性访问——.操作符
person['name'];  //键值访问——[]操作符
一般我们直接使用.来获取属性值，但是[]操作符更加强大，支持任意的UTF-8/Unicode字符串，可以接受用表达式来计算属性名。
var person = 'cy';
var student = {[person]: 'male'};
console.log(student['cy']);  //male

下面分析下数组和对象。
数组也是对象，有一套更加结构化的值存储机制，一般通过下标引用，并且每个下标都是整数；当然你也可以给数组添加属性，不过不建议这么使用。如果试图向数组添加一个属性名像数字的属性，它会变成数值下标，从而修改数组内容，而不是添加了一个属性。例：
var sport = ['football','baseball'];
sport['2'] = 'basketball';
sport.length;  //3
sport[2];  //basketball

关于对象有一个很常见的操作就是复制。对象的复制行为分为浅复制和深复制；浅复制只复制对象的直接属性，而深复制不仅复制直接属性，还会进一步复制直接属性对应的属性内容，我们一般用不到深复制。
【对于JSON安全的对象】(可以被序列化为JSON字符串并可以根据字符串解析出一个结构和值完全一样的对象)来说，有一种巧妙的方法：
var targetObj = JSON.parse(JSON.stringify(sourceObj));
【对于浅复制】ES6中引入了Object.assign来实现浅复制。Object.assign复制的时候，遍历的是源目标的可枚举属性；并且复制的时候使用的是"="进行赋值的，不会复制对应属性描述符到目标对象。例：
var sourceObjFirst = {a: 1, b: 2};
var sourceObjSecond = {};
Object.defineProperty(sourceObjSecond, "c",{
    value: 3,
    writable: true,
    configurable: true,
    enumerable: false,
});
Object.defineProperty(sourceObjSecond, "d",{
    value: 4,
    writable: false,
    configurable: true,
    enumerable: true,
});
var targetObj = Object.assign({}, sourceObjFirst, sourceObjSecond);
targetObj.d = 0;
targetObj.a;  //1
targetObj.b;  //2
targetObj.c;  //undefined
targetObj.d;  //0

上面的操作涉及到对象的属性描述符，可以使用Object.defineProperty来修改和定义。属性描述符分为：数据描述符和访问描述符。
【数据描述符】中四部分内容：
value，对应属性的值；
writable：决定是否可以修改属性值；
configurable：决定属性的writable、configurable和enumerable是否可修改以及属性是否可删除；configurable修改成false为单向操作，无法撤销。
例外：
configurable为false时，writable可以从true变成false，反过来不行。
enumerable：控制相应的属性是否出现在对象的属性枚举中。比如说：for...in中、Object.assign中都只是这对可枚举属性。
涉及到的相关操作：
Object.keys————返回的是对象本身包含的、可枚举属性；
Object.getOwnPropertyNames————返回对象本身包含的所有属性，不区分是否可枚举；
propertyIsEnumerable————判断属性名是否存在于对象本身并且可枚举。

【访问描述符】跟数据描述符差不多，区别是没有value和writable，取而代之的是getter和setter，这两个都是隐藏的函数，分别用来获取属性值和设置属性值。

有时候会希望对象的属性或对象不可变，我们可以使用多种方法来实现。但是这里的不可变都是浅不可变，只会影响目标对象和它的直接属性。下面列举了四种情形下的不可变形，程度越来越严格：
A.对象常量——【writable:false】+【configurable:false】，不可修改，不可删除
B.禁止扩展——【Object.preventExtensions】，禁止添加新属性，保留已有属性
C.密封——【Object.seal】，类似于Object.preventExtensions+configurable:false，禁止添加新属性，禁止删除已有属性
D.冻结——【Object.freeze】，类似于Object.seal+writable:false，禁止添加新属性，禁止删除、修改已有属性。*最高级别*

那如果要访问或操作对象的某个属性呢，分别会用到[[Get]]和[[Set]]操作。
当通过[[Put]]访问属性值时，首先在对象本身查找，找到即返回；如果没有，会遍历[[Prototype]]原型链进行查找，直到找到为止。如果都没有，返回undefined。
当通过[[Set]]设置对象属性时，情况会比较复杂点。这里假设对象的属性已存在，大致过程如下：
1.属性是访问描述符并且有setter，如果是就调用setter。
2.属性是数据描述符并且writable为false，调用失败（非严格），TypeError（严格）。
3.都不是上述情况，直接设为属性的值。

在操作对象属性的时候，会存在一些问题，例：
var obj = {a: undefined};
obj.a;  //undefined
obj.b;  //undefined
如何判断属性值就是undefined还是不存在呢?
可以通过in和hasOwnProperty来判断；
in————检查属性是否在对象和原型链中。不管属性是否可枚举！
hasOwnProperty——只检查属性是否在对象中。不管属性是否可枚举！

对象操作中，可以使用for...in来遍历对象中的可枚举属性列表，也可以使用for...of直接遍历属性值。
【数组中】一方面可以通过for循环+下标来间接获取值，当然也可以使用for...of来直接获取值。for...of，首先向被访问数组请求一个迭代器对象，然后调用迭代器对象的next方法来遍历所有值。例：
var arr = [11,22];
for(var val of arr) {
    console.log(val);
}
输出11,22
类似：
var arr = [11,22];
var it = arr[Symbol.iterator]();
it.next();  //{value: 11, done: false}
it.next();  //{value: 22, done: false}
it.next();  //{done: true}
【对象中】普通对象没有内置的返回迭代器对象的函数@@iterator，无法自动完成for...of的遍历。可以自定义对象的@@iterator
var obj = {a: 2, b: 3};
Object.defineProperty(obj, Symbol.iterator, {
    enumerable: false,
    writable: false,
    configurable: false,
    value: function() {
        var tmp = this;
        var index = 0;
        var arr = Object.keys(tmp);
        return {
            next: function() {
                return {
                    value: tmp[arr[index++]],
                    done: (index > arr.length)
                };
            }
        };
    }
});
//手动
var it = obj[Symbol.iterator]();
it.next();  //{value: 2, done: false}
it.next();  //{value: 3, done: false}
it.next();  //{value: undefined,done: true}
//用for...of遍历
for(var val of obj) {
    console.log(val);
}
输出2,3

for...of加上自定义的迭代对象，可以组合成非常强大的对象操作工具。




