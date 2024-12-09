---
title: 'Chrome 插件开发'
date: 2024-12-08 11:16:11
tags: [chrome 插件,JavaScript]
published: true
hideInList: false
feature: /post-images/chrome-cha-jian-kai-fa.png
isTop: false
---
## Manifest
Chrome扩展的Manifest必须包含`name`、`version`和`manifest_version`属性，对于应用来说，还必须包含`app`属性。
<!-- more -->

Manifest.json模板：
```javascript
{
    "app": {
        "background": {
            "scripts": ["background.js"]
        }
    },
    "manifest_version": 2,
    "name": "My Extension",
    "version": "versionString",
    "default_locale": "en",
    "description": "A plain text description",
    "icons": {
        "16": "images/icon16.png",
        "48": "images/icon48.png",
        "128": "images/icon128.png"
    },
    "browser_action": {
        "default_icon": {
            "19": "images/icon19.png",
            "38": "images/icon38.png"
        },
        "default_title": "Extension Title",
        "default_popup": "popup.html"
    },
    "page_action": {
        "default_icon": {
            "19": "images/icon19.png",
            "38": "images/icon38.png"
        },
        "default_title": "Extension Title",
        "default_popup": "popup.html"
    },
    "background": {
        "scripts": ["background.js"]
    },
    "content_scripts": [
        {
            "matches": ["http://www.google.com/*"],
            "css": ["mystyles.css"],
            "js": ["jquery.js", "myscript.js"]
        }
    ],
    "options_page": "options.html",
    "permissions": [
        "*://www.google.com/*"
    ],
    "web_accessible_resources": [
        "images/*.png"
    ]
}
```

通过Chrome扩展我们可以对用户当前浏览的页面进行操作，实际上就是对用户当前浏览页面的DOM进行操作。通过Manifest中的`content_scripts`属性可以指定将哪些脚本何时注入到哪些页面中，当用户访问这些页面后，相应脚本即可自动运行，从而对页面DOM进行操作。

Manifest的content_scripts属性值为数组类型，数组的每个元素可以包含matches、exclude_matches、css、js、run_at、all_frames、include_globs和exclude_globs等属性。其中matches属性定义了哪些页面会被注入脚本，exclude_matches则定义了哪些页面不会被注入脚本，css和js对应要注入的样式表和JavaScript，run_at定义了何时进行注入，all_frames定义脚本是否会注入到嵌入式框架中，include_globs和exclude_globs则是全局URL匹配，最终脚本是否会被注入由matches、exclude_matches、include_globs和exclude_globs的值共同决定。简单的说，如果URL匹配mathces值的同时也匹配include_globs的值，会被注入；如果URL匹配exclude_matches的值或者匹配exclude_globs的值，则不会被注入。

## 跨域请求：
| URL                                      | 说明      | 是否允许请求 |
| ---------------------------------------- | ------- | ------ |
| http://a.example.com/<br/>http://a.example.com/a.txt | 同域下     | 允许     |
| http://a.example.com/<br/>http://a.example.com/b/a.txt | 同域下不同目录 | 允许     |
| http://a.example.com/<br/>http://a.example.com:8080/a.txt | 同域下不同端口 | 不允许    |
| http://a.example.com/<br/>https://a.example.com/a.txt | 同域下不同协议 | 不允许    |
| http://a.example.com/<br/>http://b.example.com/a.txt | 不同域下    | 不允许    |
| http://a.example.com/<br/>http://a.foo.com/a.txt | 不同域下    | 不允许    |

Google允许Chrome扩展应用不必受限于跨域限制。但出于安全考虑，需要在Manifest的`permissions`属性中声明需要跨域的权限。
```javascript
{
    ...
    "permissions": [
        "*://*.wikipedia.org/*"
    ]
}
```

## 常驻后台
在Manifest中指定background域可以使扩展常驻后台。background可以包含三种属性，分别是scripts、page和persistent。如果指定了scripts属性，则Chrome会在扩展启动时自动创建一个包含所有指定脚本的页面；如果指定了page属性，则Chrome会将指定的HTML文件作为后台页面运行。通常我们只需要使用scripts属性即可，除非在后台页面中需要构建特殊的HTML——但一般情况下后台页面的HTML我们是看不到的。persistent属性定义了常驻后台的方式——当其值为true时，表示扩展将一直在后台运行，无论其是否正在工作；当其值为false时，表示扩展在后台按需运行，这就是Chrome后来提出的Event Page。Event Page可以有效减小扩展对内存的消耗，如非必要，请将persistent设置为false。persistent的默认值为true。

如果想在用户打开浏览器之前就让扩展运行，可以在Manifest的permissions属性中加入"background"，但除非必要，否则尽量不要这么做，因为大部分用户不喜欢这样。

## 带选项页面的扩展
有一些扩展允许用户进行个性化设置，这样就需要向用户提供一个选项页面。Chrome通过Manifest文件的options_page属性为开发者提供了这样的接口，可以为扩展指定一个选项页面。当用户在扩展图标上点击右键，选择菜单中的“选项”后，就会打开这个页面。

## 扩展页面间的通信
有时需要让扩展中的多个页面之间，或者不同扩展的多个页面之间相互传输数据，以获得彼此的状态。比如音乐播放器扩展，当用户鼠标点击popup页面中的音乐列表时，popup页面应该将用户这个指令告知后台页面，之后后台页面开始播放相应的音乐。

Chrome提供了4个有关扩展页面间相互通信的接口，分别是`runtime.sendMessage`、`runtime.onMessage`、`runtime.connect`和`runtime.onConnect`。

> Chrome提供的大部分API是不支持在content_scripts中运行的，但runtime.sendMessage和runtime.onMessage可以在content_scripts中运行，所以扩展的其他页面也可以同content_scripts相互通信。

`chrome.runtime.sendMessage(extensionId, message, options, callback)` : extensionId为所发送消息的目标扩展，如果不指定这个值，则默认为发起此消息的扩展本身，message为要发送的内容，options为对象类型，callback是回调函数。
`chrome.runtime.onMessage.addListener(callback)`：callback接收到的参数有三个，分别是message、sender和sendResponse，即消息内容、消息发送者相关信息和相应函数。

## 储存数据
### localStorage
`localStorage`是HTML5新增的方法，它允许JavaScript在用户计算机硬盘上永久储存数据（除非用户主动删除）。但localStorage也有一些限制，首先是localStorage和Cookies类似，都有域的限制，运行在不同域的JavaScript无法调用其他域localStorage的数据；其次是单个域在localStorage中存储数据的大小通常有限制（虽然W3C没有给出限制），对于Chrome这个限制是5MB(通过声明unlimitedStorage权限，Chrome扩展和应用可以突破这一限制)；最后localStorage只能储存字符串型的数据，无法保存数组和对象，但可以通过join、toString和JSON.stringify等方法先转换成字符串再储存。
### Chrome存储API
Chrome为扩展应用提供了存储API，以便将扩展中需要保存的数据写入本地磁盘。Chrome提供的存储API可以说是对localStorage的改进，它与localStorage相比有以下区别：

+ 如果储存区域指定为sync，数据可以自动同步；
+ content_scripts可以直接读取数据，而不必通过background页面；
+ 在隐身模式下仍然可以读出之前存储的数据；
+ 读写速度更快；
+ 用户数据可以以对象的类型保存。

> 首先localStorage是基于域名的，这在前面的小节中已经提到过了。而content_scripts是注入到用户当前浏览页面中的，如果content_scripts直接读取localStorage，所读取到的数据是用户当前浏览页面所在域中的。所以通常的解决办法是content_scripts通过runtime.sendMessage和background通信，由background读写扩展所在域（通常是chrome-extension://extension-id/）的localStorage，然后再传递给content_scripts。

使用Chrome存储API必须要在Manifest的permissions中声明"storage"，之后才有权限调用。Chrome存储API提供了2种储存区域，分别是sync和local。两种储存区域的区别在于，sync储存的区域会根据用户当前在Chrome上登陆的Google账户自动同步数据，当无可用网络连接可用时，sync区域对数据的读写和local区域对数据的读写行为一致。

> StorageArea = sync / local

1. `chrome.storage.StorageArea.get(keys, callback)`：keys可以是字符串、包含多个字符串的数组或对象。如果keys是字符串，则和localStorage的用法类似；如果是数组，则相当于一次读取了多个数据；如果keys是对象，则会先读取以这个对象属性名为键值的数据，如果这个数据不存在则返回keys对象的属性值（比如keys为{'name':'Billy'}，如果name这个值存在，就返回name原有的值，如果不存在就返回Billy）。如果keys为一个空数组（[]）或空对象（{}），则返回一个空列表，如果keys为null，则返回所有存储的数据。
2. `chrome.storage.StorageArea.getBytesInUse(keys, callback)`：此处的keys只能为null、字符串或包含多个字符串的数组。
3. `chrome.storage.StorageArea.set(items, callback)`：items为对象类型，形式为键/值对。items的属性值如果是字符型、数字型和数组型，则储存的格式不会改变，但如果是对象型和函数型的，会被储存为“{}”，如果是日期型和正则型的，会被储存为它们的字符串形式。
4. `chrome.storage.StorageArea.remove(keys, callback)`：其中keys可以是字符串，也可以是包含多个字符串的数组。
5. `chrome.storage.StorageArea.clear(callback)`：删除所有数据。

Chrome同时还为存储API提供了一个onChanged事件，当存储区的数据发生改变时，这个事件会被激发。callback会接收到两个参数，第一个为changes，第二个是StorageArea。changes是词典对象，键为更改的属性名称，值包含两个属性，分别为oldValue和newValue；StorageArea为local或sync。

> position属性还有另外的三个值，分别是absolute、relative和fixed。如果元素的位置属性为absolute，则它的位置是相对于除static定位以外的父系元素的，如果没有这样的父系元素，则相对于body；如果元素的位置属性为relative，则它的位置是相对于它默认在HTML流中位置的；如果元素的位置属性为fixed，则它的位置是相对于浏览器窗口的。

## Browser Actions
### 图标
Browser Actions可以在Manifest中设定一个默认的图标
```javascript
"browser_action": {
    "default_icon": {
        "19": "images/icon19.png",
        "38": "images/icon38.png"
    }
}
```
一般情况下，Chrome会选择使用19像素的图片显示在工具栏中，但如果用户正在使用视网膜屏幕的计算机，则会选择38像素的图片显示。两种尺寸的图片并不是必须都指定的，如果只指定一种尺寸的图片，在另外一种环境下，Chrome会试图拉伸图片去适应，这样可能会导致图标看上去很难看。另外，default_icon也不是必须指定的，如果没有指定，Chrome将使用一个默认图标。

通过setIcon方法可以动态更改扩展的图标:`chrome.browserAction.setIcon(details, callback)`,其中details的类型为对象，可以包含三个属性，分别是imageData、path和tabId。

imageData是图片的像素数据，可以通过HTML的canvas标签获取到。
path的值可以是字符串，也可以是对象。如果是对象，结构为{size: imagePath}。imagePath为图片在扩展根目录下的相对位置。
tabId的值限定了浏览哪个标签页时，图标将被更改。
### Popup页面
popup在关闭后，就相当于用户关闭了相应的标签页，这个页面不会继续运行。当用户再次打开这个页面时，所有的DOM和js空间变量都将被重新创建。
#### 使用带有滚动条的DIV容器。

#### [设计一个更好的滚动条样式。][1]

#### 考虑屏蔽右键菜单。

#### 使用外部引用的脚本。
#### 不要在popup页面的js空间变量中保存数据。

### 标题和badge
将鼠标移至扩展图标上，片刻后所显示的文字就是扩展的标题，在Manifest中，browser_action的default_title属性可以设置扩展的默认标题。还可以用JavaScript来动态更改扩展的标题：`chrome.browserAction.setTitle({title: 'This is a new title'});`。

Badge是扩展为用户提供有限信息的另外一种方法，这种方法较标题优越的地方是它可以一直显示，其缺点是只能显示大约4字节长度的信息。Badge目前只能够通过JavaScript设定显示的内容，同时Chrome还提供了更改badge背景的方法。如果不定义badge的背景颜色，默认将使用红色：
```javascript
chrome.browserAction.setBadgeBackgroundColor({color: '#0000FF'});
chrome.browserAction.setBadgeText({text: 'Dog'});
```

## Content Scripts

```json
"content_scripts": [
  {
    //"matches": ["http://*/*", "https://*/*"],
    // "<all_urls>" 表示匹配所有地址
    "matches": ["<all_urls>"],
    // 多个JS按顺序注入
    "js": ["js/jquery-1.8.3.js", "js/content-script.js"],
    // JS的注入可以随便一点，但是CSS的注意就要千万小心了，因为一不小心就可能影响全局样式
    "css": ["css/custom.css"],
    // 代码注入的时间，可选值： "document_start", "document_end", or "document_idle"，最后一个表示页面空闲时，默认document_idle
    "run_at": "document_start"
  }
]
```

如果没有主动指定`run_at`为`document_start`（默认为`document_idle`），下面这种代码不会生效：

``` javascript
document.addEventListener('DOMContentLoaded', function(){
	console.log('我被执行了！');
});
```

content_scripts只能访问下面4种chrome api
- chrome.extension(getURL , inIncognitoContext , lastError , onRequest , sendRequest)
- chrome.i18n
- chrome.runtime(connect , getManifest , getURL , id , onConnect , onMessage , sendMessage)
- chrome.storage

## 通信

参考 [chrome插件开发攻略](https://www.cnblogs.com/liuxianan/p/chrome-plugin-develop.html#blog-comments-placeholder)

### popup和background

>  popup可以直接调用background中的JS方法，也可以直接访问background的DOM

``` javascript
function test(){
    alert('我是background！');
}

// popup.js
var bg = chrome.extension.getBackgroundPage();
bg.test();
```

> background访问popup（popup是显示状态）

``` javascript
var views = chrome.extension.getViews({type:'popup'});
if(views.length > 0) {
    console.log(views[0].location.href);
}
```

### popup和content

> popup和background其实几乎可以视为一种东西，因为它们可访问的API都一样、通信机制一样、都可以跨域。

### background和content

``` javascript
// background或popup
function sendMessageToContentScript(message, callback){
    chrome.tabs.query({active: true, currentWindow: true}, function(tabs)
    {
        chrome.tabs.sendMessage(tabs[0].id, message, function(response)
        {
            if(callback) callback(response);
        });
    });
}
sendMessageToContentScript({cmd:'test', value:'你好，我是popup！'}, function(response){
    console.log('来自content的回复：'+response);
});
```

``` javascript
// content-script.js 接收
chrome.runtime.onMessage.addListener(function(request, sender, sendResponse){
    // console.log(sender.tab ?"from a content script:" + sender.tab.url :"from the extension");
    if(request.cmd == 'test') alert(request.value);
    sendResponse('我收到了你的消息！');
});
```

### content和background

``` javascript
// content-script.js 发送
chrome.runtime.sendMessage({greeting: '你好，我是content-script呀，我主动发消息给后台！'}, function(response) {
    console.log('收到来自后台的回复：' + response);
});
```

``` javascript
// background.js 或者 popup.js监听来自content-script的消息
chrome.runtime.onMessage.addListener(function(request, sender, sendResponse)
{
    console.log('收到来自content-script的消息：');
    console.log(request, sender, sendResponse);
    sendResponse('我是后台，我已收到你的消息：' + JSON.stringify(request));
});
```

> content_scripts向`popup`主动发消息的前提是popup必须打开！否则需要利用background作中转；
>
> 如果background和popup同时监听，那么它们都可以同时收到消息，但是只有一个可以sendResponse，一个先发送了，那么另外一个再发送就无效；

### content和popup

> 同content和background

### injected script和content-script

> `content-script`和页面内的脚本（`injected-script`自然也属于页面内的脚本）之间唯一共享的东西就是页面的DOM元素，有2种方法可以实现二者通讯：
>
> 1. 可以通过`window.postMessage`和`window.addEventListener`来实现二者消息通讯；
> 2. 通过自定义DOM事件来实现；

+ 方法一：（推荐）

``` javascript
//injected-script
window.postMessage({"test": '你好！'}, '*');

//content script
window.addEventListener("message", function(e){
    console.log(e.data);
}, false);
```

+ 方法二：

``` javascript
//injected-script
var customEvent = document.createEvent('Event');
customEvent.initEvent('myCustomEvent', true, true);
function fireCustomEvent(data) {
    hiddenDiv = document.getElementById('myCustomEventDiv');
    hiddenDiv.innerText = data
    hiddenDiv.dispatchEvent(customEvent);
}
fireCustomEvent('你好，我是普通JS！');

//content script
var hiddenDiv = document.getElementById('myCustomEventDiv');
if(!hiddenDiv) {
    hiddenDiv = document.createElement('div');
    hiddenDiv.style.display = 'none';
    document.body.appendChild(hiddenDiv);
}
hiddenDiv.addEventListener('myCustomEvent', function() {
    var eventData = document.getElementById('myCustomEventDiv').innerText;
    console.log('收到自定义事件消息：' + eventData);
});
```

## 长连接

短连接：`chrome.tabs.sendMessage`和`chrome.runtime.sendMessage`

长连接：`chrome.tabs.connect`和`chrome.runtime.connect`

``` javascript
// popup.js
getCurrentTabId((tabId) => {
    var port = chrome.tabs.connect(tabId, {name: 'test-connect'});
    port.postMessage({question: '你是谁啊？'});
    port.onMessage.addListener(function(msg) {
        alert('收到消息：'+msg.answer);
        if(msg.answer && msg.answer.startsWith('我是'))
        {
            port.postMessage({question: '哦，原来是你啊！'});
        }
    });
});

// content-script.js
// 监听长连接
chrome.runtime.onConnect.addListener(function(port) {
    console.log(port);
    if(port.name == 'test-connect') {
        port.onMessage.addListener(function(msg) {
            console.log('收到长连接消息：', msg);
            if(msg.question == '你是谁啊？') port.postMessage({answer: '我是你爸！'});
        });
    }
});
```

## 动态注入

### script

虽然在`background`和`popup`中无法直接访问页面DOM，但是可以通过`chrome.tabs.executeScript`来执行脚本，从而实现访问web页面的DOM（注意，这种方式也不能直接访问页面JS）

``` json
{
    "name": "动态JS注入演示",
    ...
    "permissions": [
        "tabs", "http://*/*", "https://*/*"
    ],
    ...
}
```

``` javascript
// 动态执行JS代码
chrome.tabs.executeScript(tabId, {code: 'document.body.style.backgroundColor="red"'});
// 动态执行JS文件
chrome.tabs.executeScript(tabId, {file: 'some-script.js'});
```

### css

``` json
{
    "name": "动态CSS注入演示",
    ...
    "permissions": [
        "tabs", "http://*/*", "https://*/*"
    ],
    ...
}
```

``` javascript
// 动态执行CSS代码，TODO，这里有待验证
chrome.tabs.insertCSS(tabId, {code: 'xxx'});
// 动态执行CSS文件
chrome.tabs.insertCSS(tabId, {file: 'some-style.css'});
```

## 获取标签和窗口

``` javascript
// 获取当前窗口ID
chrome.windows.getCurrent(function(currentWindow){
    console.log('当前窗口ID：' + currentWindow.id);
});

//获取当前标签页ID
// 1
function getCurrentTabId(callback){
    chrome.tabs.query({active: true, currentWindow: true}, function(tabs){
        if(callback) callback(tabs.length ? tabs[0].id: null);
    });
}

//2
function getCurrentTabId2(){
    chrome.windows.getCurrent(function(currentWindow){
        chrome.tabs.query({active: true, windowId: currentWindow.id}, function(tabs){
            if(callback) callback(tabs.length ? tabs[0].id: null);
        });
    });
}
```



##右键菜单

要将扩展加入到右键菜单中，首先要在Manifest的permissions域中声明contextMenus权限。
```javascript
"permissions": [
    "contextMenus"
]
```
同时还要在icons域声明16像素尺寸的图标，这样在右键菜单中才会显示出扩展的图标。
```javascript
"icons": {
    "16": "icon16.png"
}
```
Chrome提供了三种方法操作右键菜单，分别是create、update和remove，对应于创建、更新和移除操作。

通常create方法由后台页面来调用，即通过后台页面创建自定义菜单。如果后台页面是Event Page，通常在onInstalled事件中调用create方法。

右键菜单提供了4种类型，分别是普通菜单、复选菜单、单选菜单和分割线，其中普通菜单还可以有下级菜单。连续相邻的单选菜单会被自动认为是对同一设置的选项，同时单选菜单会自动在两端生成分割线。下面的代码生成了一系列的菜单：

我们还可以定义自定义的右键菜单在何时显示，比如当用户选择文本时，或者在超级链接上单击右键时。下面的代码定义当用户在超级链接上点击右键时，在菜单中显示“My Menu”菜单：
```javascript
chrome.contextMenus.create({
    type: 'normal',
    title: 'My Menu',
    contexts: ['link']
});
```
contexts域的值是数组型的，也就是说我们可以定义多种情况下显示自定义菜单，完整的选项包括all、page、frame、selection、link、editable、image、video、audio和launcher，默认情况下为page，即在所有的页面唤出右键菜单时都显示自定义菜单。其中launcher只对Chrome应用有效，如果包含launcher选项，则当用户在chrome://apps/或者其他地方的应用图标点击右键，将显示相应的自定义菜单。需要注意的是，all选项不包括launcher。

update方法可以动态更改菜单属性，指定需要更改菜单的id和所需要更改的属性即可。remove方法可以删除指定的菜单，removeAll方法可以删除所有的菜单。

## 桌面提醒
要使用桌面提醒功能，需要在Manifest中声明notifications权限。
```javascript
"permissions": [
    "notifications"
]
```
创建桌面提醒非常容易，只需指定标题、内容和图片即可。桌面系统窗口创建之后是不会立刻显示出来的，为了让其显示，还要调用show方法：`notification.show();`

> 需要注意的是，对于要在桌面窗口中显示的图片，必须在Manifest的web_accessible_resources域中进行声明，否则会出现图片无法打开的情况:
```javascript
"web_accessible_resources": [
    "icon48.png"
]
```


>如果希望images文件夹下的所有png图片都可被显示，可以通过如下声明实现：
```javascript
"web_accessible_resources": [
    "images/*.png"
]
```
桌面提醒窗口提供了四种事件：ondisplay、onerror、onclose和onclick。

[桌面通知](http://www.cnblogs.com/champagne/p/4831874.html)

## Omnibox
要使用omnibox需要在Manifest的omnibox域指定keyword：`"omnibox": { "keyword" : "hamster" }`。同时最好指定一个16像素的图标，当用户键入关键字后，这个图标会显示在地址栏的前端。`"icons": {"16": "icon16.png"}`。

Omnibox有四种事件：onInputStarted、onInputChanged、onInputEntered和onInputCancelled，分别用于监听用户开始输入、输入变化、执行指令和取消输入行为。其中执行指令是指用户敲击回车键或用鼠标点击建议结果。

## 书签
要在扩展中操作书签，需要在Manifest中声明bookmarks权限：
```javascript
"permissions": [
    "bookmarks"
]
```
书签对象有8个属性，分别是id、parentId、index、url、title、dateAdded、dateGroupModified和children。这8个属性并不是每个书签对象都具有的，比如书签分类，即一个文件夹，它就不具有url属性。index属性是这个书签在其父节点中的位置，它的值是从0开始的。children属性值是一个包含若干书签对象的数组。dateAdded和dateGroupModified的值是自1970年1月1日至修改时间所经过的毫秒数。只有id和title是书签对象必有的属性，其他的属性都是可选的。id不需要人为干预，它是由Chrome管理的。根的id为'0'。

'0'为根节点id，根节点下是不允许创建书签和书签分组的，它的下面默认只有三个书签分组：书签栏、其他书签和移动设备书签，如果创建时不指定parentId，则所创建的书签会默认加入到其他书签中。

如果创建的书签不包含url属性，则Chrome自动将其视作为书签分类。

通过move方法可以调整书签的位置，这种调整可以是跨越父节点的。

通过update方法可以更改书签属性，包括标题和URL，更新时未指定的属性值将不会更改。

通过remove和removeTree可以删除书签，remove方法可以删除书签和空的书签分组，removeTree可以删除包含书签的书签分组。

通过getTree方法可以获得用户完整的书签树。

search方法可以返回匹配指定条件的书签对象，匹配的条件只能字符串。

书签的事件：onCreated、onRemoved、onChanged、onChildrenReordered、onImportBegan、onImportEnded。
```javascript
chrome.bookmarks.onCreated.addListener(function(bookmark){
    console.log(bookmark);
});
```

## Cookies
要管理Cookies，需要在Manifest中声明cookies权限，同时也要声明所需管理Cookies所在的域：
```javascript
"permissions": [
    "cookies",
    "*://*.google.com"
]
```
如果想要管理所有的Cookies可以声明如下权限：
```javascript
"permissions": [
    "cookies",
    "<all_urls>"
]
```

Chrome定义的Cookie对象包含如下属性：name（名称）、value（值）、domain（域）、hostOnly（是否只允许完全匹配domain的请求访问）、path（路径）、secure（是否只允许安全连接调用）、httpOnly（是否禁止客户端调用）、session（是否是session cookie）、expirationDate（过期时间）和storeId（包含此cookie的cookie store的id）。

Chrome提供了get和getAll两个方法读取Cookies，get方法可以读取指定name、url和storeId的Cookie，其中storeId可以不指定，但是name和url必须指定。如果在同一URL中包含多个name相同的Cookies，则会返回path最长的那个，如果有多个Cookies的path长度相同，则返回创建最早的那个。

> 如果cookie的secure属性值为true，那么通过get获取时url应该是https协议。

set方法可以设置Cookie：
```javascript
chrome.cookies.set({
    'url':'http://github.com/test_cookie',
    'name':'TEST',
    'value':'foo',
    'secure':false,
    'httpOnly':false
}, function(cookie){
    console.log(cookie);
});
```

## 历史
要使用history接口，需要在Manifest中声明history权限：
```javascript
"permissions": [
    "history"
]
```
管理历史的方法包括search、getVisits、addUrl、deleteUrl、deleteRange和deleteAll。其中search和getVisits用于读取历史，addUrl用于添加历史，deleteUrl、deleteRange和deleteAll用于删除历史。

## 管理扩展与应用
除了通过chrome://extensions/管理Chrome扩展和应用外，也可以通过Chrome的management接口管理。management接口可以获取用户已安装的扩展和应用信息，同时还可以卸载和禁用它们。通过management接口可以编写出智能管理扩展和应用的程序。

要使用management接口，需要在Manifest中声明management权限：
```javascript
"permissions": [
    "management"
]
```
读取用户已安装扩展和应用的信息。Management提供了两个方法获取用户已安装扩展应用的信息，分别是getAll和get。
exInfo是扩展信息对象，其结构如下：
```javascript
{
    id: 扩展id,
    name: 扩展名称,
    shortName: 扩展短名称,
    description: 扩展描述,
    version: 扩展版本,
    mayDisable: 是否可被用户卸载或禁用,
    enabled: 是否已启用,
    disabledReason: 扩展被禁用原因,
    type: 类型,
    appLaunchUrl: 启动url,
    homepageUrl: 主页url,
    updateUrl: 更新url,
    offlineEnabled: 离线是否可用,
    optionsUrl: 选项页面url,
    icons: [{
        size: 图片尺寸,
        url: 图片URL
    }],
    permissions: 扩展权限,
    hostPermissions: 扩展有权限访问的host,
    installType: 扩展被安装的方式
}
```

## 标签
标签对象的结构：
```javascript
{
    id: 标签id,
    index: 标签在窗口中的位置，以0开始,
    windowId: 标签所在窗口的id,
    openerTabId: 打开此标签的标签id,
    highlighted: 是否被高亮显示,
    active: 是否是活动的,
    pinned: 是否被固定,
    url: 标签的URL,
    title: 标签的标题,
    favIconUrl: 标签favicon的URL,
    status :标签状态，loading或complete,
    incognito: 是否在隐身窗口中,
    width: 宽度,
    height: 高度,
    sessionId: 用于sessions API的唯一id
}
```
Chrome通过tabs方法提供了管理标签的方法与监听标签行为的事件，大多数方法与事件是无需声明特殊权限的，但有关标签的url、title和favIconUrl的操作（包括读取），都需要声明tabs权限。
```javascript
"permissions": [
    "tabs"
]
```
获取标签信息。Chrome提供了三种获取标签信息的方法，分别是get、getCurrent和query。get方法可以获取到指定id的标签，getCurrent则获取运行的脚本本身所在的标签，query可以获取所有符合指定条件的标签。
```javascript
chrome.tabs.get(tabId, function(tab){
    console.log(tab);
});

chrome.tabs.getCurrent(function(tab){
    console.log(tab);
});
```
query方法可以指定的匹配条件如下：
```javascript
{
    active: 是否是活动的,
    pinned: 是否被固定,
    highlighted: 是否正被高亮显示,
    currentWindow: 是否在当前窗口,
    lastFocusedWindow: 是否是上一次选中的窗口,
    status: 状态，loading或complete,
    title: 标题,
    url: 所打开的url,
    windowId: 所在窗口的id,
    windowType: 窗口类型，normal、popup、panel或app,
    index: 窗口中的位置
}
```
创建标签。创建标签与在浏览器中打开新的标签行为类似，但可以指定更加丰富的信息，如URL、窗口中的位置和活动状态等。
```javascript
chrome.tabs.create({
    windowId: wId,
    index: 0,
    url: 'http://www.google.com',
    active: true,
    pinned: false,
    openerTabId: tId
}, function(tab){
    console.log(tab);
});
```
获取指定窗口活动标签可见部分的截图。
```javascript
chrome.tabs.captureVisibleTab(windowId, {
    format: 'jpeg',
    quality: 50
}, function(dataUrl){
    window.open(dataUrl, 'tabCapture');
});
```
其中format还支持png，如果指定为png，则quality属性会被忽略。如果指定jpeg格式，quality的取值范围为0-100，数值越高，图片质量越好，体积也越大。扩展只有声明activeTab或<all_url>权限能获取到活动标签的截图：
```javascript
"permissions": [
    "activeTab"
]
```
content_scripts可以向匹配条件的页面注入JS和CSS，但是却无法向用户指定的标签注入。通过executeScript和insertCSS可以做到向指定的标签注入脚本。
```javascript
chrome.tabs.executeScript(tabId, {
    file: 'js/ex.js',
    allFrames: true,
    runAt: 'document_start'
}, function(resultArray){
    console.log(resultArray);
});
```
也可以直接注入代码：
```javascript
chrome.tabs.executeScript(tabId, {
    code: 'document.body.style.backgroundColor="red"',
    allFrames: true,
    runAt: 'document_start'
}, function(resultArray){
    console.log(resultArray);
});
```
向指定的标签注入CSS：
```javascript
chrome.tabs.insertCSS(tabId, {
    file: 'css/insert.css',
    allFrames: false,
    runAt: 'document_start'
}, function(){
    console.log('The css has been inserted.');
});
```
executeScript和insertCSS方法中runAt的值可以是'document_start'、'document_end'或'document_idle'。

与指定标签中的内容脚本（content script）通信。前面章节介绍过扩展页面间的通信，我们也可以与指定的标签通信，方法如下：
```javascript
chrome.tabs.sendMessage(tabId, message, function(response){
    console.log(response);
});
```

> 后台页面主动与content_scripts通信需要使用chrome.tabs.sendMessage方法

监控标签行为的事件包含onCreated、onUpdated、onMoved、onActivated、onHighlighted、onDetached、onAttached、onRemoved和onReplaced。

## Override Pages
目前支持替换的页面包含Chrome的书签页面、历史记录和新标签页面。
只需在Manifest中进行声明即可（一个扩展只能替换一个页面）：
```javascript
"chrome_url_overrides" : {
    "bookmarks": "bookmarks.html"
}

"chrome_url_overrides" : {
    "history": "history.html"
}

"chrome_url_overrides" : {
    "newtab": "newtab.html"
}
```

## 下载
Chrome提供了downloads API，扩展可以通过此API管理浏览器的下载功能，包括暂停、搜索和取消等。扩展使用downloads接口需要在Manifest文件中声明downloads权限：
```javascript
"permissions": [
    "downloads"
]
```
`chrome.downloads.download(options, callback);`其中options的完整结构如下：
```javascript
{
    url: 下载文件的url,
    filename: 保存的文件名,
    conflictAction: 重名文件的处理方式,
    saveAs: 是否弹出另存为窗口,
    method: 请求方式（POST或GET），
    headers: 自定义header数组,
    body: POST的数据
}
```
## 网络请求
![网络请求的整个生命周期所触发事件的时间顺序][2]
要对网络请求进行操作，需要在Manifest中声明webRequest权限以及相关被操作的URL。如需要阻止网络请求，需要声明webRequestBlocking权限。
```javascript
"permissions": [
    "webRequest",
    "webRequestBlocking",
    "*://*.google.com/"
]
```
```javascript
chrome.webRequest.onBeforeRequest.addListener(
    function(details){
        return {cancel: true};
    },
    {
        urls: [
            "*://bad.example.com/*"
        ]
    },
    [
        "blocking"
    ]
);
```
> header中的如下属性是不支持更改的：Authorization、Cache-Control、Connection、Content-Length、Host、If-Modified-Since、If-None-Match、If-Range、Partial-Data、Pragma、Proxy-Authorization、Proxy-Connection和Transfer-Encoding。

所有事件中，回调函数所接收到的信息对象均包括如下属性：requestId、url、method、frameId、parentFrameId、tabId、type和timeStamp。其中type可能的值包括"main_frame"、"sub_frame"、"stylesheet"、"script"、"image"、"object"、"xmlhttprequest"和"other"。

## 代理
Chrome浏览器提供了代理设置管理接口，这样可以让扩展来做到更加智能的代理设置。要让扩展使用代理接口，需要声明proxy权限：
```javascript
"permissions": [
    "proxy"
]
```
通过chrome.proxy.settings.set方法可以设置代理服务器，该方法需要两个参数，一个是代理设置对象，另一个是回调函数。

代理设置对象包括mode属性、rules属性和pacScript属性。其中mode属性为代理模式，可选的值有'direct'（直接连接，即不通过代理）、'auto_detect'（通过WPAD协议自动获取pac脚本）、'pac_script'（使用指定的pac脚本）、'fixed_servers'（固定的代理服务器）和'system'（使用系统的设置）。rules属性和pacScript属性都是可选的，rules指定了不同的协议通过不同的代理。

## API 

`chrome.alarms.*`

通过此制定周期性任务，需在manifest.json文件中声明权限

``` json
"permissions": [

	"alarms"

],
```

`chrome.alarms.create(string name, object alarmInfo)`

| 属性              | 类型     | 注释             |
| --------------- | ------ | -------------- |
| when            | double | 触发alarm时间，ms   |
| delayInMinutes  | double | 延迟时间，minute    |
| periodInMinutes | double | 周期性时间间隔，minute |

- 获取指定名字的alarm
  `chrome.alarms.get(string name, function(Alarm alarm) {...})`

- 获取所有alarm

  `chrome.alarms.getAll(function(array of Alarm alarms) {...})`

- 通过名字删除alarm

  `chrome.alarms.clear(string name, function(boolean wasCleared) {...})`

- 清除所有alarm

  `chrome.alarms.clearAll(function(boolean wasCleared) {...})`

- 监听alarm发生的事件，用于event page

  `chrome.alarms.onAlarm.addListener(function(Alarm alarm) {...})`

[1]: http://css-tricks.com/custom-scrollbars-in-webkit/
[2]: http://storage1.imgchr.com/CflJs.jpg