---
title: WEB系统前后端分离设计(二)--前端部分
date: 2017-03-24 11:42:46
categories: 
- 编程
- 系统设计
tags:
- angularjs
- inspinia
- 系统设计
---
## 技术框架
angularjs1.5+[inspinia v2.6.2.1](http://webapplayers.com/inspinia_admin-v2.6.2.1/Angular/#/dashboards/dashboard_1)
<!-- more -->

## 前端工程目录结构
``` yaml
app---
|---index.html 页面访问根入口
|---font-awesome 字体
|---fonts 字体
|---css
    |---plugins 插件使用的样式库
    |---bootstrap.css animate.css ... style.css 自定义样式
|---views
    |---common 页面的公共部分包括nav、header、footer等
        |---content.html 页面内容布局，包含了对navigation/topnavbar/footer/right_sidebar等模块引用，同时保留了主体内容的嵌入ui-view
        |---navigation.html 左侧菜单栏
        |---topnavbar.html 顶部导航 right_sidebar.html右侧导航 footer.html底部面包屑
    |---tfsupport tfsupport业务扩展的页面
        |---common_detail_modal.html 封装的查看表格详情的控件 
        |---common_notify.html 全局提示用的message模板,参看[angular-notify](https://github.com/cgross/angular-notify)
        |---multi-day.tpl.html、multi-datepicker.tpl.html 多日期选择控件(指定多日期)[gm.datepickerMultiSelect](https://github.com/spongessuck/gm.datepickerMultiSelect)
        |---pageination_custom.html 表格页码控件[smart-table](https://github.com/lorenzofox3/Smart-Table)
        |---其他文件夹命名规范 ${业务模块}/${业务名称}.html，config.js中不同state对应的页面地址都来自于这个目录下的html文件，最终这些页面模板被内嵌到content.html中ui-view标签内
|---js  
    |---angular、bootstrap、jquery、ui-router为基础库引用，一般不需变更
    |---plugins 插件目录
        |---[a0-angular-storage](https://github.com/auth0/angular-storage)(storage插件，提供了localstorage/sessionstorage/cookiestorage操作封装)
        |---[angular-bootstrap-datetimepicker](https://github.com/dalelotts/angular-bootstrap-datetimepicker)(日期选择器)
        |---angular-moment(momentjs angularjs集成)、moment
        |---[angular-notify](https://github.com/cgross/angular-notify)(全局提示)
        |---[angular-nvd3](http://nvd3-community.github.io/nvd3/examples/documentation.html)(d3 angularjs封装,推荐使用echarts,这个不如echarts好用,具体参看《angularjs集成nvd3打造数据可视化》一文)、d3
        |---[angular-smart-table](https://github.com/lorenzofox3/Smart-Table)(表格插件,具体参看《angularjs集成smart-table》)
        |---[angular-chosen-localytics](https://github.com/leocaseiro/angular-chosen)(带提示的select,chosen angular集成)、chosen
        |---[echarts](echarts.baidu.com)(具体参看《angularjs集成echart》一文)、[gm.datepickerMultiSelect](https://github.com/spongessuck/gm.datepickerMultiSelect)(多重日期选择器)
        |---[ladda](https://github.com/remotty/angular-ladda)(按钮加载效果)
        |---[oclazyload](https://github.com/ocombe/ocLazyLoad)(组件异步加载!!!)
        |---[searchandselect_angularjs](https://github.com/angular-ui/ui-select)(输入提示器)
        |---underscore(js函数库)
    |---app.js 公共依赖加载，常量定义
    |---config.js 路由定义，[uirout](https://github.com/angular-ui/ui-router/wiki)
    |---controller.js 公用controller，MainCtrl(提供login、logout函数)、commonDetailModalCtrl(处理表格数据详情打开页内的事件)、eventbus(全局event msg传递)
    |---directives.js 页面内一些公共组件，iboxContent为扩展的水印组件
    |---inspinia.js 页面缩放等
```

## 页面结构
{% asset_img img1.png %}
页面访问根入口 index.html
``` html
<!DOCTYPE html>
<html ng-app="inspinia">
<head>
    ...
</head>
<!-- ControllerAs syntax -->
<!-- Main controller with serveral data used in Inspinia theme on diferent view -->

<body ng-controller="MainCtrl as main" landing-scrollspy id="page-top">
    <!-- Main view  -->
    <div ui-view></div><!--这里指向content.html-->
    <script src="js/jquery/jquery-2.1.1.min.js"></script>
    ...
</body>
</html>

```
content.html
``` html
<!-- Wrapper-->
<div id="wrapper">
    <!-- Navigation -->
    <div ng-include="'views/common/navigation.html'"></div>
    <!-- Page wraper -->
    <!-- ng-class with current state name give you the ability to extended customization your view -->
    <div id="page-wrapper" class="gray-bg {{$state.current.name}}">
        <!-- Page wrapper -->
        <div ng-include="'views/common/topnavbar.html'"></div>
        <!-- Main view  -->
        <div ui-view></div>
        <!-- Footer -->
        <div ng-include="'views/common/footer.html'"></div>
    </div>
    <!-- End page wrapper-->
    <!-- Right Sidebar -->
    <div ng-include="'views/common/right_sidebar.html'"></div>
</div>
<!-- End wrapper-->
```

## 声明路由状态
状态每个路由的实际状态都会从一个抽象状态中继承，抽象状态指定content.html做为页面模板，所以正常的加载路径为index.html=>content.html(对应abstract state)=>wiki.html(对应实际state)。
``` js
//config.js
$urlRouterProvider.otherwise("/order/order_partner_failorderrate/");
    $ocLazyLoadProvider.config({
        // Set to true if you want to see what and when is dynamically loaded
        debug: false
    });
    $stateProvider
        .state('tools', {
            abstract: true,//抽象状态，tools.{children}都从tools继承
            url: "/tools",
            templateUrl: "views/common/content.html"
        })
        .state('tools.wiki', {
            url: "/wiki",
            templateUrl: "views/tfsupport/tools/wiki.html",//页面中声明了对应的处理controller
            data: { pageTitle: 'Wiki' }
            resolve: {
                loadPlugin: function($ocLazyLoad) {
                    //使用ocLazyLoad懒加载对应页面执行需要的js组件
                    return $ocLazyLoad.load([{
                        serie: true,
                        files: ['js/plugins/d3/d3.js']
                    }]);
                }
            }
        })
```

### template编写
标准angularjs模板，模板中声明使用的controller，ps:controller js文件不能采用ozLazyLoad加载，所以如果有controller js文件新增需要在index.html中添加引用，触发预先加载。

``` html
...
<div class="row wrapper border-bottom white-bg page-heading">
    <div class="col-lg-12">
        <h2>文件导出列表</h2>
        <ol class="breadcrumb">
            <li>
                <a href="index.html">首页</a>
            </li>
            <li>
                <a>更多功能</a>
            </li>
            <li class="active">
                <strong>文件导出列表</strong>
            </li>
        </ol>
    </div>
</div>
<div class="wrapper wrapper-content animated fadeInRight" ng-controller="exportRecordListSearchCtrl">
    <div class="row">
        <div class="col-lg-12">
            <div class="ibox">
                <div class="ibox-title">
                    <h5>文件导出列表</h5>
                    <div ibox-tools></div>
                </div>
                <div class="ibox-content">
                    <div class="table-responsiver row" st-pipe="callServer" st-table="rowCollection">
                        <div class="col-md-12">
                            <table class="table table-bordered table-striped table-hover">
                                <thead>
                                    <tr>
                                        <th>文件名</th>
                                        ...
                                    </tr>
                                </thead>
                                <tbody ng-if="!isLoading">
                                    <tr ng-repeat="row in rowCollection ">
                                        <td>{{row.fileName}}</td>
                                        ...
                                    </tr>
                                </tbody>
                            </table>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
```

### controller编写

* 不同于后端分层，前端的model一般都是与页面元素(模板)对应，所以模块化基本是一组件粒度封装。根据实际的使用经验，暂未强烈的controller、service分层的需求，tfsupport中controller包含了对数据访问以及业务处理，并像模板突出相应的数据模型。
``` js
angular
    .module('inspinia')
    .controller('exportRecordListSearchCtrl', function($scope, BaseRequest,  get_export_record) {
        /**
         * 服务器数据获取
         * @param  {[type]} tableState [description]
         * @return {[type]}            [description]
         */
        $scope.callServer = function(tableState) {
            $scope.isLoading = true;
            var start = 0,
                number = 20;
            if (tableState) {
                var pagination = tableState.pagination;
                start = pagination.start || 0;
                number = pagination.number || 20;
                pagination.numberOfPages = 200;
            }
            var request = new BaseRequest(get_export_record, {
                'pageIndex': start / number,
                'pageSize': number
            });
            request.sendRequest()
                .then(function(data) {
                    _.each(data, function(value, index) {
                        value.status = value.expiration && moment().isAfter(moment(value.expiration)) ? "EXPIRED" : value.status;
                    });
                    $scope.rowCollection = data;
                    $scope.isLoading = false;
                }, function(data) {
                    console.log(data);
                    $scope.isLoading = false;
                });
        };
    })
```

* basereqeust封装对接口的调用
controller调用baseRequest的sendeRequest方法发起对接口数据的调用。每个要访问的后端api都对应了一个请求描述实体，controller中声明了要请求的接口以及对应参数，最终的网络请求由BaseRequest封装。这个调用模式有点类似我们使用的sql template模式,api.js中定义了api请求(sql template), 业务层指定调用参数以及对应的接口(sql map中的method), baserequest负责向后端发起调用并处理一些通用的错误(proxy中负责对sql的build和向DB服务器发起请求).
api定义
``` js
angular
    .module('inspinia')
    .value('get_export_record', {
        api: {
            PATH: '/tools/export/getExportRecord.json',//接口访问地址
            HTTP_METHOD: 'GET',//请求方式,一般用GET即可,已经在底层处理了GET 的cache问题,放心使用
            SECURITY_TYPE: 'UserLogin',//tfsupport暂无非登入态发起的请求
            REQUIRED: {},//必填参数,如缺少必填参数,请求会被拦截不向后端发起
            OPTIONAL: {//可选参数
                'id': 'int',
                'pageIndex': 'int',
                'pageSize': 'int'
            }
        }
    })
```

baserequest.js
``` js
'use strict';

/**
 * baserequest Module
 *
 * Description
 */
angular
    .module('inspinia')
    .service('BaseRequest', function($http, $cookies, $q, $log, notify, featureUtil, ENV, USER_PROFILE_COOKIE_NAME, CONST_NOTIFY_TPL, CONST_REQUEST_URL, CONST_WEB_LOGIN_LINK, CONST_ACL_APPLY_LINK, COMMON_RETURNCODE, COMMON_CONST) {
        var SecurityType = {
            UserLogin: {
                name: 'UserLogin'
            },
            None: {
                name: 'None'
            }
        };

        var Gateway = function(options, data) {
            this.options = options;
            this.init(data);
        };

        Gateway.prototype = {
            /**
             * @function buildRequestData
             * @description 通过与定义的API接口和用户通过this.data传入的参数构建向服务端发起请求需要用到的数据
             * @return {Object} 向服务端提交的数据结构
             */
            buildRequestData: function() {
                if (this.options && this.options.api) {
                    var requestData = {};
                    // 提供公共参数
                    _.extend(requestData, this.options.api.COMMON);

                    var key;
                    // 添加必填参数
                    for (key in this.options.api.REQUIRED) {
                        requestData[key] = this.data[key];
                    }

                    // 添加选填参数
                    for (key in this.options.api.OPTIONAL) {
                        requestData[key] = this.data[key];
                    }

                    return requestData;
                }
            },

            /**
             * @override
             * @function init
             * @description 初始化方法
             * @param  {Object} data 传入的涌来提交服务端的数据
             */
            init: function(data) {
                this.setData(data);
            },

            /**
             * @function setData
             * @description 修改请求参数
             */
            setData: function(data) {
                this.data = data;
            },

            /**
             * @function sendRequest $q.defer
             * @description 构建请求
             * @param {boolean} isForceUserLogin 是否需要强制用户登录
             * @return {Object} $q.defer
             */
            sendRequest: function(isForceUserLogin) {
                //step1 调用子业务类进行数据校验
                var checkResult = this.validate(isForceUserLogin);
                if (checkResult !== true) {
                    $log.warn(checkResult);
                    return checkResult;
                }
                //step2 构建ajax请求数据，放到baseCommon的全局变量中，可以支持多个请求一起发送
                var requestData = this.buildRequestData();
                //step3 发送请求,获得返回数据
                return this.request(requestData, isForceUserLogin);
            },

            /**
             * @function validate
             * @description 根据用户SecurityType进行检验，同时也会检验比天字段是否写入
             * @returns {Error|Boolean} 错误对象|验证成功
             */
            validate: function(isForceUserLogin) {
                // 检查必填字段，如果没有必填字段直接抛错
                if (this.options && this.options.api) {
                    if (!this.options.api.PATH) return new Error('缺少方法参数' + key);
                    for (var key in this.options.api.REQUIRED) {
                        if (!_.has(this.data, key)) {
                            return new Error('缺少必填字段' + key);
                        }
                    }
                }
                //根据action校验是否登录
                //强制所有请求都登入
                if (!this.checkUserLogin()) {
                    if (isForceUserLogin) {
                        this.goToLogin();
                    } else if (this.options.api.SECURITY_TYPE === SecurityType.UserLogin.name) {
                        $log.error('该请求需要在登录后发起');
                        this.goToLogin();
                    }
                }
                return true;
            },

            /**
             * @function checkUserLogin
             * @description 检查用户是否登陆
             * @return {Boolean} 用户是否登陆
             */
            checkUserLogin: function() {
                if (ENV != "daily") {
                    //不存在userProfile认为未登入
                    return $cookies.get(USER_PROFILE_COOKIE_NAME);
                } else {
                    return true;
                }
            },

            /**
             * @function goToLogin
             * @description 跳转到登陆
             */
            goToLogin: function() {
                window.location.href = CONST_WEB_LOGIN_LINK;
            },
            /**
             * 申请权限
             */
            showAclApply: function(response) {
                if (response.data.state && response.data.state.code == COMMON_RETURNCODE.PERMISSION_ERROR && response.data.state.features) {
                    var permission = featureUtil.getValueByKey(response.data.state.features, COMMON_CONST.ACL_PERMISSION_NAME_FEATURES_KEY);
                    var applyHref = CONST_ACL_APPLY_LINK + permission;
                    notify({
                        messageTemplate: '<div>您缺少' + permission +
                            '权限,部分功能将无法正常使用.<a href=' + applyHref + ' target="_blank">[点我申请]</a></div>',
                        classes: 'alert-danger',
                        templateUrl: CONST_NOTIFY_TPL
                    });
                }
            },
            /**
             * @function request
             * @param  {Map}     data             请求参数
             * @param  {Boolean} isForceUserLogin 是否需要强制登陆
             * @param  {boolean} cms
             * @return {can.Deferred}
             */
            request: function(data, isForceUserLogin) {
                var def = $q.defer();
                var that = this;
                // https://docs.angularjs.org/api/ng/service/$http
                // $http 接受的配置项有：
                // method 方法
                // url 路径
                // params GET请求的参数
                // data post请求的参数
                // headers 头
                // transformRequest 请求预处理函数
                // transformResponse 响应预处理函数
                // cache 缓存
                // timeout 超时毫秒，超时的请求会被取消
                // withCredentials 跨域安全策略的一个东西
                $http({
                    method: this.options.api.HTTP_METHOD ? this.options.api.HTTP_METHOD : 'POST',
                    url: this.options.api.PATH ? (CONST_REQUEST_URL + this.options.api.PATH) : CONST_REQUEST_URL,
                    params: this.options.api.HTTP_METHOD === "GET" ? data : '',
                    data: this.options.api.HTTP_METHOD === "POST" ? data : '',
                    cache: false,
                    timeout: 15000
                }).then(function successCallback(response) {
                    // this callback will be called asynchronously
                    // when the response is available
                    if (response.data.state && response.data.state.code == COMMON_RETURNCODE.PERMISSION_ERROR) {
                        //ACL权限申请
                        that.showAclApply(response);
                        def.reject(response.data);
                    } else if (response.data.state && response.data.state.code != COMMON_RETURNCODE.SUCCESS) {
                        notify({
                            message: 'code:' + response.data.state.code + ',msg:' + response.data.state.msg,
                            classes: 'alert-danger',
                            position: 'center',
                            templateUrl: CONST_NOTIFY_TPL
                        });
                        def.reject(response.data);
                    } else {
                        def.resolve(response.data.content);
                    }
                }, function errorCallback(response) {
                    // called asynchronously if an error occurs
                    // or server returns response with an error status.
                    notify({
                        message: 'server error,status:' + response.status,
                        classes: 'alert-danger',
                        templateUrl: CONST_NOTIFY_TPL
                    });
                    $log.error(JSON.stringify(response));
                    def.reject(response.data);
                });
                //同步数据返回
                return def.promise;
            }
        }
        return Gateway;
    })
```

## 其他封装
### commonService
common.service.js中提供了一些公用服务(业务)的封装

* getCinemas 封装了对影院信息调用的method,返回promise
* getAllPartner 带本地缓存的获取所有系统商信息方法
* getExportRecord 查看导出记录状态,返回promise
* getUserProfileFromCookie 从cookie中获取用户信息
* getUserProfile 调用接口返回用户信息,返回promise
* cleanUserProfile 清理localstorage cache的用户信息

### commonUtil
common.util.js中提供了一些公共函数库封装

* featureUtil.getValueByKey 同后端FeatureUtil.getValueByKey方法的前端实现，解析features中的属性
* commonUtil.showCommonDetail table展示数据详情使用的函数封装
``` js
使用示例
/**
 - 详情查看
 - @param  {[type]} row [description]
 - @return {[type]}     [description]
 */
$scope.showDetail = function(row) {
    commonUtil.showCommonDetail("导出记录详情", {
        'gmtCreate': {
            //表格数据描述
            desc: '执行时间',
            //数据处理回调
            format: function(col) {
                return new moment(col).format("YYYY-MM-DD HH:mm:ss");
            }
        }
    }, row/*数据*/, ['id']/*不需要展示的字段*/);
};
```

### 水印
tfsupport水印通过前端js实现，将cookies中的用户信息通过svg生成背景图设置到页面上。目前已经将这部分功能封装为directive，绑定了class="ibox-content"匹配。
``` html
使用示例:
<div class="ibox-content">...</div>
```
