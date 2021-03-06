# pjax

可前端独立执行的 pjax。同时请求中也会携带两个ajax请求头，方便后端做优化处理。
```js
xhr.setRequestHeader("X-PJAX", true); // 表明是 pjax 请求
xhr.setRequestHeader("X-PJAX-Container", container); // 表明是当前需要请求的内容容器选择器所组成的数组的JSON字符串
```

## 依赖插件
[[jQuery]](http://jquery.com/)


## 调用方法
```js
var pjax = new Pjax({
    container: 'head,body', // 一个选择器，表示更新异步加载内容的容器。如果需要同时更新多个容器中的内容，则每个容器选择器用“,”隔开。
    // container 也可以是一个数组，每个 container 数组值对应一个 link 数组值。这种设置是为了同时设置不同容器与不同链接间的刷新关系。
    link: '', // 一个选择器，表示链接。点击后使用 ajax 加载内容。如果需要选择多个链接，则每个链接选择器用“,”隔开。
    // link 也可以是一个数组，每个 link 数组值对应一个 container 数组值
    script: '', // 一个script的选择器，将从加载到的HTML中过滤出选择器指定的script标签
    // script 也可以是一个数组，每个 script 数组值对应一个 container 数组值

    active: '', // 一个选择器，表示可以成为焦点的元素。当页面url更新后，会将href属性值与当前url相同的元素添加上焦点类
    activeClass: 'active', // 导航焦点类名

    noCache: false, // 表示是否不缓存更新内容，即每次都重新请求。
    noHistory: false, // 表示是否不新建历史记录，即每次都覆盖当前历史记录。
    noChangeURL: false, // 表示是否不改变当前URL。
    adjust: false, // 表示页面更新完成后是否调整页面位置。
    pile: false, // 表示更新到的内容是否累积到容器中，即不覆盖容器中的内容。
    // 以上五个值都与 link 配置中指定的链接对应（取值也可以为0或1）
    // 如果 link 中指定了多个链接，则以数组的形式与其对应。如果 link 是一个数组，则以二维数组的形式与其对应
    // pile 有些特殊，与 link 对应的每一个值可能并不是简单的布尔值，而是与当前 link 值对应的 container 值中每个容器一一对应的数组，它用来指明不同链接触发的更新在每一个容器中的更新规则
    // 例如：
    // container: '.container1, .container2, .container3',
    // link: '.link1, .link2',
    // pile: [[0,1,0], [0,0,0]],

    noConvertPath: false, // 表示是否不转换页面中所有资源的相对路径。默认关闭，如果确定异步加载的页面与当前页面都在同一目录下，则可以开启。
    adjustOffset: 0, // 如果开启了更新后自动调整，则该值用于设置调整的偏移量。

    load: function(url, link){}, // 加载开始时的回调，将请求的url与导航链接的DOM元素作为参数传入
    done: function(url, data, link){}, // 加载结束时的回调，将请求的url与导航链接的DOM元素以及请求到的data作为参数传入
    fail: function(url, link){}, // 加载结束时的回调，将请求的url与导航链接的DOM元素作为参数传入

    update: function($containerr, $newContainer, href){}, // 更新内容前的回调，如果有多个容器，则每个容器在内容更新前都会调用一次
    // $container    表示容器的 jQuery 对象（容器不不会被更新，更新的只是容器的内容）
    // $newContainer 表示新内容容器的 jQuery 对象，在更新前可以修改该对象，从而改变被更新的内容
    // href 触发内容更新的链接地址
    complete: function(href){} // 所有内容更新完成后的回调，只会调用一次
});


// 通过 update 更新的链接地址不会调用缓存，而是重新请求
// options 包含以下参数
// - href 表示要刷新的 url，默认为当前页面的 url
// - container 表示要刷新容器的选择器，如果需要同时刷新多个容器中的内容，则每个容器选择器用“,”隔开
//   注意，这里的 container 选择器必须是创建 pjax 对象时指定的容器选择器中包含的，否则将无法更新
// - noHistory 表示这次刷新是否不新建历史记录
// - noChangeURL 表示是否不改变当前URL
// - adjust 表示页面更新完成后是否调整页面位置
// - pile 表示更新是否累积的规则所组成的数组，该数组中的每一个值和 container 参数中每一个值对应
pjax.update(options);
```

## AMD
```js
require(["Pjax"], function(Pjax) {
    new Pjax({
        container: '.section-title, .list__label-area, .list__show-list, .pagination',
        link: '.list__label-area a, .pagination a:not(.pagination-next), .pagination a.pagination-next',
        noHistory: [0, 1, 1],
        adjust: [0, 1, 0],
        pile: [[0,0,0,0], [0,0,0,0], [0,0,1,0]],
        load: function() {
            window.bjj.progress.start();
        },
        done: function(url) {
            window.bjj.progress.done();
        },
        fail: function(url) {
            alert("您所请求的内容不存在！\n" + url);
            window.bjj.progress.fail();
        },
        complete: function() {
            $(document).trigger('lazyloadUpdate');
        }
    });
});
```


## 主要原理

1) 创建Pjax对象后，都会将页面的所有外部链接地址转化为绝对路径。

2) 如果浏览器支持 history.replaceState，则使用 history.replaceState 替换掉当前的历史记录点，保证以后回退到初始 url 时不会出错。

3) 根据配置信息，检查所有焦点元素，将href属性值与当前url相同的元素添加上焦点类。并缓存当前页面的 href 以及对应的内容。

4) 点击链接后，根据链接的 href 检查缓存，判断是否已经被加载。如果 href 已经被加载，那么读取缓存。如果没有加载，则用 ajax 去加载。

5) 拿到内容信息后，先进行内容解析（注意，这里解析时会将所有的 script 标签过滤掉），然后将需要的内容进行缓存。

6) 如果浏览器支持 history.replaceState，则保存历史记录点，并将请求的 href 作为记录点的数据。

7) 最后为容器替换新内容（这里如果需要执行新页面中的 script 脚本，则必须在 script 选项中配置该 script 标签的选择器）。

8) 监听 popstate 事件，在事件触发时根据得到的 href 拿到缓存的内容信息，并为容器替换回历史内容。




## 经验总结

### 页面总是回到顶部的问题

使用 pushState 方法时，浏览器会记录页面当前的 scroll 值，并保存在上一条的历史记录中。<br>
但在 chrome 浏览器上发现，浏览器好像并不会去获取页面的 scrollTop/scrollLeft，而是有自己的一套记录机制。

这导致了两个问题：

1) 如果两次修改历史记录期间没有滚动过页面，那么上一条历史记录的 scrollTop/scrollLeft 就会为0，当后退到该记录点时，页面会回到文档顶部。

- 解决方案是，在页面添加历史记录点前，应该使用页面的 scrollTop/scrollLeft 属性加1减1，来强制浏览器记录当前的scroll位置。

2) 回退到前一条历史时，如果不滚动页面，当加载新链接时，页面还是会回到文档顶部。

- 解决方案是，在 popstate 事件处理函数中强制浏览器记录当前的scroll位置。但是这里必须异步执行，同步执行无效。

后来测试，IE和火狐都没有这个问题。




