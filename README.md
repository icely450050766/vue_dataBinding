### vue双向数据绑定 原理

##### 每次new一个新的vue对象时，主要做了：   
   
1）监听数据：为 vm.data 的每个属性，生成一个主题对象        

2）编译HTML：为每个与 vm.data 数据绑定的相关节点，生成一个订阅者，订阅者会将自己订阅到相应主题对象中        

3）虚拟dom：nodeToFragement( dom )       


##### 代码详解：     

######【虚拟dom】nodeToFragment( node, vm ):       

（1） DocumentFragment：  
    
&gt; 不属于文档树。       

&gt; 把一个 DocumentFragment 节点插入文档树时，插入的不是 DocumentFragment 自身，而是它的所有子孙节点。     

&gt; 这使得 DocumentFragment 成了有用的占位符。     

（2）vue吸收了react虚拟DOM的优点，使用DocumentFragment处理节点，其性能和速度远胜于直接操作dom。     
vue编译时，将所有挂载在dom上的子节点，劫持到使用DocumentFragment处理节点，        
等所有操作都执行完毕，将DocumentFragment 返回到 挂载目标上。        

（3）劫持：使用 `appendChild()` 移动dom节点，因此不断检测 `node.firstChild` 即可。       

（4）劫持之前，要先对dom节点编译（数据初始化绑定）。        


    // 【虚拟dom】将要操作的dom，全部劫持到DocumentFragment
    var nodeToFragment = ( node, vm ) => {
        const flag = document.createDocumentFragment();
        let child;
        while( child = node.firstChild ){
    
            // 不断劫持 挂载元素下 的所有dom节点，到创建的DocumentFragment
            // appendChild是移动dom，不是复制插入。因此可不断 node.firstChild
    
    //            console.log( child );
            compile( child, vm ); // 编译
            flag.appendChild( child );
        }
    //        console.log( flag );
        return flag;
    };



######【编译】compile( node, vm ):       

（1）根据 dom节点的 nodeType，判断节点类型。这里只处理了元素、文本节点。     

（2）如果是元素结点，且有 `v-model` 属性（input的双向数据绑定）： 
      
&gt; 增加 input事件监听，回调函数触发 `vm.data.key` 的setter。      
    
（3）如果是文本结点（简单处理，只支持 `{{ key }}`，文本前后不能有其他内容）： 
      
&gt; 正则匹配文本：`{{ key }}`，拿到 key属性名。       

（4）给dom节点（input、文本节点） 增加订阅者（订阅者功能：根据数据 `vm.data`，设置dom内容）。   


    // 【编译】数据初始化绑定
    var compile = ( node, vm ) => {
        var regex = /\{\{(.*)\}\}/; // 正则{{ … }}
    
        if( node.nodeType === 1 ){ // 节点类型为元素
            var attrs = node.attributes; // 属性
            for( let i=0; i < attrs.length; i++ ){
                let attr = attrs[i];
    
                if( attr.nodeName == 'v-model' ){
                    let name = attr.nodeValue; // 属性值
    //                    node.value = vm.data[name]; // 设置节点的value值（初始化）
                    node.removeAttribute('v-model'); // 移除属性
                    new Watcher( vm, node, name ); // 添加订阅者
    
                    // input事件监听：触发 setter
                    node.addEventListener('input', function (e) {
                        vm.data[name] = e.target.value;
                    });
                }
            }
        }else if( node.nodeType === 3 ){ // 节点类型为文本（换行符、文字等）
    
    //            console.log( node.nodeValue );
            if( regex.test( node.nodeValue ) ){
                let name = RegExp.$1; // 获取搭配的字符串
                name = name.trim();
    //                node.nodeValue = vm.data[name];
    
                new Watcher( vm, node, name ); // 添加订阅者
            }
        }
    };



###### 订阅发布模式（观察者模式）:  

（1）让多个订阅者 同时监听一个主题对象，主题对象改变时，通知所有订阅者。       

（2）订阅者 `class Watcher{ }`（把dom和数据连接起来）     
 
&gt; 提供一个 `update()`，读取 `vm.data.key`的值（触发主题对象getter），更新dom内容。      

&gt; `new Watcher()` 时，会设置 全局变量`Dep.target`，然后执行`update()` 触发 属性的getter，从而将 订阅者添加到 主题对象。      

（3）主题发布器 `class Dep{ }`（当数据 `vm.data.key` 改变，发送通知给订阅者，更新视图）     
 
&gt; 提供 压入订阅者方法 `addSub()`，广播通知订阅者更新视图方法 `notify()`。      


    // 订阅者（根据数据vm.data，设置dom）
    class Watcher{
        constructor( vm, node, name ){
            this.name = name;
            this.node = node;
            this.vm = vm;
    
            // 每次增加订阅者，都设置 Dep.target全局变量，
            // 执行update（触发getter），将订阅者添加到dep.subs[]
            Dep.target = this; // 全局变量
            this.update();
            Dep.target = null;
        }
        update(){
            let _val = this.get();
            this.node.value = _val; // input
            this.node.nodeValue = _val; // 文本元素
        }
        get(){
            return this.vm.data[this.name]; // 触发 getter
        }
    }
    
    // 主题发布器（当数据vm.data.key改变，发送通知给订阅者，更新视图）
    class Dep{
        constructor(){
            this.subs = [];
        }
        // 发出通知，更新视图
        notify(){
            this.subs.forEach( (sub) => {
                sub.update();
            })
        }
        // 压入 订阅者
        addSub( sub ){
            this.subs.push( sub );
        }
    }


######【监听数据】observe( obj ):  

（1）Object.defineProperty( obj, prop, descriptor )：      

直接在对象上定义一个新属性，或修改对象上的现有属性，并返回该对象。       

&gt; obj：要在其上 定义属性的对象      
 
&gt; prop：要 定义/修改 的属性的名称       

&gt; descriptor：定义/修改 属性的描述符（包括：数据描述符、访问器描述符）      
 
&gt; 访问器描述符：get、set       

（2）观测对象的 所有属性，修改对象属性的 getter、setter。        

（3）对象的 所有属性，都有一个独立的 主题对象。       

（4）***整个 订阅发布模式 流程：***      

&gt; vue编译时，对 `{{ key }}`文本元素、input元素，添加订阅者：`new Watcher()`。        

&gt; `new Watcher()` 时，设置 全局变量`Dep.target`为当前订阅者，然后初始化 执行`update()`，触发 `vm.data.key`的 getter。     

&gt; `vm.data.key`的 getter 检测到 `Dep.target` 不为空，将其（订阅者）加入到 自己的主题对象的 `subs[]`。     

&gt; getter 返回了属性值，回到`new Watcher()`，`Dep.target` 设为空。防止多次插入 已插入过的订阅者。     

&gt; 当`vm.data.key`的 setter 被触发，属性的主题对象执行 `dep.notify()`，通知其下所有订阅者（保存在`subs[]`）更新dom。     


    // 【监听数据】
    let defineReactive = ( obj, key, val ) => {
    
        // 每个key，都有自己的 主题发布器对象。
        // 所有订阅者在new时，都会 使对应key触发getter，把自己加入到 对应key的 主题发布器的subs[]；
        // 每个key 触发setter时，更新所有订阅者
        let dep = new Dep();
    
        // 定义 getter、setter（val是形参，属于私有变量）
        Object.defineProperty( obj, key, {
            get(){
                console.log( 'getter:', val );
                if( Dep.target ) dep.addSub( Dep.target ); // 只有在 new订阅者时，才会执行
                return val;
            },
            set( newVal, oldVal ){
                console.log( 'setter:', newVal );
                if( newVal === oldVal ) return;
    
                console.log( dep );
                val = newVal;
                dep.notify(); // 发出通知，更新视图
            }
        })
    };
    
    // 观察者，观测对象 所有属性
    let observe = ( obj ) => {
        Object.keys( obj ).forEach( (key) => {
            defineReactive( obj, key, obj[key] );
        })
    };
    
    
    // vue构造函数
    class Vue{
        constructor( option ){
            this.data = option.data;
            let _id = option.el;
            observe( this.data ); // 监听数据
    
            let _dom = nodeToFragment( document.getElementById( _id ), this ); // 虚拟dom
            document.getElementById( _id ).appendChild( _dom );
        }
    }
    var vm = new Vue({
        el: 'app',
        data: {
            text: 'hello world'
        }
    });



（5）总结：

&gt; 订阅者是 根据程序数据更新dom的。      

&gt; 对象的所有属性，都有一个独立的主题对象。主题对象存放所有 有关的订阅者。              

&gt; 对象的属性 的setter被触发时，就找到其主题对象下的所有订阅者，通知它们更新dom。        
