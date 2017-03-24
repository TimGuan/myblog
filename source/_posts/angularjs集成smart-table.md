---
title: angularjs集成smart-table
date: 2017-03-24 11:53:02
categories: 
- 编程
- angularjs
tags:
- angularjs
- inspinia
- smart-table
---
# Smart-Table
smart-table是一款高效的表格利器,在github上也是千star的项目,在分页、排序、检索等表格常用操作的支持上封装做的相当优秀，使用也相当简单。[源码地址](https://github.com/lorenzofox3/Smart-Table),[入门示例](http://lorenzofox3.github.io/smart-table-website/)。
<!--more-->

# 基础依赖
angularjs 1.0.0^,bootstrap 3.2.0^,smart-table
注册依赖
``` js
angular.module('inspinia',['smart-table'...])...
```

# smart-table使用
## 模板
### 基础使用
* 1.st-pipe="callServer"数据回调函数，表格初始化以及分页获取数据的函数封装；
* 2.st-table="rowCollection"数据节点，我理解这个也定义smart-table的作用范围，因为callServer等交互行为触发的变化应当仅在该容器内，类似于table的闭包；
* 3.数据行
``` html
<tr ng-repeat="row in rowCollection">
```

### 分页
``` html
<div st-items-by-page="20" st-pagination=""></div>
```
st-pagination分页标记，st-item-by-page用于指定数据页大小， st-template="/views/common/pagination_custom.html"自定义分页模板设置
```  html
<!--分页模板-->
<style type="text/css">
.pagination{
    margin: 10px 0;
}
.pagination li a input {
  padding: 0;
  margin: -5px 0;
}
</style>
<nav ng-if="pages.length >= 2">
    <ul class="pagination pull-left">
        <li><a ng-click="selectPage(1)">First</a>
        </li>
        <li><a ng-click="selectPage(currentPage - 1)">pre</a>
        </li>
        <li>
            <a>
                <page-select></page-select>
            </a>
        </li>
        <li><a ng-click="selectPage(currentPage + 1)">next</a>
    </ul>
</nav>
```
### 终端适配
``` html
<th device-switch-sytle mobile-class="ng-hide">请求地址</th>
```
device-switch-sytle是我自己扩展的用于不同终端适配的directive,pc和mobile在打开相同页面在某些dom上进行差异化处理；
```  js
angular
    .module('inspinia')
    .directive('deviceSwitchSytle', ["deviceDetector", function(deviceDetector) {//浏览器终端识别http://srfrnk.github.io/ng-device-detector/
        //不同终端样式定制
        return {
            restrict: "A",
            link: function(scope, elm, attrs) {
                var styles, clazz;
                if (deviceDetector.isMobile()) {
                    clazz = attrs["mobileClass"];
                    styles = attrs["mobileStyle"];
                } else {
                    clazz = attrs["pcClass"];
                    styles = attrs["pcStyle"];
                }
                if (clazz) {
                    elm.addClass(clazz);
                }
                if (styles && styles.length > 0) {
                    styles.forEach(function(style) {
                        var kv = style.split(":");
                        if (kv && kv.length == 2) {
                            elm.css(kv[0], kv[1]);
                        }
                    });
                }
            }
        };
    }]);
```

### bootstrap table
对于数据内容不确定最好选择table-responsiver,进行内容自适配;table样式可用选项table table-bordered(带border) table-striped(斑马效果) table-hover(自带hover效果);

### filter search
[filter search相关](http://lorenzofox3.github.io/smart-table-website/#section-filter)

### 完整模板
``` html
<div class="table-responsiver row" st-pipe="callServer" st-table="rowCollection">
    <div class="col-md-12">
        <table class="table table-bordered table-striped table-hover">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>调用时间</th>
                    <th>调用耗时(ms)</th>
                    <th>状态</th>
                    <th device-switch-sytle mobile-class="ng-hide">请求地址</th>
                </tr>
            </thead>
            <tbody ng-if="!isLoading">
                <tr ng-repeat="row in rowCollection">
                    <td>
                        <button type="button" ng-click="showDetail(row)" class="btn btn-sm btn-success">
                            <span device-switch-sytle mobile-class="ng-hide">{{ row.id }}</span><i class="fa fa-folder-open-o"></i>
                        </button>
                    </td>
                    <td>{{ row.gmtCreate | date : 'yyyy-MM-dd HH:mm:ss' }}</td>
                    <td>{{ row.time }}</td>
                    <td><span class="badge" ng-class="{true:'badge-primary',false:'badge-danger'}[(row.exception||'')==='']">{{((row.exception||'')==='')?"success":"failed"}}</span></td>
                    <td device-switch-sytle mobile-class="ng-hide">
                        <a href="{{row.url}}" target="_blank" ng-if="row.url" alt="{{row.url}}">
                            <i class="fa fa-link"></i> {{row.url|cut:true:100:' ...'}}
                        </a>
                        <a ng-if="row.request" alt="{{row.url}}">
                            <i class="fa fa-unlink"></i> {{row.request|cut:true:100:' ...'}}
                        </a>
                    </td>
                </tr>
            </tbody>
            <tbody ng-if="isLoading">
                <tr>
                    <td colspan="5">
                        <div class="spiner-example">
                            <div class="sk-spinner sk-spinner-wave">
                                <div class="sk-rect1"></div>
                                <div class="sk-rect2"></div>
                                <div class="sk-rect3"></div>
                                <div class="sk-rect4"></div>
                                <div class="sk-rect5"></div>
                            </div>
                        </div>
                    </td>
                </tr>
            </tbody>
        </table>
    </div>
    <div class="col-md-12">
        <div st-items-by-page="20" st-pagination></div>
    </div>
</div>
```

## 数据获取
smart-table没有过多干涉table的数据渲染，而是通过设定数据回调函数，通过分页插件触发数据回调函数进行数据变更，所以使用上还是很容易的。
### 数据初始化及分页
```
/**
 * 服务器数据获取回调函数 st-pipe
 * @param  {[type]} tableState table状态信息封装，分页等信息
 * @return {[type]}            [description]
 */
$scope.callServer = function(tableState) {
    $scope.isLoading = true;
    var start = 0; // This is NOT the page number, but the index of item in the list that you want to use to display the table.
    var number = 20; // Number of entries showed per page.st-items-by-page="20"
    if (tableState) {
        var pagination = tableState.pagination;
        start = pagination.start || 0;
        number = pagination.number || 20;
        pagination.numberOfPages = 200; //set the number of pages so the pagination can update
    }
    //my request
    var request = new BaseRequest(syslog_tfbackyard_digest, {
        'date': $scope.selectedLogDate,
        'partner': $scope.selectedPartner,
        'cinema': $scope.inputCinemaId,
        'bizcode': $scope.selectedBizCode,
        'pageindex': start / number,
        'pagesize': number
    });
    request.sendRequest()
        .then(function(data) {
            //数据回调
            $scope.rowCollection = data;
            $scope.isLoading = false;
        }, function(data) {
            console.log(data);
            $scope.isLoading = false;
        });
};
```

##实现效果
PC效果
{% asset_img img1.png %}
移动端适配后效果
{% asset_img img2.png %}
