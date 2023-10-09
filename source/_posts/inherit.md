---
title: JS的各种类型继承模式总结
date: 2023-05-07 10:51:31
tags: [类型继承,原型,寄生式组合继承]
categories: [JavaScript]
index_img: https://imgnothing.top/images/createBlog.png
banner_img: https://imgnothing.top/images/createBlog.png
---

# 原型式继承

+ 原型式继承就是`Object.create()`的实现原理。
+ 原型式继承非常适用于不需要单独创建构造函数，但仍需要在对象实例之间共享信息的场合。

```js
 function create(o){
    // 创建一个临时构造函数，将传入的对象赋值给这个构造函数的原型
    function F(){}  
    F.prototype = o
    return new F()
  }
  let sup = {name:'jiabin'};
  let sub = create(sup);
  sub.age = 23
  console.log(sub);
```

![image-20230507160253096](https://imgnothing.top/images/image-20230507160253096.png)

> 记住`create()`这个函数，下文的代码中会反复用到

# 寄生式继承

```js
  function createAnother(o){
    let clone = create(o);   // 调用函数创建一个新的实现继承的对象
    clone.sayHi = function(){   // 以某种方式增强这个对象
      console.log('hello');
    }
    return clone
  }
```

与原型式继承比较接近的是寄生式继承，它的思想是在原型式继承的基础上，以某种方式对子对象进行改造（增强）。

### 寄生式 vs 原型式

+ 与原型式继承的不同之处是，寄生式继承不仅实现了实例之间的继承关系，并且增强了子实例。

+ 寄生式与原型式都适合于不需要构造函数，只需关注对象实例的场景

# 盗用构造函数

由于原型链的原因，以上两种继承方式创建的对象之间是会共享引用类型的属性的，这导致不同的对象之间无法拥有自己独立的数据。

通过调用父类构造函数的的call / apply函数，可以实现夫类型构造函数的借用。使得子类型的构造函数也能创建独立的父类型数据。

``` js
  function SupType(){
    this.name = 'jiabin';
    this.age = 23;
    this.sayHi(){
        console.log('hello!')
    }
  } 
  function SubType(){
    SupType.apply(this)
    this.job = 'worker'
  }
```

由`SubType`构造函数创建的对象也包含`SupType`构造函数的属性，并且是对象本身所有的。但缺点是不能重用父类型的方法（大量同名同作用，但内存地址不相同的函数）

# 组合继承

```js
function SupType(){
    this.name = 'jiabin';
    this.age = 23
  }
  SupType.prototype.getName = function(){
    console.log(this.name);
  }
  function SubType(){
    this.job='programmer'
    SupType.apply(this)    // 盗用父类构造函数
  }
  SubType.prototype = new SupType();     // 第二次调用父类构造函数
  SubType.prototype.constructor = SubType   // 修正子类型原型的constructor值，保持原型链不变，使得instanceof和isPropertyOf()正常有效
  sub = new SubType()
  console.log(sub);
```

![image-20230507160643061](https://imgnothing.top/images/image-20230507160643061.png)

综合了原型链和盗用构造函数，使得子类型的实例既可以实现方法重用，又可以拥有自己的属性数据



# 寄生式组合继承

组合继承实现了基本的方法重用和独立属性，但他存在着效率问题。

+ 最主要的效率问题是父类型的构造函数被调用了两次，一次是在盗用构造函数时，另一次是在给子类型构造函数的原型赋值时。实际上，对于第二次调用，目的只是为了重写子类型的原型，完全不需要调用父类型构造函数来实例化一个父类对象这么麻烦，可以通过创建一个继承父类型的简单对象（寄生于父类型原型的寄生虫），然后增强这个对象（寄生虫），实现子类型对父类型的继承
+ 其次是子类型构造函数的原型中存在着冗余的父类型的属性。

```js
  function SupType1(){
    this.name = 'jiabin';
    this.age = 23;
  }

  // 在父类型的原型上声明可让子类型重用的方法
  SupType1.prototype.getAge = function(){
    console.log(this.age);
  }

  function SubType1(){
    SupType.apply(this)
    this.job = 'worker'
  }

  function inheritSupType(SupType,SubType){
    let prototype = create(SupType.prototype);  // 创建一个寄生虫对象，它就是子类型的原型
    prototype.constructor = SubType    // 修正寄生虫对象的constructor的值
    SubType.prototype = prototype      // 给子类型的原型重新赋值
  }
  inheritSupType(SupType1,SubType1);

  sub = new SubType1();
  console.log(sub);

```

![image-20230507162321625](https://imgnothing.top/images/image-20230507162321625.png)

可以看到**没有了冗余**的父类型属性，而且不同的子类型示例**可以重用**同一个父类型的方法。



# 总结

+ 原型式继承和寄生式继承适用于继承某个对象实例的场景。创建的对象存在数据共享的问题
+ 在子类型构造函数中盗用父类型构造函数，解决了数据共享问题，但是引发方法不能重用的问题
+ 组合继承，组合了盗用构造函数和原型链，解决了上述两个问题，但存在效率问题和冗余父类型属性
+ 寄生式组合继承解决了组合继承的效率问题、避免了冗余对的父类属性，是最佳的类型继承范式，是ES6 extends关键字的实现原理。

总而言之，在ES6之前，JS 的确可以通过各种操作模拟类似于类的行为，但最终实现的代码显得非常冗长和杂乱，这也正是ES6 推出类（class) 这个语法糖结构的必要性所在。但同时需要指明的是，class背后使用的仍然是原型和构造函数的概念。
