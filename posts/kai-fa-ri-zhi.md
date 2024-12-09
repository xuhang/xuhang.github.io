---
title: '开发日志'
date: 2024-12-08 11:35:42
tags: [debug,Java]
published: true
hideInList: false
feature: /post-images/kai-fa-ri-zhi.png
isTop: false
---
## Spring AOP 注解

在使用spring框架时,通常用它的aop来记录日志,但在spring mvc采用@Controller注解时,对Controller进行Aop拦截不起作用,原因是该注解的Controller已被spring容器内部代理了.

需要对org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter进行Aop才能起作用.经过多次测试可行.

<!-- more -->

```java
package com.autoabacus.dal.controller;  
import org.aspectj.lang.ProceedingJoinPoint;  
import org.aspectj.lang.annotation.Around;  
import org.aspectj.lang.annotation.Aspect;  
import org.springframework.stereotype.Component;  
@Component  
@Aspect  
public class Aop {  
    public Aop() {  
        System.out.println("Aop");  
    }  
    // @Around("within(org.springframework.web.bind.annotation.support.HandlerMethodInvoker..*)")  
    @Around("execution(* org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter.handle(..))")  
    public Object aa(ProceedingJoinPoint pjp)  throws Throwable   
    {  
        try {  
            Object retVal = pjp.proceed();  
            System.out.println(retVal);  
            return retVal;  
        } catch (Exception e) {  
            System.out.println("异常");  
            return null;  
        }  
    }  
}
```

Controller:

```java
package com.autoabacus.dal.controller;  
import org.springframework.stereotype.Controller;  
import org.springframework.web.bind.annotation.RequestMapping;  
@Controller    
public class WelcomeController {  
    @RequestMapping  
    public void welcome() {  
        if (true)  
            throw new RuntimeException("fdsafds");  
    }    
}  
```

## jQuery 自定义选择器

```jquery
$(document).ready(function() {
    $.extend($.expr[':'], {
        moreThen1000px: function(a) {
            return $(a).width() > 1000;
        }
    });
    $('.widthDiv:moreThen1000px').click(function() {
        alert('The element that you have clicked is over 1000 pixels wide');
    });
});
```

## HTML文字自动裁剪

```html
<a style="text-overflow: ellipsis;overflow: hidden;white-space: nowrap;width: 100%;display: inline-block;">文字自动裁剪</a>
```

## FIREFOX图片占位符

```css
@-moz-document url-prefix(http), url-prefix(file) {
    img:-moz-broken{
        -moz-force-broken-image-icon:1;
        width:220px;
        height:220px;
    }
}
```

## JAVASCRIPT轻量模板

```javascript
function tmpl(template, data) {
    return template.replace(/\{([\*\w\.]*)\}/g, function(str, key) {
        var keys = key.split(".");
        var rep = '_300*300';
        //如果有形如“_300*300”的key，则作为图片的大小，并从keys中移除
        for(var i=0; i<keys.length; i++){
            if(/_\d{1,4}\*\d{1,4}/.test(keys[i])){
                rep = keys.splice(i,1);
                break;
            }
        }
        /*if(keys.length>2) {
            var ss = keys[1];
            if(/_\d{1,4}\*\d{1,4}/.test(ss)){
                rep = keys.splice(1,1);
            }
        }*/
        var v = data[keys.shift()];
        for (var i = 0, l = keys.length; i < l; i++) v = v[keys[i]];
        v = (typeof v !== "undefined" && v !== null) ? v : "";
        if(strEndsWith(v.toString(), ".jpg") || strEndsWith(v.toString(), ".JPG") || strEndsWith(v.toString(), ".webp")){
            v = v.replace(/\.(jpg|JPG|webp)$/, rep+"$&");
        }
        return v;
    });
    function strEndsWith(str, suffix) {
        return str.match(suffix+"$")==suffix;
    }
}
```
注释：
```javascript
function tmpl(template, data) {
    return template.replace(/\{([\w\.]*)\}/g, function(str, key, idx, ori) {//str-匹配正则，key-匹配括号内，idx-匹配正则的str在原字符串中的位置，ori-原始字符串
        var keys = key.split(".");
        //shift把数组的第一个元素从其中删除，并返回第一个元素的值
        var v = data[keys.shift()];
        for (var i = 0, l = keys.length; i < l; i++) v = v[keys[i]];
        v = (typeof v !== "undefined" && v !== null) ? v : "";
        return v;
    });
}
```

## 在浏览器中调试异步加载的js脚本

如果页面在请求之后返回的是需要插入在原页面中的代码，这些代码通常除了啦HTML代码还会带着一些JS脚本。这些脚本通常不会在Source栏中显示，导致调试这些脚本会比较麻烦，除了在代码中插入`debugger;`还可以在脚本的最后（`</script>`之前）加上标签`//# sourceURL=xxx.js`，这样在浏览器的调试窗口选择打开文件（快捷键`Ctrl + P`）输入xxx代表的名字即可打开调试。

## Spring中注入枚举类型的属性值

```xml
<bean id="databaseId" class="java.lang.Integer">
    <constructor-arg type="int" value="#{T(com.yhd.kaleido.kedis.domain.KedisPool).${redis.databasePool}.id}" />
</bean>
```

## ConcurrentModificationException

`foreach loop`调用的其实是`Iterator`迭代器，`Iterator`内部维护`expectedModCount`，`ArrayList`维护`modCount`。只有调用`Iterator`的`remove`方法会保证`expectedModCount`和`modCount`会相等，而调用`ArrayList`的`remove`方法只修改`modCount`，`foreach loop`遍历元素时调用`next()`，会触发`checkForComodification()`，此方法中`expectedModCount！=modCount`会抛出`ConcurrentModificationException`。 

## Linux查看连接的创建时间

查看ssh到ip地址221.234.44.52连接的创建时间：

1. `netstat -npt | grep 221.234.44.52  `
2. 记录结果中的pid和port
3. 查看这个进程打开的这个连接的文件名：`lsof -p PID | grep PORT `
4. 结果中有表示进程在这个端口上的连接的文件编号 ：xxxu
5. 查看文件的创建时间：`ll /proc/PID/fd/xxx`
6. 结果格式类似`lrwx------ 1 root root 64 Jul 28 14:38 170 -> socket:[788102217]`
7. 查看socket对应inode信息`netstat -altp | egrep -i 'Inode|788102217'`

## Linux中标准输入输出

标准输入0    从键盘获得输入 /proc/self/fd/0 

标准输出1    输出到屏幕（即控制台） /proc/self/fd/1 

错误输出2    输出到屏幕（即控制台） /proc/self/fd/2 

 

/dev/null代表linux的空设备文件，所有往这个文件里面写入的内容都会丢失，俗称“黑洞” 

+ `2>/dev/null`意思就是把错误输出到“黑洞” 
+ `>/dev/null 2>&1`默认情况是1，也就是等同于1>/dev/null 2>&1。意思就是把标准输出重定向到“黑洞”，还把错误输出2重定向到标准输出1，也就是标准输出和错误输出都进了“黑洞” 
+ `2>&1 >/dev/null`意思就是把错误输出2重定向到标准出书1，也就是屏幕，标准输出进了“黑洞”，也就是标准输出进了黑洞，错误输出打印到屏幕 

关于这里”&”的作用，我们可以这么理解2>/dev/null重定向到文件，那么2>&1，这里如果去掉了&就是把错误输出给了文件1了，用了&是表明1是标准输出。

## Linux 设置命令执行超时时间

```bash
command & pid=$!;sleep 2;kill -9 $pid
```

## JS可拖拽元素

``` javascript
function pauseEvent(e) {
    if (e.stopPropagation) e.stopPropagation();
    if (e.preventDefault) e.preventDefault();
    e.cancelBubble = true;
    e.returnValue = false;
    return false;
}
let dragableTar;
let winHeight = window.screen.height, winWidth = window.screen.width;
let startX,
    startY,
    currentLeft,
    currentTop,
    flag = 0;
document.body.onmousedown = function (e) {
    let tar = e.target;
    if (tar.tagName == 'INPUT' || tar.tagName == 'A') {
        return;
    }

    pauseEvent(e);
    while (tar.parentNode) {
        tar = tar.parentNode;
        if (tar == document.body) {
            break;
        }
        if (tar.className && tar.className.indexOf('dragable') >= 0) {
            dragableTar = tar;
            startX = e.clientX;
            startY = e.clientY;
            currentLeft = dragableTar.offsetLeft;　　//通过对象获取对象的坐标
            currentTop = dragableTar.offsetTop;
            break;
        }
    }
    let callback = (e) => {
        if (dragableTar) {
            let tarWidth = dragableTar.clientWidth||dragableTar.offsetWidth;
            let tarHeight = dragableTar.clientHeight||dragableTar.offsetHeight;

            let x = e.clientX;　　　　　　　　//e.clientX是一个触摸事件，即是鼠标点击时的X轴上的坐标
            let y = e.clientY;　　　　　　　　//e.clientY是一个触摸事件，即是鼠标点击时的Y轴上的坐标
            let distanceX = x - startX;　　　　//X轴上获得移动的实际距离
            let distanceY = y - startY;　　　　　//Y轴上获得移动的实际距离

            let tarLeft = distanceX + currentLeft;
            if (tarLeft >= 0 && tarLeft + tarWidth <= winWidth) {
                dragableTar.style.left = (tarLeft) + "px";　　//currentLeft下面的方法会有解释，需要注意最后要添加PX单位，如果给left赋值会破坏文档流，不加单位就会无效
            }
            let tarTop = distanceY + currentTop;
            if (tarTop >= 0 && tarTop + tarHeight <= winHeight) {
                dragableTar.style.top = (distanceY + currentTop) + "px";
            }
        }
    }

    console.log(dragableTar)
    document.addEventListener('mousemove',callback);
    document.addEventListener('mouseup', () => {
        console.log('鼠标UP')
        document.removeEventListener('mousemove',callback);
        if (tar.id == 'musicController') {
            updateThumbnail(tar.offsetLeft, tar.offsetTop);
        }
        // pauseEvent(e);
        dragableTar = null;
    });
```

## 在MyBatis中引用常量

```xml
${@全类名$内部类@常量名}
${@com.wyouquan.coupon.entity.MarketingCouponDictionary$DeleteStatus@NOT_DELETED}
```

## SpringMVC复合参数注入和Mybatis复合参数使用

接收参数的Bean定义

```java
public class Page {
    private long total;
    private Map<String, Object> paramMap;
    private int limit;
    private int offset;
    private String orderBy;
    ...
}
```

要传递limit和offset的同时，还要往paramMap里注入参数

request请求头的`contentType`值为`application/x-www-form-urlencoded`

传递的JSON参数值为

```json
{
    'limit': 10,
    'offset': 0,
    'paramMap[type]': 1,
    'orderBy': 'name'
}
```

这样Controller层接收到的参数Page中的Map里就包含了指定的值。

在Dao层使用复合参数Page根据对象的使用规则，用`.`调用即可

```xml
SELECT <include refid="Base_Column_List"/>
FROM marketing_coupons_dictionary
<trim prefix="WHERE" prefixOverrides="AND |OR ">
    <if test="page.paramMap.type != null">
        dictionary_type = #{page.paramMap.type} AND
    </if>
    is_delete = ${@com.wyouquan.coupon.entity.MarketingCouponDictionary$DeleteStatus@NOT_DELETED}
</trim>
<if test="page.orderBy != null">
    ORDER BY #{page.orderBy}
</if>
<if test="page.offset != null and page.limit != null">
    LIMIT #{page.offset}, #{page.limit}
</if>
```

## Shardingsphere 3.1.0分页Bug

ShardingSphere在3.1.0的版本中，对没有配置分表的查询，limit查询会默认把前N条记录都查询出来再做处理，比如limit 20, 10，shardingsphere会把所有分表的前30条记录都查询出来，再整合数据选择10条，但是对于没有配置分表的查询，默认返回30条记录。这一bug在4.0.0-RC1修复，但是4.0.0之前的版本的package是`shardingsphere.io`，4.0.0的版本package是`org.apache.shardingsphere`，并且命名空间的schema文件也有相应的变化，如果IDEA报找不到命名空间的文件错误，需要额外导入namespace的包。

```xml
<!-- ShardingSphere包 -->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>sharding-jdbc-core</artifactId>
    <version>${shardingsphere.version}</version>
</dependency>
<!-- 命名空间的包 -->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>sharding-jdbc-spring-namespace</artifactId>
    <version>${shardingsphere.version}</version>
</dependency>
```

## MySQL 8.x登陆失败`caching_sha2_password`

8.0.4版本以上，mysql默认授权插件改成了caching_sha2_password模式，所以实际设置的密码是被转换过的。用如下方法解决问题：

命令行登陆MySQL之后，执行命令：`ALTER USER 'username'@'ip_address' IDENTIFIED WITH mysql_native_password BY 'password';`

比如：`alter user 'root'@'%' identified with mysql_native_password by '123456';`

修改之后即可登陆。

## MySQL查询分组的前N条记录

案例：查询学生成绩表`stu`中，各班（`class`）各科目（`subject`）的前10名：

```mysql
SELECT * FROM `stu` AS a WHERE 10 > (SELECT COUNT(*) FROM `stu` WHERE class = a.class AND `subject` = a.`subject` AND score < a.score) ORDER BY a.class ASC,a.subject ASC,a.score DESC
```

总结：查询表中，分组1（各班）、分组2下（各科目），按条件（成绩）降序的前N（10）名：

```mysql
SELECT * FROM 表 AS a WHERE N > (SELECT COUNT(*) FROM 表 WHERE 分组1=a.分组1 AND 分组2=a.分组2 AND 条件 < a.条件) ORDER BY a.分组1, a.分组2, a.条件 DESC
```

## JAVASCRIPT变量作用域

```javascript
var a = 10;
function foo() {
    console.log(a);
    var a = 20;
}
foo();
```

上述代码打印值为`undefined`，原因是`var`关键字声明的变量会被提升，并在内存中分配值`undefined`，具体初始化是发生在赋值的地方。另外`var`声明的变量作用域是函数基本的，`let`和`const`是块级别的。

所以在执行上述foo()函数时，在打印变量`a`时，函数内部用`var`声明变量，会在打印之前局部变量会提升并分配`undefined`，执行到`var a = 20`时才会分配具体值。

`let`和`const`声明的变量不会被提升，只有执行到声明的地方才能访问，提前访问会抛出`ReferenceError`异常。下面的代码会抛出`ReferenceError`异常：

```javascript
var a = 10;
function foo() {
    console.log(a);
    let a = 20;
}
foo();
```

## Python图片字符识别tesserocr异常`RuntimeError: Failed to init API, possibly an invalid tessdata path: D:\Python36\/tessdata/`

```python
import tesserocr
from PIL import Image
image = Image.open('C:\\Users\\Administrator\\Desktop\\a.jpg')
print(tesserocr.image_to_text(image))
```

将Tesseract-OCR安装路径下的`tessdata`文件夹拷贝到python的安装路径下即可。

## java程序执行会启动的线程

```java
public  static Thread[] findAllThread(){
    ThreadGroup currentGroup =Thread.currentThread().getThreadGroup();
    while (currentGroup.getParent()!=null){
        // 返回此线程组的父线程组
        currentGroup=currentGroup.getParent();
    }
    //此线程组中活动线程的估计数
    int noThreads = currentGroup.activeCount();
    Thread[] lstThreads = new Thread[noThreads];
    //把对此线程组中的所有活动子组的引用复制到指定数组中。
    currentGroup.enumerate(lstThreads);
    for (Thread thread : lstThreads) {
        System.out.println("线程数量："+noThreads+" " +
                "线程id：" + thread.getId() + 
                " 线程名称：" + thread.getName() + 
                " 线程状态：" + thread.getState());
    }
    return lstThreads;
}
```

输出：

```
线程数量：6 线程id：2 线程名称：Reference Handler 线程状态：WAITING
线程数量：6 线程id：3 线程名称：Finalizer 线程状态：WAITING
线程数量：6 线程id：4 线程名称：Signal Dispatcher 线程状态：RUNNABLE
线程数量：6 线程id：5 线程名称：Attach Listener 线程状态：RUNNABLE
线程数量：6 线程id：1 线程名称：main 线程状态：RUNNABLE
线程数量：6 线程id：6 线程名称：Monitor Ctrl-Break 线程状态：RUNNABLE
```

| 线程 | **所属**     | **说明**     |
| ------------------- | ------------ | ------------------------- |
| ==Attach Listener==| JVM          | Attach Listener线程是负责接收到外部的命令，而对该命令进行执行的并且吧结果返回给发送者。通常我们会用一些命令去要求jvm给我们一些反馈信息，如：java -version、jmap、jstack等等。如果该线程在jvm启动的时候没有初始化，那么，则会在用户第一次执行jvm命令时，得到启动。 |
| ==Signal Dispatcher== | JVM          | 前面我们提到第一个Attach Listener线程的职责是接收外部jvm命令，当命令接收成功后，会交给signal dispather线程去进行分发到各个不同的模块处理命令，并且返回处理结果。signal dispather线程也是在第一次接收外部jvm命令时，进行初始化工作。 |
| CompilerThread0  | JVM          | 用来调用JITing，实时编译装卸class。通常，jvm会启动多个线程来处理这部分工作，线程名称后面的数字也会累加，例如：CompilerThread1 |
| ConcurrentMark-SweepGCThread| JVM          | 并发标记清除垃圾回收器（就是通常所说的CMS GC）线程，该线程主要针对于老年代垃圾回收。ps：启用该垃圾回收器，需要在jvm启动参数中加上：-XX:+UseConcMarkSweepGC |
| DestroyJavaVM| JVM          | 执行main()的线程在main执行完后调用JNI中的jni_DestroyJavaVM()方法唤起DestroyJavaVM线程。  JVM在Jboss服务器启动之后，就会唤起DestroyJavaVM线程，处于等待状态，等待其它线程（java线程和native线程）退出时通知它卸载JVM。线程退出时，都会判断自己当前是否是整个JVM中最后一个非deamon线程，如果是，则通知DestroyJavaVM线程卸载JVM。ps：扩展一下：1.如果线程退出时判断自己不为最后一个非deamon线程，那么调用thread->exit(false)，并在其中抛出thread_end事件，jvm不退出。2.如果线程退出时判断自己为最后一个非deamon线程，那么调用before_exit()方法，抛出两个事件： 事件1：thread_end线程结束事件、事件2：VM的death事件。然后调用thread->exit(true)方法，接下来把线程从active list卸下，删除线程等等一系列工作执行完成后，则通知正在等待的DestroyJavaVM线程执行卸载JVM操作。 |
| ContainerBackgroundProcessor| JBOSS        | 它是一个守护线程,在jboss服务器在启动的时候就初始化了,主要工作是定期去检查有没有Session过期.过期则清除.参考：http://liudeh-009.iteye.com/blog/1584876 |
| ConfigClientNotifier| ConfigServer | ConfigServer服务端当有配置变更时，就会将最新的配置推送到ConfigServer客户端的一个数据列队中，ConfigClientNotifier线程用于定期检查该数据列队中是否有数据，如果有数据，则将数据分发到订阅该数据的组件去做业务逻辑，比如：tair和hsf的数据都订阅了ConfigServer数据源,当ConfigClientNotifier线程发现数据有更新时，就触发做数据分发特定特定信号标识将数据分发到相应的订阅者。 |
| ConfigClientWorker-Default| ConfigServer | 包括主动向服务器端发送数据（主要是订阅和发布的数据）和接收服务器推送过来的数据（主要是订阅数据的值） 。 |
| Dispatcher-Thread-3| Log4j        | Log4j具有异步打印日志的功能，需要异步打印日志的Appender都需要注册到AsyncAppender对象里面去，由AsyncAppender进行监听，决定何时触发日志打印操作。AsyncAppender如果监听到它管辖范围内的Appender有打印日志的操作，则给这个Appender生成一个相应的event，并将该event保存在一个buffuer区域内。 Dispatcher-Thread-3线程负责判断这个event缓存区是否已经满了，如果已经满了，则将缓存区内的所有event分发到Appender容器里面去，那些注册上来的Appender收到自己的event后，则开始处理自己的日志打印工作。Dispatcher-Thread-3线程是一个守护线程。 |
| ==Finalizer线程==| JVM          | 这个线程也是在main线程之后创建的，其优先级为10，主要用于在垃圾收集前，调用对象的finalize()方法；关于Finalizer线程的几点：1)只有当开始一轮垃圾收集时，才会开始调用finalize()方法；因此并不是所有对象的finalize()方法都会被执行；2)该线程也是daemon线程，因此如果虚拟机中没有其他非daemon线程，不管该线程有没有执行完finalize()方法，JVM也会退出；3) JVM在垃圾收集时会将失去引用的对象包装成Finalizer对象（Reference的实现），并放入ReferenceQueue，由Finalizer线程来处理；最后将该Finalizer对象的引用置为null，由垃圾收集器来回收；4) JVM为什么要单独用一个线程来执行finalize()方法呢？如果JVM的垃圾收集线程自己来做，很有可能由于在finalize()方法中误操作导致GC线程停止或不可控，这对GC线程来说是一种灾难； |
| Gang worker#0| JVM          | JVM用于做新生代垃圾回收（monir gc）的一个线程。#号后面是线程编号，例如：Gang worker#1 |
| GC Daemon | JVM          | GC Daemon线程是JVM为RMI提供远程分布式GC使用的，GC Daemon线程里面会主动调用System.gc()方法，对服务器进行Full GC。 其初衷是当RMI服务器返回一个对象到其客户机（远程方法的调用方）时，其跟踪远程对象在客户机中的使用。当再没有更多的对客户机上远程对象的引用时，或者如果引用的“租借”过期并且没有更新，服务器将垃圾回收远程对象。不过，我们现在jvm启动参数都加上了-XX:+DisableExplicitGC配置，所以，这个线程只有打酱油的份了。 |
| IdleRemover | JBOSS        | Jboss连接池有一个最小值，该线程每过一段时间都会被Jboss唤起，用于检查和销毁连接池中空闲和无效的连接，直到剩余的连接数小于等于它的最小值。 |
| Java2D Disposer| JVM          | 这个线程主要服务于awt的各个组件。说起该线程的主要工作职责前，需要先介绍一下Disposer类是干嘛的。Disposer提供一个addRecord方法。如果你想在一个对象被销毁前再做一些善后工作，那么，你可以调用Disposer#addRecord方法，将这个对象和一个自定义的DisposerRecord接口实现类，一起传入进去，进行注册。Disposer类会唤起“Java2D Disposer”线程，该线程会扫描已注册的这些对象是否要被回收了，如果是，则调用该对象对应的DisposerRecord实现类里面的dispose方法。Disposer实际上不限于在awt应用场景，只是awt里面的很多组件需要访问很多操作系统资源，所以，这些组件在被回收时，需要先释放这些资源。 |
| FelixDispatchQueue | Sofa         | 该线程会在sofa启动时会唤起该线程,该线程用于分发OSGI事件到Declarative Services 中去发布，查找，绑定目标服务。其实，我们接口配置的service和reference就涉及到服务的发布、查找和绑定工作。  Declarative Services 主要工作职责是方便地对服务之间的依赖关系和状态进行监听和管理。OSGI使用事件策略去调用Declarative Services中的服务。 |
| InsttoolCacheScheduler_QuartzSchedulerThread| Quartz       | InsttoolCacheScheduler_QuartzSchedulerThread是Quartz的主线程，它主要负责实时的获取下一个时间点要触发的触发器，然后执行触发器相关联的作业。原理大致如下：Spring和Quartz结合使用的场景下，Spring IOC容器初始化时会创建并初始化Quartz线程池（TreadPool），并启动它。刚启动时线程池中每个线程都处于等待状态，等待外界给他分配Runnable（持有作业对象的线程）。继而接着初始化并启动Quartz的主线程（InsttoolCacheScheduler_QuartzSchedulerThread），该线程自启动后就会处于等待状态。等待外界给出工作信号之后，该主线程的run方法才实质上开始工作。run中会获取JobStore中下一次要触发的作业，拿到之后会一直等待到该作业的真正触发时间，然后将该作业包装成一个JobRunShell对象（该对象实现了Runnable接口，其实看是上面TreadPool中等待外界分配给他的Runnable），然后将刚创建的JobRunShell交给线程池，由线程池负责执行作业。线程池收到Runnable后，从线程池一个线程启动Runnable，反射调用JobRunShell中的run方法，run方法执行完成之后，TreadPool将该线程回收至空闲线程中。 |
| InsttoolCacheScheduler_Worker-2| Quartz       | InsttoolCacheScheduler_Worker-2线程就是ThreadPool线程的一个简单实现，它主要负责分配线程资源去执行InsttoolCacheScheduler_QuartzSchedulerThread线程交给它的调度任务（也就是JobRunShell）。 |
| java.util.concurrent.ThreadPoolExecutor$Worker| JVM          |                                                              |
| JBossLifeThread| Jboss        | Jboss主线程启动成功，应用程序部署完毕之后将JBossLifeThread线程实例化并且start，JBossLifeThread线程启动成功之后就处于等待状态，以保持Jboss Java进程处于存活中。 所得比较通俗一点，就是Jboss启动流程执行完毕之后，为什么没有结束？就是因为有这个线程hold主了它。牛b吧～～ |
| JBoss System Threads(1)-1| Jboss        | 该线程是一个socket服务，默认端口号为：1099。主要用于接收外部naming service（Jboss JNDI）请求。 |
| JCA PoolFiller| Jboss        | 该线程主要为JBoss内部提供连接池的托管。 简单介绍一下工作原理 ：Jboss内部凡是有远程连接需求的类，都需要实现ManagedConnectionFactory接口，例如需要做JDBC连接的XAManagedConnectionFactory对象，就实现了该接口。然后将XAManagedConnectionFactory对象，还有其它信息一起包装到InternalManagedConnectionPool对象里面，接着将InternalManagedConnectionPool交给PoolFiller对象里面的列队进行管理。  JCA PoolFiller线程会定期判断列队内是否有需要创建和管理的InternalManagedConnectionPool对象，如果有的话，则调用该对象的fillToMin方法，触发它去创建相应的远程连接，并且将这个连接维护到它相应的连接池里面去。 |
| JDWP Event Helper Thread| JVM          | JDWP是通讯交互协议，它定义了调试器和被调试程序之间传递信息的格式。它详细完整地定义了请求命令、回应数据和错误代码，保证了前端和后端的JVMTI和JDI的通信通畅。 该线程主要负责将JDI事件映射成JVMTI信号，以达到调试过程中操作JVM的目的。 |
| JDWP TransportListener: dt_socket| JVM          | 该线程是一个Java Debugger的监听器线程，负责受理客户端的debug请求。通常我们习惯将它的监听端口设置为8787。 |
| Low MemoryDetector| JVM          | 这个线程是负责对可使用内存进行检测，如果发现可用内存低，分配新的内存空间。 |
| process reaper| JVM          | 该线程负责去执行一个OS命令行的操作。                         |
| ==Reference Handler==| JVM          | JVM在创建main线程后就创建Reference Handler线程，其优先级最高，为10，它主要用于处理引用对象本身（软引用、弱引用、虚引用）的垃圾回收问题。 |
| SurrogateLockerThread(CMS)| JVM          | 这个线程主要用于配合CMS垃圾回收器使用，它是一个守护线程，其主要负责处理GC过程中，Java层的Reference（指软引用、弱引用等等）与jvm内部层面的对象状态同步。这里对它们的实现稍微做一下介绍：这里拿WeakHashMap做例子，将一些关键点先列出来（我们后面会将这些关键点全部串起来）：1.   我们知道HashMap用Entry[]数组来存储数据的，WeakHashMap也不例外,内部有一个Entry[]数组。2.    WeakHashMap的Entry比较特殊，它的继承体系结构为Entry->WeakReference->Reference。3.   Reference里面有一个全局锁对象：Lock，它也被称为pending_lock.  注意：它是静态对象。4.   Reference 里面有一个静态变量：pending。5. Reference 里面有一个静态内部类：ReferenceHandler的线程，它在static块里面被初始化并且启动，启动完成后处于wait状态，它在一个Lock同步锁模块中等待。6.   另外，WeakHashMap里面还实例化了一个ReferenceQueue列队，这个列队的作用，后面会提到。7.   上面关键点就介绍完毕了，下面我们把他们串起来。  假设，WeakHashMap对象里面已经保存了很多对象的引用。JVM在进行CMS GC的时候，会创建一个ConcurrentMarkSweepThread（简称CMST）线程去进行GC，ConcurrentMarkSweepThread线程被创建的同时会创建一个SurrogateLockerThread（简称SLT）线程并且启动它，SLT启动之后，处于等待阶段。CMST开始GC时，会发一个消息给SLT让它去获取Java层Reference对象的全局锁：Lock。直到CMS GC完毕之后，JVM会将WeakHashMap中所有被回收的对象所属的WeakReference容器对象放入到Reference的pending属性当中（每次GC完毕之后，pending属性基本上都不会为null了），然后通知SLT释放并且notify全局锁:Lock。此时激活了ReferenceHandler线程的run方法，使其脱离wait状态，开始工作了。ReferenceHandler这个线程会将pending中的所有WeakReference对象都移动到它们各自的列队当中，比如当前这个WeakReference属于某个WeakHashMap对象，那么它就会被放入相应的ReferenceQueue列队里面（该列队是链表结构）。当我们下次从WeakHashMap对象里面get、put数据或者调用size方法的时候，WeakHashMap就会将ReferenceQueue列队中的WeakReference依依poll出来去和Entry[]数据做比较，如果发现相同的，则说明这个Entry所保存的对象已经被GC掉了，那么将Entry[]内的Entry对象剔除掉。 |
| taskObjectTimerFactory| JVM          | 顾名思义，该线程就是用来执行任务的。当我们把一个任务交给Timer对象，并且告诉它执行时间，周期时间后，Timer就会将该任务放入任务列队，并且通知taskObjectTimerFactory线程去处理任务，taskObjectTimerFactory线程会将状态为取消的任务从任务列队中移除，如果任务是非重复执行类型的，则在执行完该任务后，将它从任务列队中移除，如果该任务是需要重复执行的，则计算出它下一次执行的时间点。 |
| VM Periodic Task Thread| JVM          | 该线程是JVM周期性任务调度的线程，它由WatcherThread创建，是一个单例对象。该线程在JVM内使用得比较频繁，比如：定期的内存监控、JVM运行状况监控，还有我们经常需要去执行一些jstat这类命令查看gc的情况，如下：jstat -gcutil 23483 250 7 这个命令告诉jvm在控制台打印PID为：23483的gc情况，间隔250毫秒打印一次，一共打印7次。 |
| VM Thread| JVM          | 这个线程就比较牛b了，是jvm里面的线程母体，根据hotspot源码（vmThread.hpp）里面的注释，它是一个单例的对象（最原始的线程）会产生或触发所有其他的线程，这个单个的VM线程是会被其他线程所使用来做一些VM操作（如，清扫垃圾等）。在 VMThread的结构体里有一个VMOperationQueue列队，所有的VM线程操作(vm_operation)都会被保存到这个列队当中，VMThread本身就是一个线程，它的线程负责执行一个自轮询的loop函数(具体可以参考：VMThread.cpp里面的void VMThread::loop())，该loop函数从VMOperationQueue列队中按照优先级取出当前需要执行的操作对象(VM_Operation)，并且调用VM_Operation->evaluate函数去执行该操作类型本身的业务逻辑。ps：VM操作类型被定义在vm_operations.hpp文件内，列举几个：ThreadStop、ThreadDump、PrintThreads、GenCollectFull、GenCollectFullConcurrent、CMS_Initial_Mark、CMS_Final_Remark…..有兴趣的同学，可以自己去查看源文件。 |

## Markdown画类图

~~~markdown
``` mermaid
classDiagram
  classA --|> classB : 继承
  classC --* classD : 组成
  classE --o classF : 集合
  classG --> classH : 关联
  classI -- classJ : 实线连接
  classK ..> classL : 依赖
  classM ..|> classN : 实现
  classO .. classP : 虚线连接
```
~~~

``` mermaid
classDiagram
  classA --|> classB : 继承
  classC --* classD : 组成
  classE --o classF : 集合
  classG --> classH : 关联
  classI -- classJ : 实线连接
  classK ..> classL : 依赖
  classM ..|> classN : 实现
  classO .. classP : 虚线连接
```

---



``` mermaid
classDiagram
	Bird --|> Animal: 继承
	Wing "2" --> "1" Bird: 组合
	Animal ..> Oxygen: 依赖
	Animal ..> Water: 依赖
	class Animal {
		<<interface>>
		+ Life life
		+ Breathe(Oxygen)
		+ Reproduction()
	}
	class Bird {
		+ feature
		+ beak
		+ layEgg() Egg
	}
```

## C语言printf的占位符

1. 宽度限定

   `printf("%5d\n", 123)`，默认==右对齐==，长度为5，前面补2位空格

   `printf("%-5d\n", 123)`，长度前加个减号-，改为左对齐，后面补2位空格

2. 显示正负号

   负数前面总是显示减号-，正数默认不显示加号+，`printf("%+d\n", 12)`则会显示加号+

3. 小数宽度

   `printf("%.2f\n", 2.7814)`表示保留2位有效小数

   `printf("%10.4f\n", 3.14)`表示小数整体长度为10位，小数部分为4为，整数部分为5位（小数点占1位），左对齐

   `printf("%*.*f\n", 10, 4, 3.14)`中10和4替换占位符中的星号*，效果同上

4. 切割字符

   `printf("%.5s", "Hello World")`输出字符串Hello World的前5个字符

## C语言的for循环作用域

`for`的循环条件部分是一个单独的作用域，跟循环体内部不是同一个作用域。

```c
for (int i = 0; i < 5; i++) {
  int i = 999;
  printf("%d\n", i);
}

printf("%d\n", i); // 非法
```

上面示例中，`for`的循环变量是`i`，循环体内部也声明了一个变量`i`，会优先读取。但由于循环条件部分是一个单独的作用域，所以循环体内部的`i`不会修改掉循环变量`i`，因此这段代码的运行结果就是打印5次`999`。

## MacOS审查WebViews

输入命令

```shell
defaults write [NSGlobalDomain替换成对应的全局命名空间] WebkitDeveloperExtras -bool true
defaults write -g WebkitDeveloperExtras -bool YES
```

重启电脑之后就可以在MacOS上像调试网页一样审核元素（右键）。

## Java Web应用中查找资源的路径

在Java Web应用中通过`ServletContext.getResource(String path)`或者`ServletContext.getResourceAsStream(String path)`获取资源，需要传入一个以`/`开头的字符串，该路径是相对于上下文的根路径，或者Web应用的`WEB-INF/lib`目录下的jar包中的`META-INF/resources`目录。但是`WEB-INF/lib`下jar包的查找顺序是不确定的。

## 正则匹配Emoji

```javascript
 /(\ud83c[\udf00-\udfff])|(\ud83d[\udc00-\ude4f\ude80-\udeff])|[\u2600-\u2B55]/
```

这是一部分Emoji的Unicode范围值，但是Emoji是不断在新增的，所以上面的正则对于一些较新的Emoji是无法匹配的。

类似`\d`,`\W`,`\s`这类转义字符，`\p`和`\P`也是一个特殊Unicode转义字符，表示**一组**Unicode字符，使用时在后面加上`/u`启动Unicode感知模式。

`/\p{Hex_Digit}/u` 表示匹配十六进制的字符。

`/\p{Extended_Pictographic}/u` 表示Emoji表情

`/\p{Emoji}/u` 虽然指定的Emoji，但是`123456789*#`这些值也会被认为是Emoji。