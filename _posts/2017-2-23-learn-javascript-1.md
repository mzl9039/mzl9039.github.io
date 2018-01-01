---
layout:     post
title:      "学习javascript"
subtitle:   "Hello javascript"
date:       2017-2-23 08:38:00
author:     mzl
catalog:    true
tags:
    - javascript
    - 入门
    - 廖雪峰
---

{:toc}
# 基础语法学习
## 快速入门
1. 在见面插入js的方法
    1. 网页的任何地方，一般在`<head>`标签中，并用`<script></script>`括起来
    2. 代码放在以 js 后缀的文件中，用如下方式引用：
    ``` javascript
    <script src="/static/js/jsfile.js"></script>
    ```
    有些 script 标签时包含 `type` 属性，这是不需要的，因为默认的 `type` 就是javascript
    当同一个页面引用多个 js 文件时，浏览器会按顺序执行
    这种方式更加推荐，更利用维护和代码重用
2. 编写js不要用word和写字本，而是notepad++/sublime等文件编辑器
3. 调试：建议使用Chrome的开发者工具
安装好chrome后，按 ctrl+shift+i 快捷键调出开发者工具(Developer Tools),选择Console，即可进行测试自己的代码
也可以在Sources下，进行断点/单步执行等

### 基本语法
1. 语句以 `;` 结束，语句块为 `{...}`, 虽然js不强制要求添加 `;` ,但我们写规范代码时要求这样做。
    ``` javascript
    var x = 1;      // 完整的赋值语句
    ”hello world!“; // 也是完整的语句
    var x = 1; var y = 1;   // 不建议一行写多个语句
    if (2 > 1) {    // 语句块
        x = 1;
        y = 2;
        z = 3;
    }
    // 注释符号有两种，分别为 //(单行注释) 和 /* ... */（多行注释）
    ```
2. javascript严格区分大小写

### 数据类型和变量
1. 数据类型
javascript的数据类型有：
    1. Number
    不区分浮点数和整点数，如下：
        ``` javascript
        123;        // 整数
        0.456;      // 浮点数
        1.23e3;     // 科学计数法
        -99;        // 负数
        NaN;        // Not a Number
        Infinity;   // 无限大，当数值超过javascript所能表示的最大值时，表示为Infinity
        0xff00;     // 十六进制，使用 0x 前缀和 0-9 a-f 表示
        ```
    四则运算与数学一致：
        ``` javascript
        1 + 2;          // 加号
        (3 - 1) * 5 / 2;// 减号，乘号，除号
        2 / 0；         // Infinity
        0 / 0;          // NaN
        10 % 3;         // 1, % 求余符号
        10.5 % 3;       // 1.5
        ```
    2. 字符串
    3. 布尔值
    true 和 false 两种，也可以通过布尔运算结果表示
        ``` javascript
        // 基本表示
        true; // 这是一个true值
        false; // 这是一个false值
        2 > 1; // 这是一个true值
        2 >= 3; // 这是一个false值
        // && 与运算，一假为假
        true && true; // 这个&&语句计算结果为true
        true && false; // 这个&&语句计算结果为false
        // || 或运算，一真为真
        false || false; // 这个||语句计算结果为false
        true || false; // 这个||语句计算结果为true
        // ！ 非运算，真假相反
        ! true; // 结果为false
        ! false; // 结果为true
        ```
    4. 比较运算符
    javascript 允许任意类型数据做比较，但 javascript 存在两种比较运算符：`==` 和 `===`:
    `==` 会自动转换数据类型再比较，很多时候会出现非常诡异的比较结果
    `===` 不会自动转换数据类型，如果数据类型不一致，则返回 false
    因此建议比较时， **不要**使用 `==`,始终坚持使用 `===` 进行比较
    一些特别说明的比较结果：
        ``` javascript
        NaN === NaN;    // false, NaN与其它所有值比较都为false，包括它自己
        isNaN(NaN);     // true, 唯一能判断 NaN 的方法是通过 isNaN() 函数
        1/3 === (1-2/3);  // false，注意浮点数比较，浮点数运算时会产生误差，因此比较两个浮点数时，只能计算它们之间的绝对值，看是否小于某个阈值
        Math.abs(1/3 - (1-2/3)) < 0.0000001;    // true
        ```
    5. null和undefined
    `null` 表示‘空’值，它与 `0` 以及 空字符串`''` 不同，`0`是一个数值，`''`是一个长度为0的字符串，而`null`表示为空
    `undefined` 表示**‘未定义’**
    大多数我们使用 `null`,只有在判断函数参数是否传递时，才使用 `undefined`
    6. 数组
    数组是一组按顺序排列的集合，集合中的每个值称为元素，数组中可以包含任意类型的数据,如：
        ``` javascript
        [1, 2, 3.14, 'Hello', null, true];
        ```
    还可以用 `Array()` 函数创建数组，如 ` new Array(1, 2, 3) // 创建数组[1, 2, 3]`
    建议全部使用 `[]` 创建数组
    数组元素通过索引访问，索引从0开始，如：
        ``` javascript
        var arr = [1, 2, 3.14, 'Hello', null, true];
        arr[0]; // 返回索引为0的元素，即1
        arr[5]; // 返回索引为5的元素，即true
        arr[6]; // 索引超出了范围，返回undefined
        ```
    7. 对象
    JavaScript的对象是一组由键-值组成的无序集合,其中键是字符串类型，值可以是任意类型，例如：
        ```javascript
        var person = {
            name: 'Bob',
            age: 20,
            tags: ['js', 'web', 'mobile'],
            city: 'Beijing',
            hasCar: true,
            zipcode: null
        };

        // 通过 . 获取对象的属性，如
        person.name;    // "Bob"
        person.zipcode; // null
        ```

2. 变量:
变量命名规则：略
在JavaScript中，使用等号=对变量进行赋值。可以把任意数据类型赋值给变量，同一个变量可以反复赋值，而且可以是不同类型的变量，但是要注意只能用var申明一次，例如：
    ```javascript
    var a = 123; // a的值是整数123
    a = 'ABC'; // a变为字符串
    ```

3. strict 模式
    1. JavaScript在设计之初，为了方便初学者学习，并不强制要求用var申明变量。
    这个设计错误带来了严重的后果：如果一个变量没有通过var申明就被使用，那么该变量就自动被申明为全局变量。
    2. 使用 `var` 申明的变量则不是全局变量，它的范围被限制在该变量被申明的函数体内（函数的概念将稍后讲解），同名变量在不同的函数体内互不冲突。
    3. 为了修补JavaScript这一严重设计缺陷，ECMA在后续规范中推出了strict模式，在strict模式下运行的JavaScript代码，
    强制通过var申明变量，未使用var申明变量就使用的，将导致运行错误。
    4. 启用strict模式的方法是在JavaScript代码的第一行写上：
    ```javascript
    use strict;
    ```

### 字符串
1. JavaScript的字符串用 `''` 或 `""` 括起来表示
2. 如果 `'` 本身也是字符串中的字符，则用 `""` 括起来
3. 如果字符串中即包括 `'`，也包括 `"`，则用转义字符 `\\` 来标识,如：
    ```
    'I\'m \"OK\"!';     // I'm "OK"!
    // 转义字符 \ 可以转义很多字符
    \n                  // 换行符
    \t                  // 制表符
    \\                  // 转义自身
    // ASCII 用 \x## 形式表示
    '\x41';             // 完全等同于 'A'
    // Unicode 用 \u#### 形式表示
    '\u4e2d\u6587';     // 完全等同于 '中文'
    ```

4. 多行字符串
    1. 用换行符 `\n` 来换行；
    2. 用反引号 `\`` 来表示(这是ES6中的新标准，可能不被支持)，如：
        ```javascript
        `这是一个
        多行
        字符串`;
        ```

5. 模板字符串
多个字符串连接时，可以用 `+` 连接，如：
    ```javascript
    var name = '小明';
    var age = 20;
    var message = '你好, ' + name + ', 你今年' + age + '岁了!';
    alert(message);
    ```
但很多变量需要连接时，用 `+` 比较麻烦（没看出来），ES6新标准增加了一种模板字符串（语法糖，可读性更好了）,变量用`${}`表示，如下：
    ```javascript
    var name = '小明';
    var age = 20;
    var message = `你好, ${name}, 你今年${age}岁了!`;
    alert(message);
    ```

6. 字符串操作
    1. 字符串长度：用 **属性** `length`:
        ```javascript
        var s = 'Hello, world!';
        s.length; // 13
        ```
    2. 获取字符串中的字符，用 `[]`，索引从0开始：
        ```javascript
        var s = 'Hello, world!';
        s[0]; // 'H'
        s[6]; // ' '
        s[7]; // 'w'
        s[12]; // '!'
        s[13]; // undefined 超出范围的索引不会报错，但一律返回undefined
        ```
    3. 字符串不可变，对某个字符串的某个索引赋值，不会报错，但也不会有效果，示例如下：
        ```
        var s = 'Test';
        s[0] = 'X';
        alert(s); // s仍然为'Test'
        ```
    4. toUpperCase(),将字符串内全部字符变为大写字符
    5. toLowerCase(),将字符串内全部字符变为小写字符
    6. indexOf(),搜索字符串出现的位置，如：
        ```
        var s = 'hello, world';
        s.indexOf('world'); // 返回7
        s.indexOf('World'); // 没有找到指定的子串，返回-1
        ```
    7. substring(),返回索引区间内的子串，如：
        ```
        var s = 'hello, world'
        s.substring(0, 5); // 从索引0开始到5（不包括5），返回'hello'
        s.substring(7); // 从索引7开始到结束，返回'world'
        ```
    
### 数组
1. javascript的 `Array` 可以包含任意数据类型，并通过索引访问元素
2. 数组长度
    1. 用 **属性** `length` 获得，如：
        ```
        var arr = [1, 2, 3.14, 'Hello', null, true];
        arr.length; // 6
        ```
    2. 给 `length` 赋值会引入 `Array` 大小变化，如：
        ```
        var arr = [1, 2, 3];
        arr.length;         // 3
        arr.length = 6;
        arr;                // arr变为[1, 2, 3, undefined, undefined, undefined]
        arr.length = 2;
        arr;                // arr变为[1, 2]
        ```
3. 数组索引
    1. 获取数组某一元素，用 `[]`
    2. 给数组某一元素赋值，会修改数组， **特别注意**：索引超出范围，会引入数组大小变化
        ```
        var arr = ['A', 'B', 'C'];
        arr[1] = 99;
        arr; // arr现在变为['A', 99, 'C']

        // 索引超出范围时，引起数组大小变化
        var arr = [1, 2, 3];
        arr[5] = 'x';
        arr; // arr变为[1, 2, 3, undefined, undefined, 'x']
        ```
    因此 **建议**: 大多数其他编程语言不允许直接改变数组的大小，越界访问索引会报错。然而，JavaScript的Array却不会有任何错误。
    在编写代码时，不建议直接修改Array的大小，访问索引时要确保索引不会越界。
4. indexOf(), 搜索某一元素的位置，没找到时返回 -1，如：
    ```
    var arr = [10, 20, '30', 'xyz'];
    arr.indexOf(10);    // 元素10的索引为0
    arr.indexOf(20);    // 元素20的索引为1
    arr.indexOf(30);    // 元素30没有找到，返回-1
    arr.indexOf('30');  // 元素'30'的索引为2,注意 数字30和字符串'30'是不同的元素
    ```
5. slice(),截取 `Array`的部分元素，并返回一个新的 `Array`，类似于字符串的 `substring`
`slice()` 包括起始索引，但不包括结束索引
如果不给 `slice()` 任何参数，它会从头到尾截取所有元素，可以用来复制数组
    ```
    var arr = ['A', 'B', 'C', 'D', 'E', 'F', 'G'];
    arr.slice(0, 3);    // 从索引0开始，到索引3结束，但不包括索引3: ['A', 'B', 'C']
    arr.slice(3);       // 从索引3开始到结束: ['D', 'E', 'F', 'G']
    var arr = ['A', 'B', 'C', 'D', 'E', 'F', 'G'];
    var aCopy = arr.slice();
    aCopy;              // ['A', 'B', 'C', 'D', 'E', 'F', 'G']
    aCopy === arr;      // false
    ```
6. push() 和 pop(), `push()` 向 `Array` 末尾添加**若干**元素，`pop()` 则把 `Array` 最后一个元素删除
    ```
    var arr = [1, 2];
    arr.push('A', 'B'); // 返回Array新的长度: 4
    arr; // [1, 2, 'A', 'B']
    arr.pop(); // pop()返回'B'
    arr; // [1, 2, 'A']
    arr.pop(); arr.pop(); arr.pop(); // 连续pop 3次
    arr; // []
    arr.pop(); // 空数组继续pop不会报错，而是返回undefined
    arr; // []
    ```
7. unshift() 和 shift(), `unshift()` 向 `Array` 头部添加 **若干**元素，`shift()` 则把 `Array` 第一个元素删除
    ```
    var arr = [1, 2];
    arr.unshift('A', 'B'); // 返回Array新的长度: 4
    arr; // ['A', 'B', 1, 2]
    arr.shift(); // 'A'
    arr; // ['B', 1, 2]
    arr.shift(); arr.shift(); arr.shift(); // 连续shift 3次
    arr; // []
    arr.shift(); // 空数组继续shift不会报错，而是返回undefined
    arr; // []
    ```
8. sort(), 对当前 `Array` 进行排序，会直接修改元素的位置，按默认顺序规则排序
9. reverse(), 把当前 `Array` 反转
10. splice(), 是修改 `Array` 的万能方法，可以从指定的索引开始删除若干元素，然后再从这个位置添加若干元素
    ```
    var arr = ['Microsoft', 'Apple', 'Yahoo', 'AOL', 'Excite', 'Oracle'];
    // 从索引2开始删除3个元素,然后再添加两个元素:
    arr.splice(2, 3, 'Google', 'Facebook'); // 返回删除的元素 ['Yahoo', 'AOL', 'Excite']
    arr;                                    // ['Microsoft', 'Apple', 'Google', 'Facebook', 'Oracle']

    // 只删除,不添加:
    arr.splice(2, 2);                       // ['Google', 'Facebook']
    arr;                                    // ['Microsoft', 'Apple', 'Oracle']

    // 只添加,不删除:
    arr.splice(2, 0, 'Google', 'Facebook'); // 返回[],因为没有删除任何元素
    arr;                                    // ['Microsoft', 'Apple', 'Google', 'Facebook', 'Oracle']
    ```
11. concat(), 把当前的 `Array` 和另一个 `Array` 连接在一起，并返回**新的** `Array`
实际上，`concat()` 方法可以接受任意个元素和 `Array`，并且自动把 `Array` 拆开，然后全部添加到新的 `Array`中，如：
    ```
    var arr = ['A', 'B', 'C'];
    arr.concat(1, 2, [3, 4]);   // ['A', 'B', 'C', 1, 2, 3, 4]
    ```
12. join(), 把当前的 `Array` 中的每个元素，按指定的字符串连接起来，并然后连接起来的字符串, 如果 `Array` 中的元素不是字符串，则会自动转为字符串
    ```
    var arr = ['A', 'B', 'C', 1, 2, 3];
    arr.join('-'); // 'A-B-C-1-2-3'
    ```
13. 多维数组，即 `Array` 中的某个元素又是 `Array`,如：`var arr = [[1, 2, 3], [400, 500, 600], '-'];`

### 对象
1. javascript 对象是一种无序的集合数据类型，它由若干键值对组成,用于描述现实世界中的对象，如
用 `{...}` 表示一个对象，键值对用 `xxx:xxx` 表示，不同键值对用 `,` 隔开，注意最后一个键值对不需要加 `,`，否则一些老的浏览器会报错（此处必须有IE）
    ```
    var xiaoming = {
        name: '小明',
        birth: 1990,
        school: 'No.1 Middle School',
        height: 1.70,
        weight: 65,
        score: null
    };
    ```
2. 属性定义和访问
属性用 `.` 来访问，如：
```
xiaoming.name; // '小明'
xiaoming.birth; // 1990
```
属性名必须是有效的变量名，如果包含特殊字符，属性名需要用 ‘’ 括起来，其访问也必须用 `[xxx]` ,如：
    ```
    var xiaohong = {
        name: '小红',
        'middle-school': 'No.1 Middle School'
    };
    xiaohong['middle-school'];  // 'No.1 Middle School'
    xiaohong['name'];           // '小红'
    xiaohong.name;              // '小红'
    ```
访问的属性不存在时，不会报错，而是返回 `undefined`
    ```
    var xiaoming = {
        name: '小明'
    };
    xiaoming.age; // undefined
    ```
3. 属性添加和修改
由于JavaScript的对象是动态类型，你可以自由地给一个对象添加或删除属性：
    ```
    var xiaoming = {
        name: '小明'
    };
    xiaoming.age;           // undefined

    xiaoming.age = 18;      // 新增一个age属性
    xiaoming.age;           // 18

    delete xiaoming.age;    // 删除age属性
    xiaoming.age;           // undefined

    delete xiaoming['name'];// 删除name属性
    xiaoming.name;          // undefined

    delete xiaoming.school; // 删除一个不存在的school属性也不会报错
    ```
4. 检测某个对象是否存在某个属性，可以用 `in` 操作符：
不过用 `in` 判断属性是否存在时，这个属性可能不是对象的，而这个对象继承得到的，如 `toString` 属性，
要判断一个属性是否是某对象自身拥有的，而不是继承得到的，可以使用函数 `hasOwnProperty()` 
    ```
    var xiaoming = {
        name: '小明',
        birth: 1990,
        school: 'No.1 Middle School',
        height: 1.70,
        weight: 65,
        score: null
    };
    'name' in xiaoming;     // true
    'grade' in xiaoming;    // false·

    // 因为 toString 属性定义在 object 对象中，而所有对象最终都会在原型链上指向 object，所以所有对象都会拥有 toString 属性
    'toString' in xiaoming; // true
    var xiaoming = {
        name: '小明'
    };
    xiaoming.hasOwnProperty('name'); // true
    xiaoming.hasOwnProperty('toString'); // false
    ```

### 条件判断
javascript 把 `null`、`undefined`、`0`、`NaN` 和空字符串 `‘’` 均视为 `false`，其它一概视为true

### 循环
1. for 循环
    ```
    // 通过索引遍历数组
    var arr = ['Apple', 'Google', 'Microsoft'];
    var i, x;
    for (i=0; i<arr.length; i++) {
        x = arr[i];
        alert(x);
    }
    ```
2. for...in 
    ```
    // 例如把对象的所有属性依次遍历
    var o = {
        name: 'Jack',
        age: 20,
        city: 'Beijing'
    };

    // 注意 key 是对象 o 的属性名
    for (var key in o) {
        // 滤掉继承的属性
        if (o.hasOwnProperty(key)) {
            alert(key); // 'name', 'age', 'city'
        }
    }

    // 循环数组的索引
    var a = ['A', 'B', 'C'];
    // 注意 i 是数组 a 的索引
    for (var i in a) {
        alert(i); // '0', '1', '2'
        alert(a[i]); // 'A', 'B', 'C'
    }
    ```

### Map和Set
**说明**：JavaScript的默认对象表示方式 `{}` 可以视为其他语言中的 `Map` 或 `Dictionary` 的数据结构，即一组键值对。
但是JavaScript的对象有个小问题，就是键必须是字符串。但实际上Number或者其他数据类型作为键也是非常合理的。
为了解决这个问题，最新的ES6规范引入了新的数据类型 `Map` 和 `Set`。
```
// 测试浏览器是否支持 Map 和 Set 的代码
'use strict';
var m = new Map();
var s = new Set();
alert('你的浏览器支持Map和Set！');
```
1. Map ，是一组键值对结构，具有极快的查找速度
    1. Map 初始化
    使用二维数组初始化，用 javascript 写一个 Map 如下：
    ```
    var m = new Map([['Michael', 95], ['Bob', 75], ['Tracy', 85]]);
    m.get('Michael'); // 95
    ```
    直接初始化一个空的 `Map`
    2. Map 的方法有 `set(x, x), has(x), get(x), delete(x)`，如：
    ```
    var m = new Map();  // 空Map
    m.set('Adam', 67);  // 添加新的key-value
    m.set('Bob', 59);
    m.has('Adam');      // 是否存在key 'Adam': true
    m.get('Adam');      // 67
    m.delete('Adam');   // 删除key 'Adam'
    m.get('Adam');      // undefined
    ```
    set 方法赋值时，会把原来这个属性的值冲掉，如：
    ```
    var m = new Map();
    m.set('Adam', 67);
    m.set('Adam', 88);
    m.get('Adam'); // 88
    ```
2. Set ，和 `Map` 类似，也是一组key的集合，但不存储values。由于 **key不能重复**，所以，在 `Set` 中，没有重复的key。
    1. 创建 `Set`
    创建一个空的 `Set`, 或提供一个 `Array` 作为输入：
    ```
    var s1 = new Set();         // 空Set
    var s2 = new Set([1, 2, 3]);// 含1, 2, 3
    ```
    重复元素会被自动过滤，如：
    ```
    var s = new Set([1, 2, 3, 3, '3']);
    s;                          // Set {1, 2, 3, "3"}
    ```
    2. 操作 `Set` 的函数有 `add(key) delete(key)`:
    ```
    s.add(4);
    s;              // {1, 2, 3, 4}
    s.add(4);       // 重复添加不会有效果
    s;              // {1, 2, 3, 4}
    var s = new Set([1, 2, 3]);
    s;              // Set {1, 2, 3}
    s.delete(3);
    s;              // Set {1, 2}
    ```

### iterable
遍历 `Array` 可以采用下标循环，遍历 `Map` 和 `Set` 就无法使用下标。
为了统一集合类型，ES6标准引入了新的 `iterable` 类型，`Array`、`Map`和 `Set`都属于 `iterable` 类型。
具有 `iterable` 类型的集合可以通过新的 `for ... of` 循环来遍历。
检测浏览器是否支持 `for ... of` 的语法：
```
'use strict';
var a = [1, 2, 3];
for (var x of a) {
}
alert('你的浏览器支持for ... of');
```
1. `for ... of` 的用法：
```
var a = ['A', 'B', 'C'];
var s = new Set(['A', 'B', 'C']);
var m = new Map([[1, 'x'], [2, 'y'], [3, 'z']]);
for (var x of a) { // 遍历Array
    alert(x);
}
for (var x of s) { // 遍历Set
    alert(x);
}
for (var x of m) { // 遍历Map
    // 遍历 Map 时，由于Map是二维数组，所以 x 是一个键值对
    alert(x[0] + '=' + x[1]);
}
```
2. `for ... of` 与 `for ... in` 的区别：
`for ... in` 遍历对象的属性（也包括数组的索引），当我们手动给 `Array` 添加额外属性时，会与 `Array` 的 `length` 属性发生冲突：
```
var a = ['A', 'B', 'C'];
a.name = 'Hello';
for (var x in a) {
    alert(x);   // '0', '1', '2', 'name'
}
a.length;       // 3
```
`for ... of` 不会有这种情况发生，它只循环集合本身的元素，如：
```
var a = ['A', 'B', 'C'];
a.name = 'Hello';
for (var x of a) {
    alert(x); // 'A', 'B', 'C'
}
```
3. 更好的方式是直接使用 `iterator` 本身的 `forEach` 方法，它接收一个函数，每次迭代自动回调这个函数：
    ```
    // Array
    var a = ['A', 'B', 'C'];
    a.forEach(function (element, index, array) {
        // element: 指向当前元素的值
        // index: 指向当前索引
        // array: 指向Array对象本身
        alert(element);
    });

    // Set
    var s = new Set(['A', 'B', 'C']);
    s.forEach(function (element, sameElement, set) {
        // 回调函数的前两个参数都是元素本身
        alert(element);
    });

    // Map
    var m = new Map([[1, 'x'], [2, 'y'], [3, 'z']]);
    m.forEach(function (value, key, map) {
        alert(value);
    });
    ```

## 函数
JavaScript的函数不但是“头等公民”，而且可以像变量一样使用，具有非常强大的抽象能力

### 函数定义和调用
1. 函数定义方法有两种,且两种定义完全等价
    1. 函数定义方法一：
        ```
        function abs(x) {
            if (x >= 0) {
                return x;
            } else {
                return -x;
            }
        }
        ```
    说明：
        * function 指出这是一个函数定义
        * abs 是函数名
        * (x) 是参数列表，多个参数以 `,` 隔开
        * `{...}` 内是代码体，可以包含若干语句，甚至没有任何语句
        * 函数体内部的语句在执行时，一旦执行到 `return` 时，函数就执行完毕，并将结果返回.
        * 如果没有 `return` 语句，函数执行完毕后也会返回结果，只是结果为 `undefined`。
    2. 函数定义方法二：
    由于 javascript 中函数也是一个对象，所以可以进行如下定义：
        ```
        var abs = function (x) {
            if (x >= 0) {
                return x;
            } else {
                return -x;
            }
        };
        ```
    **注意**：需要在函数体末尾加一个 `;`，表示赋值语句结束。
2. 函数调用
    1. 按顺序传入参数即可。
    2. JavaScript允许传入任意个参数而不影响调用，因此传入的参数比定义的参数多也没有问题，虽然函数内部并不需要这些参数:
        ```
        abs(10, 'blablabla');           // 返回10
        abs(-9, 'haha', 'hehe', null);  // 返回9
        ```
    3. 传入的参数比定义的少也没有问题:
        ```
        abs();      // 返回NaN
        ```
    因此要对函数进行检查：
        ```
        function abs(x) {
            if (typeof x !== 'number') {
                throw 'Not a number';
            }
            if (x >= 0) {
                return x;
            } else {
                return -x;
            }
        }
        ```
    4. arguments 是一个关键字，只在函数内部起作用，且永远指向当前函数的调用者传入的所有参数列表，它似乎于 `Array`，但并不是 `Array`，
    利用 arguments 可以获得调用者传入的所有参数，即使函数不定义任何函数，也可以拿到所有参数的值：
    因此常用来判断参数的个数
        ```
        // arguments 类似于数组，具有length属性，通过 [] 索引参数
        function foo(x) {
            alert(x);                   // 10
            for (var i=0; i<arguments.length; i++) {
                alert(arguments[i]);    // 10, 20, 30
            }
        }
        foo(10, 20, 30);

        function abs() {
            if (arguments.length === 0) {
                return 0;
            }
            var x = arguments[0];
            return x >= 0 ? x : -x;
        }
        abs();      // 0
        abs(10);    // 10
        abs(-9);    // 9

        // 定义一个可选参数的函数 foo(a[, b], c), 
        // 要求函数能接收2~3个参数，b是可选参数，如果只传2个参数，b默认为null,则其实现方法为：
        function foo(a, b, c) {
            if (arguments.length === 2) {
                // 实际拿到的参数是a和b，c为undefined
                c = b;      // 把b赋给c
                b = null;   // b变为默认值
            }
        }
        ```
    5. rest 参数
    由于JavaScript函数允许接收任意个参数，于是我们就不得不用arguments来获取所有参数:
        ```
        function foo(a, b) {
            var i, rest = [];
            if (arguments.length > 2) {
                for (i = 2; i<arguments.length; i++) {
                    rest.push(arguments[i]);
                }
            }
            console.log('a = ' + a);
            console.log('b = ' + b);
            console.log(rest);
        }
        ```

    为了获取除了已定义参数a、b之外的参数，我们不得不用arguments，并且循环要从索引2开始以便排除前两个参数.
    这种写法很别扭，只是为了获得额外的rest参数。
    因此，ES6标准引入了rest参数，上面的函数可以改写为：
        ```
        function foo(a, b, ...rest) {
            console.log('a = ' + a);
            console.log('b = ' + b);
            console.log(rest);
        }

        foo(1, 2, 3, 4, 5);
        // 结果:
        // a = 1
        // b = 2
        // Array [ 3, 4, 5 ]

        foo(1);
        // 结果:
        // a = 1
        // b = undefined
        // Array []
        ```
    rest参数只能写在最后，前面用 `...` 标识，从运行结果可知，传入的参数先绑定`a`、`b`，多余的参数以数组形式交给变量`rest`，
    所以，不再需要 `arguments` 我们就获取了全部参数。
    如果传入的参数连正常定义的参数都没填满，也不要紧，`rest` 参数会接收一个空数组（注意不是 `undefined`）。
    6. return 语句的换行问题
    JavaScript引擎有一个在行末自动添加分号的机制，这可能让你栽到 `return` 语句的一个大坑：
        ```
        function foo() {
            return { name: 'foo' };
        }

        foo(); // { name: 'foo' }

        // 如果把return语句拆成两行写下这种形式：
        function foo() {
            return
                { name: 'foo' };
        }
        foo(); // undefined
        // JavaScript引擎在行末自动添加分号的机制，上面的代码实际上变成了:
        function foo() {
            return;                 // 自动添加了分号，相当于return undefined;
                { name: 'foo' };    // 这行语句已经没法执行到了
        }
        // 所以正确的多行写法是:
        function foo() {
            return { // 这里不会自动加分号，因为{表示语句尚未结束
                name: 'foo'
            };
        }
        ```

### 变量作用域
1. 在JavaScript中，用 `var` 申明的变量实际上是有作用域的。
2. 如果一个变量在函数体内部申明，则该变量的作用域为整个函数体，在函数体外不可引用该变量
    ```
    'use strict';
    function foo() {
        var x = 1;
        x = x + 1;
    }
    x = x + 2; // ReferenceError! 无法在函数体外引用变量x
    ```
3. 不同函数内部的同名变量互相独立，互不影响
4. JavaScript的函数可以嵌套，此时，内部函数可以访问外部函数定义的变量，反过来则不行
    ```
    'use strict';
    function foo() {
        var x = 1;
        function bar() {
            var y = x + 1;  // bar可以访问foo的变量x!
        }
        var z = y + 1;      // ReferenceError! foo不可以访问bar的变量y!
    }
    ```
5. 内部函数和外部函数的变量名重名时，内部函数的变量将“屏蔽”外部函数的变量。
    ```
    'use strict';

    function foo() {
        var x = 1;
        function bar() {
            var x = 'A';
            alert('x in bar() = ' + x);     // 'A'
        }
        alert('x in foo() = ' + x);         // 1
        bar();
    }
    ```
6. 变量声明提升
JavaScript的函数定义有个特点，它会先扫描整个函数体的语句，把所有申明的变量“提升”到函数顶部,如：
    ```
    'use strict';

    function foo() {
        var x = 'Hello, ' + y;
        alert(x);
        var y = 'Bob';
    }
    foo();
    // 上面的函数实际上相当于
    function foo() {
        var y; // 提升变量y的申明
        var x = 'Hello, ' + y;
        alert(x);
        y = 'Bob';
    }
    ```
7. 全局作用域
不在任何函数内定义的变量就具有全局作用域。实际上，JavaScript默认有一个全局对象 `window`，全局作用域的变量实际上被绑定到 `window` 的一个属性
    ```
    'use strict';

    var course = 'Learn JavaScript';
    alert(course);          // 'Learn JavaScript'
    alert(window.course);   // 'Learn JavaScript'
    ```
由于函数定义有两种方式，以变量方式 `var foo = function () {}` 定义的函数实际上也是一个全局变量，
因此，顶层函数的定义也被视为一个全局变量，并绑定到 `window` 对象
我们每次直接调用的 `alert()` 函数其实也是 `window` 的一个变量
    ```
    'use strict';

    function foo() {
        alert('foo');
    }

    foo();          // 直接调用foo()
    window.foo();   // 通过window.foo()调用
    ```
这说明JavaScript实际上只有一个全局作用域。任何变量（函数也视为变量），如果没有在当前函数作用域中找到，
就会继续往上查找，最后如果在全局作用域中也没有找到，则报ReferenceError错误。
8. 名字空间
全局变量会绑定到 `window` 上，不同的JavaScript文件如果使用了相同的全局变量，或者定义了相同名字的顶层函数，都会造成命名冲突，并且很难被发现。
减少冲突的一个方法是把自己的所有变量和函数全部绑定到一个全局变量中
    ```
    // 唯一的全局变量MYAPP:
    var MYAPP = {};

    // 其他变量:
    MYAPP.name = 'myapp';
    MYAPP.version = 1.0;

    // 其他函数:
    MYAPP.foo = function () {
        return 'foo';
    };
    ```
把自己的代码全部放入唯一的名字空间 `MYAPP` 中，会大大减少全局变量冲突的可能
9. 局域作用域
由于JavaScript的变量作用域实际上是函数内部，我们在 `for` 循环等语句块中是无法定义具有局部作用域的变量的
为了解决块级作用域，ES6引入了新的关键字 `let`，用 `let` 替代 `var` 可以申明一个块级作用域的变量
    ```
    'use strict';

    function foo() {
        var sum = 0;
        for (var i=0; i<100; i++) {
            sum += i;
        }
        i += 1; // 仍然可以引用变量i
    }

    function foo() {
        var sum = 0;
        for (let i=0; i<100; i++) {
            sum += i;
        }
        i += 1; // SyntaxError
    }
    ```
10. 常量
由于 `var` 和 `let` 申明的是变量，如果要申明一个常量，在ES6之前是不行的，我们通常用全部大写的变量来表示“这是一个常量，不要修改它的值”
    ```
    var PI = 3.14;
    ```
ES6标准引入了新的关键字 `const` 来定义常量，`const` 与 `let` 都具有块级作用域
    ```
    const PI = 3.14;
    PI = 3;         // 某些浏览器不报错，但是无效果！
    PI;             // 3.14
    ```
### 方法
在一个对象中绑定函数，称为这个对象的方法.
给xiaoming绑定一个函数，就可以做更多的事情,如：
    ```
    var xiaoming = {
        name: '小明',
        birth: 1990,
        age: function () {
            var y = new Date().getFullYear();
            return y - this.birth;
        }
    };

    xiaoming.age;   // function xiaoming.age()
    xiaoming.age(); // 今年调用是25,明年调用就变成26了
    ```
1. this 关键字
在一个方法内部，`this` 是一个特殊变量，它始终指向当前对象,JavaScript的函数内部如果调用了this，那么这个this的指向将 **视情况而定**！
这个是 javascript 的大坑,而且，即使用 obj.xxx()的方式赋值给其它全局对象，也会出错。
    ```
    function getAge() {
        var y = new Date().getFullYear();
        // 这里的 this 指向的对象将视情况而定
        return y - this.birth;
    }

    var xiaoming = {
        name: '小明',
        birth: 1990,
        age: getAge
    };

    // 这里调用age,则 this 指向 xiaoming 这个对象
    xiaoming.age();         // 25, 正常结果
    // 这里调用getAge()方法，this 指向的是全局对象 window
    getAge();               // NaN
    var fn = xiaoming.age;  // 先拿到xiaoming的age函数
    fn();                   // NaN
    ```
所以， **要保证 `this` 指向正确，必须用 `obj.xxx()` 的形式调用**
由于这是一个巨大的设计错误，要想纠正可没那么简单。ECMA决定，在strict模式下让函数的 `this` 指向 `undefined`，因此，在strict模式下，你会得到一个错误：
这个决定只是让错误及时暴露出来，并没有解决 `this` 应该指向的正确位置
    ```
    'use strict';

    var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function () {
        var y = new Date().getFullYear();
        return y - this.birth;
    }
    };

    var fn = xiaoming.age;
    fn();           // Uncaught TypeError: Cannot read property 'birth' of undefined
    ```
而且，在方法重构时，也会产生错误：
    ```
    'use strict';

    var xiaoming = {
        name: '小明',
        birth: 1990,
        age: function () {
            function getAgeFromBirth() {
                var y = new Date().getFullYear();
                return y - this.birth;
            }
            return getAgeFromBirth();
        }
    };

    xiaoming.age(); // Uncaught TypeError: Cannot read property 'birth' of undefined
    ```
因为 `this` 指针只在 `age` 方法的函数内指向 `xiaoming`，在函数内部定义的函数，
`this` 又指向`undefined`了！（在非strict模式下，它重新指向全局对象 `window`！）
解决办法为用一个 `that` 变量先捕获 `this`：
    ```
    'use strict';

    var xiaoming = {
        name: '小明',
        birth: 1990,
        age: function () {
            var that = this;            // 在方法内部一开始就捕获this
            function getAgeFromBirth() {
                var y = new Date().getFullYear();
                return y - that.birth;  // 用that而不是this
            }
            return getAgeFromBirth();
        }
    };

    xiaoming.age();                     // 25
    ```
2. apply() 和 call()
虽然在一个独立的函数调用中，根据是否是strict模式，`this` 指向 `undefined` 或 `window`，不过，我们还是可以控制 `this` 的指向的！

要指定函数的 `this` 指向哪个对象，可以用函数本身的 `apply` 方法.
它接收两个参数，第一个参数就是需要绑定的 `this` 变量，第二个参数是 `Array`，表示函数本身的参数。用 `apply()` 修复过的 `getAge()` 调用如下：
    ```
    function getAge() {
        var y = new Date().getFullYear();
        return y - this.birth;
    }

    var xiaoming = {
        name: '小明',
        birth: 1990,
        age: getAge
    };

    xiaoming.age();             // 25
    getAge.apply(xiaoming, []); // 25, this指向xiaoming, 参数为空
    ```
另一个与apply()类似的方法是call()，唯一区别是：
    * apply()把参数打包成Array再传入;
    * call()把参数按顺序传入。
例如：
    ```
    Math.max.apply(null, [3, 5, 4]);    // 5
    Math.max.call(null, 3, 5, 4);       // 5
    ```
3. 装饰器
因为JavaScript的所有对象都是动态的，即使内置的函数，我们也可以重新指向新的函数。
所以利用 `apply()` 函数，我们可以动态改变函数的行为。
现在假定我们想统计一下代码一共调用了多少次 `parseInt()`，可以把所有的调用都找出来，然后手动加上 `count += 1`，
不过这样做太傻了。最佳方案是用我们自己的函数替换掉默认的 `parseInt()`：
    ```
    var count = 0;
    var oldParseInt = parseInt; // 保存原函数

    window.parseInt = function () {
        count += 1;
        // 这里巧妙地用apply方法和 arguments,实现了对原方法的重新调用。
        return oldParseInt.apply(null, arguments); // 调用原函数
    };

    // 测试:
    parseInt('10');
    parseInt('20');
    parseInt('30');
    count; // 3
    ```

### 高阶函数
高阶函数英文叫Higher-order function。
JavaScript的函数其实都指向某个变量。既然变量可以指向函数，函数的参数能接收变量，那么接收一个函数作为参数的函数，就称之为高阶函数,如：
    ```
    function add(x, y, f) {
        return f(x) + f(y);
    }
    ```

#### map/reduce
map/reduce的概念不再介绍，可以看Google的论文[MapReduce: Simplified Data Processing on Large Clusters](https://research.google.com/archive/mapreduce.html)
1. map()，定义在 `Array` 内部，可以调用 `Array` 的 `map()` 方法，传入自己的函数，这个函数会对 `Array` 的每个元素执行该函数，从而得到一个新的 `Array`：
    ```
    function pow(x) {
        return x * x;
    }

    var arr = [1, 2, 3, 4, 5, 6, 7, 8, 9];
    // 注意 map 传入的参数是函数对象本身，是一个函数。
    arr.map(pow); // [1, 4, 9, 16, 25, 36, 49, 64, 81]
    ```
    `map()` 作为高阶函数，实际上是把运算规则抽象了，例如还可以进行如下调用：
    ```
    var arr = [1, 2, 3, 4, 5, 6, 7, 8, 9];
    arr.map(String); // ['1', '2', '3', '4', '5', '6', '7', '8', '9']
    ```
2. reduce(), 同样定义在 `Array` 内部，接受一个函数，这个函数有两个参数，然后 `reduce` 会对 `Array` 内的元素，从前到后，依次调用这个函数，如下：
    ```
    [x1, x2, x3, x4].reduce(f) = f(f(f(x1, x2), x3), x4)

    // 比如求和函数可以这样实现：
    var arr = [1, 3, 5, 7, 9];
    arr.reduce(function (x, y) {
        return x + y;
    }); // 25
    ```

#### filter
1. filter(), 用于把 `Array` 的某些元素过滤掉，然后返回剩下的元素。
与 `map()` 类似，`filter()` 也接收一个函数，但 `filter()` 会把传入的函数依次作用于每个元素，然后根据返回值是 `true` 还是 `false` 决定是否保留元素。
例如，在一个 `Array` 中，删掉偶数，只保留奇数，可以这么写：
    ```
    var arr = [1, 2, 4, 5, 6, 9, 10, 15];
    var r = arr.filter(function (x) {
        return x % 2 !== 0;
    });
    r;              // [1, 5, 9, 15]
    ```
2. 回调函数
`filter()` 接收的回调函数，其实可以有多个参数。通常我们仅使用第一个参数，表示 `Array` 的某个元素。回调函数还可以接收另外两个参数，表示元素的位置和数组本身
    ```
    var arr = ['A', 'B', 'C'];
    var r = arr.filter(function (element, index, self) {
        console.log(element);   // 依次打印'A', 'B', 'C'
        console.log(index);     // 依次打印0, 1, 2
        console.log(self);      // self就是变量arr
        return true;
    });
    ```

#### sort
`Array` 的 `sort()` 方法默认把所有元素先转换为String,按照ASCII的大小比较再排序的。
幸运的是，`sort()` 方法也是一个高阶函数，它还可以接收一个比较函数来实现自定义的排序
通常规定，对于两个元素x和y，如果认为 `x < y`，则返回 `-1`，如果认为 `x == y`，则返回 `0`，如果认为 `x > y`，则返回 `1`
要按数字大小排序:
    ```
    var arr = [10, 20, 1, 2];
    arr.sort(function (x, y) {
        if (x < y) {
            return -1;
        }
        if (x > y) {
            return 1;
        }
        return 0;
    }); // [1, 2, 10, 20]
    ```
`sort()` 方法会直接对 `Array` 进行修改，它返回的结果仍是当前 `Array`：
    ```
    var a1 = ['B', 'A', 'C'];
    var a2 = a1.sort();
    a1; // ['A', 'B', 'C']
    a2; // ['A', 'B', 'C']
    a1 === a2; // true, a1和a2是同一对象
    ```

### 闭包
1. 函数作为返回值：高阶函数除了可以接受函数作为参数外，还可以把函数作为结果值返回。
通常情况下，求和的函数是这样定义的:
    ```
    function sum(arr) {
        return arr.reduce(function (x, y) {
            return x + y;
        });
    }

    sum([1, 2, 3, 4, 5]); // 15
    ```
但是，如果不需要立刻求和，而是在后面的代码中，根据需要再计算怎么办？解决办法是：不返回求和的结果，而是返回求和的函数！
    ```
    function lazy_sum(arr) {
        var sum = function () {
            return arr.reduce(function (x, y) {
                return x + y;
            });
        }
        // 这里 sum 是一个函数！
        return sum;
    }
    ```
当我们调用 `lazy_sum()` 时，返回的并不是求和结果，而是求和函数：
    ```
    var f = lazy_sum([1, 2, 3, 4, 5]); // 这里的 f 实际上是函数 sum() 的一个对象
    ```
调用函数f时，才真正计算求和的结果：
    ```
    f(); // 15
    ```
用例子说明“闭包”的定义：在上面的例子中，函数 `lazy_sum()` 内部定义了函数 `sum()`, 并且，内部的函数 `sum()` 可以引用外部函数的参数和局部变量，
当 `lazy_sum()` 返回函数 `sum()` 时，相关的参数和变量都保存在返回的函数中，这种称为 **“闭包”** 的程序结构有极大的能力。
请再注意一点，当我们调用 `lazy_sum()` 时，每次调用都会返回一个新的函数，即使传入相同的参数：
    ```
    var f1 = lazy_sum([1, 2, 3, 4, 5]);
    var f2 = lazy_sum([1, 2, 3, 4, 5]);
    f1 === f2;          // false
    ```
且 `f1` 与 `f2` 的调用结果互不影响，因为是两个不同的对象。
2. 闭包：返回的函数在其定义内部引用了局部变量 `arr`，所以，当一个函数返回了一个函数后，其内部的局部变量还被新函数引用
**注意**:返回的函数并没有立刻执行，而是直到调用了 `f()` 才执行。我们来看一个例子：
    ```
    function count() {
        var arr = [];
        for (var i=1; i<=3; i++) {
            arr.push(function () {
                return i * i;
            });
        }
        return arr;
    }

    var results = count();
    var f1 = results[0];
    var f2 = results[1];
    var f3 = results[2];
    ```
上面的例子中，每次循环，都创建了一个新的函数，然后，把创建的3个函数都添加到一个 `Array` 中返回
但调用 `f1() f2() 和 f3()` 的结果不是 `1, 4, 9`,如：
    ```
    f1(); // 16
    f2(); // 16
    f3(); // 16
    ```
原因就在于返回的函数引用了变量 `i`，但它并非立刻执行。等到3个函数都返回时，它们所引用的变量 `i` 已经变成了4，因此最终结果为16。
**特别注意**：返回函数时不要引用任何循环变量，或者后续会发生变化的变量！！！
如果一定要引用循环变量怎么办？方法是:再创建一个函数，用该函数的参数绑定循环变量当前的值，无论该循环变量后续如何更改，已绑定到函数参数的值不变：
    ```
    function count() {
        var arr = [];
        for (var i=1; i<=3; i++) {
            // 里面的 function 定义好之后就直接传入参数 i 进行调用了
            arr.push((function (n) {
                return function () {
                    return n * n;
                }
            })(i));
        }
        return arr;
    }

    var results = count();
    var f1 = results[0];
    var f2 = results[1];
    var f3 = results[2];

    f1(); // 1
    f2(); // 4
    f3(); // 9
    ```
补充：创建一个匿名函数并立刻执行的语法是：
    ```
    (function (x) {
        return x * x;
    })(3); // 9
    ```
理论上讲，创建一个匿名函数并立刻执行可以这样写：
    ```
    function (x) { return x * x } (3);
    ```
但由于JavaScript语法解析的问题，会报SyntaxError错误，因此需要用括号把整个函数定义括起来：
    ```
    (function (x) { return x * x }) (3);
    ```
3. 闭包的功能：
    1. 上面讲到的 延迟计算
    2. 封装私有变量
        ```
        'use strict';

        function create_counter(initial) {
            var x = initial || 0;
            return {
                inc: function () {
                    x += 1;
                    return x;
                }
            }
        }
        // 其使用方法如下：
        var c1 = create_counter();
        c1.inc(); // 1
        c1.inc(); // 2
        c1.inc(); // 3

        var c2 = create_counter(10);
        c2.inc(); // 11
        c2.inc(); // 12
        c2.inc(); // 13
        ```
    在返回的对象中，实现了一个闭包，这个闭包携带了私有变量 `x`，并且，代码无法从外部访问到该变量。
    换句话说，闭包就是携带状态的函数，并且它的状态可以完全对外部隐藏。
### 箭头函数
ES6标准新定义了一种新的函数：箭头函数(其实是lambda表达式，用箭头表示而已),类似于匿名函数（但有区别），其定义及其涵义如下：
    ```
    x => x * x          // 定义1

    function (x) {      // 涵义1
        return x * x;
    }

    x => {              // 定义2, 包含多条语句，用 {...} 和return
        if (x > 0) {
            return x * x;
        }
        else {
            return - x * x;
        }
    }

    (x, y) => x*x + y*y // 定义3,如果参数不只一个，用 () 括起来
    () => 3.14          // 定义3,无参
    (x, y, ...rest) => {// 可变参
        var i, sum = x + y;
        for (i=0; i<rest.length; i++) {
            sum += rest[i];
        }
        return sum;
    }

    // 返回一个对象时,为了避免与函数体的语法冲突，需要在对象外面加一个 {...},如下：
    // SyntaxError:
    x => { foo: x }
    // ok:
    x => ({ foo: x })
    ```
2. this ，箭头函数与匿名函数的区别：箭头函数内部的 `this` 是词法作用域，由上下文确定,说简单一些，箭头函数里的 `this` 的作用域很明确。
上面提到过，这种情况下，this的作用域容易出问题：
    ```
    var obj = {
        birth: 1990,
        getAge: function () {
            var b = this.birth; // 1990
            var fn = function () {
                return new Date().getFullYear() - this.birth; // this指向window或undefined
            };
            return fn();
        }
    };
    ```
但箭头函数里，不会有这样的问题，写法如下：
    ```
    var obj = {
        birth: 1990,
        getAge: function () {
            var b = this.birth; // 1990
            var fn = () => new Date().getFullYear() - this.birth; // this指向obj对象
            return fn();
        }
    };
    obj.getAge(); // 25
    ```
由于 `this` 在箭头函数中已经按照词法作用域绑定了，所以，用 `call()` 或者 `apply()` 调用箭头函数时，无法对 `this` 进行绑定，即传入的第一个参数被忽略
    ```
    var obj = {
        birth: 1990,
        getAge: function (year) {
            var b = this.birth; // 1990
            var fn = (y) => y - this.birth; // this.birth仍是1990
            // 这里调用了 call() 函数
            return fn.call({birth:2000}, year);
        }
    };
    obj.getAge(2015); // 25
    ```

### generator
generator 是ES6标准引入的新的数据类型。一个generator看上去像一个函数，但可以返回多次.
javascript 的 generator 借鉴了 Python 的生成器概念和语法。
1. 生成器概念
函数一般是传入一个参数，返回一个计算结果。在函数执行过程中，如果没有遇到 `return` 语句，函数会继续执行到函数结尾或遇到 `return`。
generator 与 函数很像，定义如下：
    ```
    function* foo(x) {
        // 注意 yield 的用法
        yield x + 1;
        yield x + 2;
        return x + 3;
    }
    ```
generator 与函数的区别在于：generator 通过 `function*` 定义， **注意**：这里的 `*` 不能省略，且除了 `return` 外，还有 `yield` **多次返回**。
对 **多次返回** 的解释如下：
    ```
    // 正常的斐波那契数列生成算法
    function fib(max) {
        var
            t,
            a = 0,
            b = 1,
            arr = [0, 1];
        while (arr.length < max) {
            t = a + b;
            a = b;
            b = t;
            arr.push(t);
        }
        return arr;
    }

    // 测试:
    fib(5); // [0, 1, 1, 2, 3]
    fib(10); // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
    // 这里返回的是一个数组，且函数只能返回一次。
    // 但是如果换成 generator ，就可以一次返回一个数，不断返回多次。
    // generator 的斐波那契数列生成算法如下：
    function* fib(max) {
        var
            t,
            a = 0,
            b = 1,
            n = 1;
        while (n < max) {
            // 注意这里用到了 yield
            yield a;
            t = a + b;
            a = b;
            b = t;
            n ++;
        }
        return a;
    }
    ```
直接调用上面的 generator 写法的 `fib()` 函数，如下所示：
    ```
    fib(5); // fib {[[GeneratorStatus]]: "suspended", [[GeneratorReceiver]]: Window}
    ```
这表明，直接调用 generator 与直接调用函数不同，直接调用 generator 会返回一个 generator 对象，但不会立刻执行它。
调用 generator 的方法有两种：
    1. 调用 generator 的 `next()` 方法：
        ```
        var f = fib(5);
        f.next(); // {value: 0, done: false}
        f.next(); // {value: 1, done: false}
        f.next(); // {value: 1, done: false}
        f.next(); // {value: 2, done: false}
        f.next(); // {value: 3, done: true}
        ```
    `next()` 方法会执行generator的代码，然后，每次遇到 `yield x`;就返回一个对象 `{value: x, done: true/false}`，
    然后“暂停”。返回的 `value` 就是 `yield` 的返回值，`done` 表示这个generator是否已经执行结束了。
    如果 `done` 为 `true`，则 `value` 就是 `return`的返回值。
    执行到 `none` 为 `true` 时，这个generator对象就已经全部执行完毕，不要再继续调用 `next()` 了。
    2. 通过 `for ... of` 循环迭代 generator 对象，这种方式不需要我们自己判断 `done`：
        ```
        for (var x of fib(5)) {
            console.log(x); // 依次输出0, 1, 1, 2, 3
        }
        ```

## 标准对象
### Date
### RegExp
### Json
## 面向对象编程
### 创建对象
### 原型继承
### class继承
## 浏览器
### 浏览器对象
### 操作DOM
#### 更新DOM
#### 插入DOM
#### 删除DOM
### 操作表单
### 操作文件
### Ajax
### Promise
### Canvas
## jQuery
### 选择器
#### 层级
#### 查找和过滤
### 操作DOM
#### 修改DOM结构
### 事件
### 动画
### Ajax
### 扩展
## 错误处理
### 错误传播
### 异步错误处理
## underscore
### Collections
### Arrays
### Functions
### Objects
### Chaining
## Node.js
### 安装Node.js和npm
### 第一个Node程序
### 搭建Node开发环境
### 模块
### 基本模块
#### fs
#### stream
#### http
#### crypto
### Web开发
#### koa
##### koa入门
##### 处理URL
##### 使用Nunjucks
##### 使用MVC
#### mysql
##### 使用Sequlize
##### 建立Model
#### mocha
##### 编写测试
##### 异步测试
#### Websocket
##### 使用ws
##### 编写聊天室
#### REST
##### 编写REST API
##### 开发REST API
#### MVVM
##### 单向绑定
##### 双向绑定
##### 同步DOM结构
##### 集成API
##### 在线电子表格
#### 自动化工具
### React
### 期末总结
