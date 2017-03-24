---
title: angularjs集成nvd3打造数据可视化
date: 2017-03-24 11:56:44
---
title: angularjs集成nvd3打造数据可视化
date: 2017-03-23 16:28:13
categories: 
- 编程
- angularjs
tags:
- angularjs
- inspinia
- nvd3
---
# NVD3 Re-usable charts for d3.js
d3.js是一个大名鼎鼎的数据可视化工具,[官网地址](https://d3js.org/),[API文档](https://github.com/d3/d3/wiki/API--%E4%B8%AD%E6%96%87%E6%89%8B%E5%86%8C).
nvd3是在d3.js基础上封装的一个开源可复用的图表库,能够大大的提升图表制作的开发效率,在github上也是5000+star的项目,
[项目官网地址](http://nvd3.org/index.html),[项目地址](https://github.com/novus/nvd3),[这是一些demo](http://nvd3.org/examples/index.html).
<!-- more -->

# 基础依赖
nvd3本身仅只依赖d3库,因为我们的项目使用的是angularjs,所以这里也需要用到nvd3的angular插件angular-nvd3.js
``` js
$stateProvider
.state('charts', {
    abstract: true, //类似继承的概念，不会直接转换到这个转态，而是从这个转态进行过度。
    url: "/charts",
    templateUrl: "views/common/content.html",
})
.state('charts.order_monitor_chart', { //从charts继承
            url: "/order_monitor_chart",
            templateUrl: "views/tfsupport/order_monitor_chart.html",
            data: { pageTitle: '失败订单监控' },
            resolve: {
                loadPlugin: function($ocLazyLoad) {
                    return $ocLazyLoad.load([{
                        serie: true,
                        files: ['js/plugins/d3/d3.js']
                    }, {
                        serie: true,
                        files: ['js/plugins/nvd3/build/nv.d3.js', 'js/plugins/nvd3/build/nv.d3.css']
                    }, {
                        serie: true,
                        name: 'nvd3',
                        files: ['js/plugins/angular-nvd3/dist/angular-nvd3.js']
                    }]);
                }
            }
        })
```
（项目中使用了oclazyload做了依赖项的懒加载，如需立即加载直接进行页面资源引用即可）

# nvd3+angular使用
## 模板
angular-nvd3.js库中已经封装了nvd3相关的directive,在模板中引用会非常的便利
``` html
<nvd3 options="options" data="datas" class="with-3d-shadow with-transitions row" config="{refreshDataOnly: true, deepWatchData: true}"></nvd3>
```
没错就这么简单一句,加上数据就能绘制出酷炫的图表。
options对应了图表的配置,data绑定了图表数据,config对应了图表的一些系统属性
``` js
$scope.config = {
    visible: true, // default: true
    extended: false, // default: false
    disabled: false, // default: false
    refreshDataOnly: true, // default: true 这个定义数据更新的策略是重新绘图还是只绘制数据
    deepWatchOptions: true, // default: true
    deepWatchData: true, // default: true
    deepWatchDataDepth: 2, // default: 2 //定义了数据和视图的绑定层次 
    //0 - By Reference (the least powerful, but the most efficient) 
    //1 - By Collection Items
    //2 - By Value (the most powerful, but also the most expensive; default value)
    debounce: 10 // default: 10
};
```
## 数据
准备好模板接下来就需要通过controller向模板输出数据，用于图表绘制。
图表选项定义
``` js
$scope.options = {
    chart: {
        type: 'lineChart',
        height: 500,
        margin: {
            top: 20,
            right: 20,
            bottom: 120,
            left: 80
        },
        duration: 500,
        xAxis: {//x轴数据相关配置
            axisLabel: '时间',
            rotateLabels: -45,//x轴标签旋转
            tickFormat: function(d) {//x轴数据刻度的格式化
                return moment(d).format('MM-DD hh:mm');
            }
        },
        yAxis: {
            axisLabel: '数量',
        },
        useInteractiveGuideline: true,
        line: {//图表的点击等事件的自定义回调
            dispatch: {
                elementClick: function(e) {
                    $state.go('charts.partner_order_monitor_chart', { partnerCode: e.data.x });
                }
            }
        },
        tooltip: {
            contentGenerator: function(d) {//自定义数据tip(鼠标移至数据图上的tip效果)
                return '<h3>' + d.point.day + '</h3>' +
                    '<p>成功：' + d.point.y + '单，失败：<strong style="color:red">' + d.point.failedCount + '</strong>单</p>';
            }
        },
        legend: {//图例标签的相关配置
            dispatch: {
                legendClick: function(e) {//图例点击方法回调，可以在这里做一些自定义事件处理，还有一系列其他事件回调legendDblclick,legendMouseover,legendMouseout,,stateChange等
                    var disabledLegend = orderMonitorStore.get("historyDisabledLegend") ? orderMonitorStore.get("historyDisabledLegend") : [];
                    if (e.disabled) {
                        disabledLegend.remove(e.key);
                    } else {
                        disabledLegend.push(e.key);
                    }
                    orderMonitorStore.set("historyDisabledLegend", disabledLegend);
                    $scope.init(); //触发重新绘制
                }
            }
        }
    }
};
```
图表数据,基本的数据格式如下
``` js
$scope.data.push({
    key: "数据key",
    values: [{x:x轴数据,y:y轴数据}]
    }),
    //其他属性color,disabled等
});
```
这是我们项目中使用nvd3做出来的一些图表效果
{%asset_img img1.png %}
{%asset_img img2.png %}