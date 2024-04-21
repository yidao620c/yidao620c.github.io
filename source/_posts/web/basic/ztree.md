---
title: 树形控件zTree使用
date: 2016-06-09 17:55:22 +0800
toc: true
categories: [ web ]
tags: [ web ]
---

经常有这个需求，用用到树形菜单做展示或者选择，基于jquery的一个控件zTree非常简单好用，这里做一下记录。

官网介绍，zTree 是一个依靠 jQuery 实现的多功能"树插件"。优异的性能、灵活的配置、多种功能的组合是 zTree 最大优点。

官网地址：<http://www.treejs.cn/v3/main.php>

另外还有好多实际效果的演示，可以去看看。
<!-- more -->

## 下载

直接去github下载源码：<https://github.com/zTree/zTree_v3>

选取了一个最基本的例子，带metro风格的，比较好看。

<https://github.com/zTree/zTree_v3/blob/master/demo/cn/super/metro.html>

提取最核心的东东

```html
<!DOCTYPE html>
<HTML>
<HEAD>
    <TITLE> ZTREE DEMO - Simple Data</TITLE>
    <meta http-equiv="content-type" content="text/html; charset=UTF-8">
    <link rel="stylesheet" href="./css/demo.css" type="text/css">
    <link rel="stylesheet" href="./css/metroStyle.css" type="text/css">
    <script type="text/javascript" src="./js/jquery-1.11.1.js"></script>
    <script type="text/javascript" src="./js/jquery.ztree.core.min.js"></script>
    <script type="text/javascript" src="./js/jquery.ztree.excheck.min.js"></script>
    <SCRIPT type="text/javascript">
        var setting = {
            view: {
                selectedMulti: false,
				showLine: false
            },
            check: {
                enable: true
            },
            data: {
                simpleData: {
                    enable: true
                }
            },
            edit: {
                enable: false
            }
        };

        var zNodes =[
            { id:1, pId:0, name:"父节点1", open:true},
            { id:11, pId:1, name:"父节点11"},
            { id:111, pId:11, name:"叶子节点111"},
            { id:112, pId:11, name:"叶子节点112"},
            { id:113, pId:11, name:"叶子节点113"},
            { id:114, pId:11, name:"叶子节点114"},
            { id:12, pId:1, name:"父节点12"},
            { id:121, pId:12, name:"叶子节点121"},
            { id:122, pId:12, name:"叶子节点122"},
            { id:123, pId:12, name:"叶子节点123"},
            { id:124, pId:12, name:"叶子节点124"},
            { id:13, pId:1, name:"父节点13", isParent:true},
            { id:2, pId:0, name:"父节点2"},
            { id:21, pId:2, name:"父节点21", open:true},
            { id:211, pId:21, name:"叶子节点211"},
            { id:212, pId:21, name:"叶子节点212"},
            { id:213, pId:21, name:"叶子节点213"},
            { id:214, pId:21, name:"叶子节点214"},
            { id:22, pId:2, name:"父节点22"},
            { id:221, pId:22, name:"叶子节点221"},
            { id:222, pId:22, name:"叶子节点222"},
            { id:223, pId:22, name:"叶子节点223"},
            { id:224, pId:22, name:"叶子节点224"},
            { id:23, pId:2, name:"父节点23"},
            { id:231, pId:23, name:"叶子节点231"},
            { id:232, pId:23, name:"叶子节点232"},
            { id:233, pId:23, name:"叶子节点233"},
            { id:234, pId:23, name:"叶子节点234"},
        ];

        $(document).ready(function(){
            $.fn.zTree.init($("#treeDemo"), setting, zNodes);
        });
    </SCRIPT>
</HEAD>

<BODY>
<div class="content_wrap">
    <div class="">
        <ul id="treeDemo" class="ztree"></ul>
    </div>
</div>
</BODY>
</HTML>
```

记住引入`metroStyle.css`，还有metro的图片文件夹也引入，另外自己还自定义了一些样式：

```css
.ztree li.title {
    list-style: none;
}

.ztree ul.ztree {
    margin-top: 10px;
    border: 1px solid #617775;
    background: #f0f6e4;
    width: 220px;
    height: 360px;
    overflow-y: scroll;
    overflow-x: auto;
}

.ztree ul.log {
    border: 1px solid #617775;
    background: #f0f6e4;
    width: 300px;
    height: 170px;
    overflow: hidden;
}

.ztree ul.log.small {
    height: 45px;
}

.ztree ul.log li {
    color: #666666;
    list-style: none;
    padding-left: 10px;
}

.ztree ul.log li.dark {
    background-color: #E3E3E3;
}

ul.ztree li a {
    text-decoration: none;
    cursor: default;
}

ul.ztree li a:hover {
    text-decoration: none;
    cursor: default;
}

ul.ztree li a:active {
    text-decoration: none;
    cursor: default;
}

ul.ztree li a:visited {
    text-decoration: none;
    color: #333;
    background-color: #fff
}

ul.ztree li a.curSelectedNode {
    color: #333;
    opacity: 1;
    background-color: #fff;
}

ul.ztree li span {
    cursor: default;
}

ul.ztree li span.button.chk {
    cursor: default;
}

ul.ztree li span.button {
    cursor: default
}
```

## API

zTree的API介绍也很详细，<http://www.treejs.cn/v3/api.php>

这里我弄两个最典型的来讲，一个是获取所有的选择节点，另一个是将所有选择的叶子节点id拿出来。

```js
// 通过id号来获取这个树
var treeObj2 = $.fn.zTree.getZTreeObj("treeDemo");
// 获取所有已经选择了的节点，获得一个node列表
var nodes2 = treeObj2.getCheckedNodes(true);
// 如果是叶子节点就把id拿出来
var idlist = [];
$.each(nodes2, function (i, item) {
    if (!item.isParent) {
        //alert(item.id + ","  + item.name);
        idlist.push(item.id);
    }
});
console.log(idlist);
```

使用很简单，如果有其他需求直接去参考官网API文档即可。

最后来张效果图:

![](https://xnstatic-1253397658.file.myqcloud.com/ztree01.png)
