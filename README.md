# mvvm
### 什么是MVVM？
MVVM是Model-View-ViewModel的缩写,是一种程序架构设计。

**Model**

Model指的是数据层，是纯净的数据,不处理行为.

**View**

View指视图层，是直接呈现给用户的部分，简单的来说，对于前端就是HTML。当然视图层是可变的，你完全可以在其中随意添加元素。这不会改变数据层，只会改变视图层呈现数据的方式。视图层应该和数据层完全分离。

**ViewModel**
既然视图层应该和数据层分离，那么我们就需要设计一种结构，让它们建立起某种联系。当我们对Model进行修改的时候，ViewModel就会把修改自动同步到View层去。同样当我们修改View，Model同样被ViewModel自动修改。

可以看出，如何设计能够高效自动同步View与Model的ViewModel是整个MVVM框架的核心和难点。

### 实现效果



模仿Vue,当我们修改页面上的绑定的内容的时候,对后台绑定的数据数据进行修改.实现数据的双向绑定.
主要会用到`Object.defineProperty()`

`Object.defineProperty()` 方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性。这个方法执行完成后会返回修改后的这个对象。
利用这个方法实现数据的动态监听.

`Object.defineProperty()`可以设置`configurable configurable`

`configurable` 的值设置为 false 后(如果没设置，默认就是 false)，以后就不能再次通过 `Object.defineProperty`修改属性，也无法删除该属性。

设置 `enumerable` 属性为 false 后，遍历对象的时候会忽略当前属性(如果未设置，默认就是 false不可遍历)。

#### 数据劫持
`get` 和` set `叫存取描述符，有以下可选键值：

`get`：一个给属性提供 getter 的方法，如果没有 getter 则为 undefined。该方法返回值被用作属性值。默认为 undefined。
`set`：一个给属性提供 setter 的方法，如果没有 setter 则为 undefined。该方法将接受唯一参数，并将该参数的新值分配给该属性。默认为 undefined。

当用户读取某个数据的时候会调用`get`方法,当用户修改某个数据的时候调用`set`方法

利用`Object.defineProperty()`的方法实现数据的动态监听
```
var data = {
  name: 'hunger',
  friends: [1, 2, 3]
}
observe(data)

console.log(data.name)
data.name = 'valley'
data.friends[0] = 4


function observe(data) {
  if(!data || typeof data !== 'object') return
  for(var key in data) {
    let val = data[key]     //注意点1：这里是 let 不是 var，想想为什么
    Object.defineProperty(data, key, {
      enumerable: true,
      configurable: true,
      get: function() {
        console.log(`get ${val}`)
        return val
      },
      set: function(newVal) {
        console.log(`changes happen: ${val} => ${newVal}`)
        val = newVal
      }
    })
    if(typeof val === 'object'){
      observe(val)
    }
  }
}
```
使用let是因为,var会变量提升,在做遍历的时候会重读声明这个`val`,最终只会得到最后一轮遍历的值,而let则不会出现这个问题.

#### 观察者模式(发布订阅模式)
```
//发布者
function Subject() {
  this.observers = [] //存放订阅者
}
Subject.prototype.addObserver = function(observer) {
  this.observers.push(observer)
}
Subject.prototype.removeObserver = function(observer) {
  var index = this.observers.indexOf(observer)//找到指定的观察者取消订阅
  if(index > -1){
    this.observers.splice(index, 1)
  }
}
Subject.prototype.notify = function() {
  this.observers.forEach(function(observer){
    observer.update()//通知观察者更新
  })
}
//订阅者
function Observer(name) {
  this.name = name
  this.update = function(){
    console.log(name + ' 已订阅...')
  }
}  


```
![](https://ws1.sinaimg.cn/large/006EdmRUly1fuwaffeki9j30j407bglu.jpg)

### mvvm 创建的步骤


```
<body>
  <div id="app" >{{name}}</div>

  <script>
    function mvvm(){
        //todo...
    }
    var vm = new mvvm({
      el: '#app',
      data: { 
          name: 'jirengu' 
      }
    })
  </script>
<body>
```

1.类似于VUE的一个模板,当执行代码的时候解析模板中的变量`{{name}}`替换成实际的值,为此,声明这个mvvm构造函数,初始化`data`中的数据,在mvvm函数中写入具体的执行方法.

这里会用到观察者模式,观察者是{{name}},主题是`data.name`,解析模板时读取`data.name`中的值,将数据展示出来.当用户修改数据的时候利用`set`发出通知,对数据做出修改.

2.`let new mvvm()` mvvm.init(opts)获取到当前作用的元素,将mvvm的`data`数据暂存到mvvm对象中,以便调用.监控这些数据的变化.

3.声明一个`observe`函数监控数据的变化,监控过程中,将监控到的每个属性创造一个主题,这样做是为了当用户修改视图上的数据,就表示这个主题的值变了,然后再通知其他订阅者,主要依靠的是set方法.

4. 解析模板 用`traverse`函数 遍历节点.获取到文本,具体步骤查看代码.

5.替换数据 `renderText()`,用正则表达式匹配到文本,将视图上的文本替换去掉了括号后的文本.这个过程还会创建一个观察者`new Observer`,具体见代码

6.声明`new Observer`作为观察者, 执行第2个步骤并执行`getValue`,什么时候订阅呢,那就是执行`getValue`时候内部会执行`let value = this.vm.$data[this.key]`,那么就会激活`observe`的`get`方法,`get`方法`return`新的值.

7.当用户修改的数据之后,就会调用`set`,通知所有的主题订阅者 ,执行`update`更新数据.

