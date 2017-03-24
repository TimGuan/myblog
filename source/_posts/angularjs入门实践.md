---
title: angularjs入门实践
date: 2017-03-24 12:37:44
categories: 
- 编程
- angularjs
tags:
- angularjs
---

angularjs的基础知识在网上有很丰富的介绍资料，这里我将以一个后端开发的视角，从这个角度上来介绍一下angularjs的一些概念。

# Template   HTML with additional markup
view模板，与velocity的vm文件类似，区别在于vm作用于服务端渲染，数据与模板的作用关系在渲染完成后就结束了，而angularjs等前端框架实现了数据模板的双向绑定。
一个简单的模板demo，模板中通过ng-${??}等内置的angularjs指令，对数据进行操作。

``` html
...
<div class="col-lg-6">
    <ul class="stat-list" ng-repeat="failed in chart.failedBounceTop">
        <li>
            <h2 class="no-margins">{{failed.failedBounceOrderCnt}}/{{failed.totalBounceOrderCnt}}</h2>
            <small>{{failed.channel}}</small>
            <div class="stat-percent">{{100*failed.failedBounceRate| number:2}}% <i ng-if="failed.failedRate>0.1" class="fa fa-bolt text-navy"></i></div>
            <div class="progress progress-mini">
                <div style="width:{{failed.failedBounceRate*100}}% " class="progress-bar progress-bar-danger"></div>
            </div>
        </li>
    </ul>
</div>
```
<!-- more -->
# Data Binding   sync data between the model and the view
angularjs的定位是webapp，必然需要做到数据和视图的双向绑定，即界面的操作能实时反映到数据，数据的变更能实时展现到界面。
{% asset_img img1.png %}
关于双向绑定:http://www.html-js.com/article/2145
# Directives extend HTML with custom attributes and elements
写过jsp或者structs应该用过taglib，通过自定义的一些标签指令来复用处理逻辑（自定义的指令不要以ng-开头避免与angularjs自带的指令冲突）。

``` js
appModule.directive('hello', function() {
    return {
        restrict: 'E',//'A' - only matches attribute name,'E' - only matches element name,'C' - only matches class name,'M' - only matches comment
        template: '<div>{{name}} hello!</div>',
        replace: true
    };
});
```
自定义指令使用

``` html
<html ng-app='app'>
    <hello></hello>
...
</html>
```
# Model  the data shown to the user in the view and with which the user interacts
数据模型，类似springmvc中，modelAndView的model,封装了特定数据模型。
# Scope  context where the model is stored so that controllers, directives and expressions can access 
所有的数据的访问入口，view中的数据访问根节点。
# Filter formats the value of an expression for display to the user
封装了简单数据的处理逻辑,类似velocity中的tool。

``` html
<td>{{ data.date | date : 'yyyy-MM-dd HH:mm:ss' }}</td>
```
#Controller Service
controller类似springmvc中的controller，封装了部分业务逻辑，返回view中所接受的model数据。service与springmvc中的service/component类似，封装了通用业务逻辑，并注册至factory,在不同的controller中通过ioc注入，以单例方式使用，这一点与spring也一致。
controller scope view之间的关联（mvc）
{% asset_img img2.png %}

controller service template之间的关联（ioc思想）
{% asset_img img3.png %}


``` js
'use strict';
angular
    .module('inspinia')
    .service('Gateway', function($http, $q, CONST_REQUEST_URL, CONST_WEB_LOGIN_LINK) {...}
angular
    .module('inspinia')
    .controller('searchLogCtrl',
        function searchLogCtrl(DTOptionsBuilder, Gateway, search_one) {
            ...
            var searchOne = new Gateway(search_one);
            var that = this;
            searchOne.sendRequest()
                .then(function(data) {
                    that.datas = data;
                }, function(data) {
                    console.log(data);
                });
            ...
        })
```