---
title: angularjs集成echart
date: 2017-03-24 12:01:07
categories: 
- 编程
- angularjs
tags:
- angularjs
- inspinia
- echarts
---
# echarts
echarts是baidu开源的知名数据可视化工具([官网地址](http://echarts.baidu.com/index.html),[API文档](http://echarts.baidu.com/api.html#echarts),[github地址](https://github.com/ecomfe/echarts)),最近项目中将angularjs+echarts进行了一次集成,下面分享一些使用心得和过程中遇到的一些问题。
<!-- more -->

# 基础依赖
angularjs 1.5.0
echarts 3.4.0
echarts官方并未提供echars与angularjs集成的库，推荐了三款[第三方的集成插件](https://github.com/ecomfe/echarts)

* angular-echarts 
* echarts-ng 
支持angularjs - 1.6.0+, echarts - 3.0.0+
* ng-echarts 
支持ECharts3.x (目前我使用的是这款插件)

``` js
.state('echarts.demo', {
            url: "/demo",
            templateUrl: "views/tfsupport/order/demo.html",
            data: { pageTitle: 'demo' },
            resolve: {
                //使用时需要装载echarts以及集成插件
                loadPlugin: function($ocLazyLoad) {
                    return $ocLazyLoad.load([{
                        files: ['js/plugins/echarts/echarts-all-3.js']
                    }, {
                        name: 'ng-echarts',
                        files: ['js/plugins/echarts/ng-echarts.js']
                    }]);
                }
            }
        })
```
（项目中使用了oclazyload做了依赖项的懒加载，如需立即加载直接进行页面资源引用即可）

# echarts+angular使用
## 模板
ng-echarts.js库中已经封装了echarts相关的directive,模板写法如下
``` html
<div ng-echarts ec-config="config" ec-option="option" style="height: 600px;"></div>
```
使用时有几个注意点:

* ngEcharts虽然match了element和attribute，但在实际使用过程中使用element匹配存在echarts兼容问题，无法达到理想效果，推荐还是使用attribute进行directive匹配。
* 核心参数ecConfig,ecOption
``` js
angular.module('ng-echarts', [])
    .directive('ngEcharts', [function() {
        return {
            link: function(scope, element, attrs, ctrl) {
                function refreshChart() {
                    function refreshChart() {
                    var theme = (scope.config && scope.config.theme) ? scope.config.theme : 'default';
                    if ($(element[0]).parent()) {
                       $(element[0]).css('width', $(element[0]).parent().width());
                    }
                    var chart = echarts.init(element[0], theme);
                    if (scope.config && scope.config.dataLoaded === false) {
                        chart.showLoading();
                    }

                    if (scope.config && scope.config.dataLoaded) {
                        chart.setOption(scope.option);
                        chart.resize();
                        chart.hideLoading();
                    }

                    if (scope.config && scope.config.event) {
                        if (angular.isArray(scope.config.event)) {
                            angular.forEach(scope.config.event, function(value, key) {
                                for (var e in value) {
                                    chart.on(e, value[e]);
                                }
                            });
                        }
                    }
                };

                };
                //自定义参数 - config
                // event 定义事件
                // theme 主题名称
                // dataLoaded 数据是否加载
                scope.$watch(
                    function() {
                        return scope.config;
                    },
                    function(value) {
                        if (value) { refreshChart(); }
                    },
                    true
                );
                //图表原生option
                scope.$watch(
                    function() {
                        return scope.option;
                    },
                    function(value) {
                        if (value) { refreshChart(); }
                    },
                    true
                );
            },
            scope: {
                option: '=ecOption',
                config: '=ecConfig'
            },
            restrict: 'AE'
        }
    }]);
```
ng-echarts源码中可以发现，ecConfig用来传递配置信息，注册echarts event事件回调以及loading效果都由这个参数进行传递，并且只有当config.dataLoaded属性为true，才会触发echarts数据绘制，例如

``` js
var echartConfig = { dataLoaded: false, event: [{ legendselectchanged: onLegendSelected }] };//图例切换触发onLegendSelected函数回调
function onLegendSelected(params) {
    ...
};
```
[echarts事件可参看官方配置手册](http://echarts.baidu.com/api.html#events)
ecOption用来传递图表的配置信息以及数据,与echarts的[setOption](http://echarts.baidu.com/option.html#title)函数对应,包括设置图表类型、标题、图例等

## 数据
准备好模板接下来就需要通过controller向模板输出数据，用于图表绘制。
图表选项定义

``` js
var option = {
    tooltip: {
        trigger: 'axis',
        formatter: function(dataPoints) {
            //定制数据提示
        },
        axisPointer: {
            animation: false
        },
        backgroundColor: '#FFFF',
        borderColor: '#333',
        borderWidth: 1,
        textStyle: { color: '#333', fontWeight: 'bolder' }
    },
    xAxis: {
        type: 'time',
        splitLine: {
            show: true
        },
        axisLabel: {
            rotate: 45,
            textStyle: { color: '#333', fontWeight: 'bolder' },
            formatter: function(value, index) {
                return moment(value, 'x').format('MM-DD HH:mm');
            }
        },
        splitNumber: 48,
        boundaryGap: true,
        min: 'dataMin'
    },
    yAxis: {
        type: 'value',
        splitLine: {
            show: true
        },
        axisLabel: {
            textStyle: { color: '#333', fontWeight: 'bolder' },
            formatter: function(value, index) {
                return value + '%';
            }
        },
    },
    series: [{
        name: '失败率',
        type: 'line',
        smooth: true,
        symbol: 'none',
        sampling: 'average',
        data: []//图表数据[{value: [x轴数据, y轴数据]}]
};
```

在项目实际使用过程中我们会需要数据重新reload触发echarts数据重新绘制，按照ng-echarts源码中原理，其实是注册了对scope.option数据的监听，但偶尔会遇到数据变更refresh未生效状况，可以通过手动触发apply，进行图表重新绘制

``` js
$timeout(function() {
    $scope.$apply(function() {
        //$scope.option数据变更
    });
}, 100);
```

这是我们项目中使用echarts做出来的一些图表效果
{%asset_img img1.png %}
{%asset_img img2.png %}
