---
title: WEB系统前后端分离设计(一)--后端部分
date: 2017-03-24 11:26:09
categories: 
- 编程
- 系统设计
tags:
- angularjs
- 系统设计
---
## 错误码设计
* 接口报文设计
``` js
{
    "state": {
        "code": -2,//接口返回码
        "cost": 8,//接口响应时间
        "features": ";aclPermissionName:demo;",//features,某些错误场景需要扩展信息以便处理，结构同服务端数据库features
        "msg": "权限不足",//错误提示
        "systime": 1488782151891,//请求时间点
        "traceId": "0ab20a3b14887821518837557dadb2-143-1488782151883"//traceId，同logback MDC中traceId
    },
    "content": {}//接口返回内容
}
```
<!-- more -->
* 后端错误码处理
扩展webx RenderResultAsJsonValve => spring mvc的话可以考虑扩展json view resolver
``` java
...
public class JsonResponseValve extends RenderResultAsJsonValve {
    private static final Logger logger = LoggerFactory.getLogger(JsonResponseValve.class);
    @Autowired
    private HttpServletRequest  request;
    @Autowired
    private HttpServletResponse response;
    private static final String TRACEID = "traceId";

    @Override
    public void invoke(PipelineContext pipelineContext) throws Exception {
        long start = System.currentTimeMillis();
        AbstractReturnCode abstractReturnCode = CommonReturnCode.SUCCESS;
        String features = null;
        Object resultObject = null;
        String traceId = generateTraceId();
        try {
            MDC.put(TRACEID, traceId);
            pipelineContext.invokeNext();
            TurbineRunData rundata = getTurbineRunData(request);
            if (!rundata.isRedirected()) {
                resultObject = consumeInputValue(pipelineContext);
            }
        } catch (ServiceRuntimeException serviceRuntimeException) {
            abstractReturnCode = serviceRuntimeException.getAbstractReturnCode();
            features = serviceRuntimeException.getFeatures();
            logger.error("error code:" + abstractReturnCode, serviceRuntimeException);
        } catch (PipelineException pipelineException) {
            abstractReturnCode = CommonReturnCode.INTERNAL_ERROR;
            if (pipelineException.getCause() instanceof WebxException) {
                WebxException webxException = (WebxException) pipelineException.getCause();
                if (webxException.getCause() instanceof ModuleEventException) {
                    ModuleEventException moduleEventException = (ModuleEventException) webxException.getCause();
                    if (moduleEventException.getCause() instanceof ServiceRuntimeException) {
                        ServiceRuntimeException serviceRuntimeException = (ServiceRuntimeException) moduleEventException
                                .getCause();
                        abstractReturnCode = serviceRuntimeException.getAbstractReturnCode();
                        features = serviceRuntimeException.getFeatures();
                    } else if (moduleEventException.getCause() instanceof ServiceException) {
                        ServiceException serviceException = (ServiceException) moduleEventException.getCause();
                        abstractReturnCode = serviceException.getAbstractReturnCode();
                        features = serviceException.getFeatures();
                    }
                }
            }
            logger.error("pipeline exception!", pipelineException);
        } catch (Exception e) {
            abstractReturnCode = CommonReturnCode.INTERNAL_ERROR;
            logger.error("unknow error!", e);
        } finally {
            MDC.remove(TRACEID);
            long cost = System.currentTimeMillis() - start;
            response.setContentType(getContentType());
            PrintWriter out = response.getWriter();
            ResponseWrapper responseWrapper = new ResponseWrapper();
            ResponseWrapper.State state = new ResponseWrapper.State.Builder().code(abstractReturnCode)
                    .systime(System.currentTimeMillis()).features(features).traceId(traceId).cost(cost).build();
            responseWrapper.setState(state);
            responseWrapper.setContent(resultObject);
            out.print(JSON.toJSONString(responseWrapper));
            ...
        }
    }
    ...
}
```
* 前端错误码处理
封装统一的请求处理模块，预先处理预定义的错误码，再返回promise
``` js
$http({
    ...
}).then(function successCallback(response) {
    // this callback will be called asynchronously
    // when the response is available
    if (response.data.state && response.data.state.code == COMMON_RETURNCODE.PERMISSION_ERROR) {
        //ACL权限申请
        that.showAclApply(response);
        def.reject(response.data);
    } else if (response.data.state && response.data.state.code != COMMON_RETURNCODE.SUCCESS) {
        ...//异常处理
        def.reject(response.data);
    } else {
        def.resolve(response.data.content);
    }
}, function errorCallback(response) {
    ...
    def.reject(response.data);
});
//同步数据返回
return def.promise;
}
```

## SSO方案
{% asset_img ssologin.jpg %}

{% asset_img ssologout.jpg %}

## ACL方案
前后端分离架构，权限控制主要作用在后端接口上，通过接口授权、前端提示达到权限控制目的。

* 接口权限声明
``` java
public class Export {
    ...
    //作用在webx action上,通过annotation指定访问的permission
    @Authorized(permission = PermissionEnum.TEST)
    public List<TkeExportRecordDO> doGetExportRecord() {
        ...
    }
    ...
}
```

* ACL配置
ACL授权体系 user <=1:n=> permission, role <=1:n=> permission, user <=n:n=> role

* 前端效果
{% asset_img img1.png %}

* 实现细节
对比基于webx valve实现方案ACLValve和基于spring aop的ACLInterceptor,两者的实现原理都是通过在调用链上添加处理逻辑，valve是采用的类似Filter机制，aop采用编译期织入切面实现。两者实现原理的明显优缺点并不突出，不过webx并未提供很明确的解析方式用来检索到对应请求资源的permission，这一点采用AOP实现相对而言简单很多。
PS：webx3的module的action上集成spring aop存在低版本兼容问题，需要升级spring至3.2.8以上版本，具体原因参看[webx Module无法使用springAOP的问题解决思路]
``` java
class ACLInterceptor implements MethodInterceptor {
    private static final Logger logger = LoggerFactory.getLogger(ACLInterceptor.class);
    @Autowired
    private AccessControlService accessControlService;
    @Value("#{constants['buc.acl.accesskey']}")
    private String               bucAccessKey;

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
        Authorized authorized = method.getAnnotation(Authorized.class);
        if (authorized != null) {
            if (checkAcl(authorized.permission().getPermission())) {
                return invocation.proceed();
            } else {
                throw new ServiceRuntimeException(CommonReturnCode.PERMISSION_ERROR).features(
                        CommonConst.ACL_PERMISSION_NAME_FEATURES_KEY + authorized.permission().getPermission());
            }
        } else {
            return invocation.proceed();
        }
    }

    /**
     * ACL校验
     */
    private boolean checkAcl(String permission) {
        TFSupportContext.User user = TFSupportContext.getCurrentUser();
        if (user == null) {
            throw new ServiceRuntimeException(CommonReturnCode.USER_NOTLOGIN_ERROR);
        }
        if (StringUtils.isNotEmpty(permission)) {
            CheckPermissionsParam checkPermissionsParam = new CheckPermissionsParam();
            checkPermissionsParam.setUserId(user.getUserId());
            List<String> permissions = new ArrayList<String>();
            permissions.add(permission);
            checkPermissionsParam.setPermissionNames(permissions);
            checkPermissionsParam.setAccessKey(bucAccessKey);
            CheckPermissionsResult result = null;
            result = accessControlService.checkPermissions(checkPermissionsParam);
            if (result != null && result.isSuccess() && !CollectionUtils.isEmpty(result.getCheckPermissionResults())) {
                for (CheckPermissionsResult.CheckPermissionResultInner checkPermissionDataResult : result
                        .getCheckPermissionResults()) {
                    if (!checkPermissionDataResult.isAccessible()) {
                        throw new ServiceRuntimeException(
                                "access denied! userId:" + user.getUserId() + " do not have permssion:" + permission,
                                CommonReturnCode.PERMISSION_ERROR).features(FeatureUtil.toString(
                                FeatureUtil.putFirst(CommonConst.ACL_PERMISSION_NAME_FEATURES_KEY, permission)));
                    }
                }
                return true;
            } else {
                logger.error("accessControlService.checkPermissions failed, check result is null or invalid! result:{}",
                        result);
                throw new ServiceRuntimeException(CommonReturnCode.ACL_CONFIG_ERROR);
            }
        } else {
            throw new ServiceRuntimeException(CommonReturnCode.ACL_CONFIG_ERROR);
        }
    }
}
```

* 前端实现，建议封装统一的请求模块，对于类似权限不足这种系统层面的错误进行统一处理。以下是实现的部分代码片段:
``` js
'use strict';
/**
 * baserequest Module
 *
 * Description
 */
angular
    .module('inspinia')
    .service('BaseRequest', function($http, $cookies, $q, $log, notify, featureUtil, ...) {
        ...
        var Gateway = function(options, data) {
            this.options = options;
            this.init(data);
        };

        Gateway.prototype = {

            /**
             * api的定义需要被每个子类复写
             * @type {Object}
             */
            api: {},
            ...
            
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
                        ...
                        def.reject(response.data);
                    } else {
                        def.resolve(response.data.content);
                    }
                }, function errorCallback(response) {
                    // called asynchronously if an error occurs
                    // or server returns response with an error status.
                    ...
                    def.reject(response.data);
                });
                //同步数据返回
                return def.promise;
            }
        }
        return Gateway;
    })
```

## 通用模块

### excel导出
{% asset_img img2.png %}

前端同步提交导出任务，异步loop+batchget方式获取数据回写oss private bucket；前端loop获取任务状态以及下载地址；

* 前端
``` js
/**
 - 数据导出
 - @return {[type]} [description]
 */
$scope.export = function() {
...
    var request = new BaseRequest(export_failed_lockseats, {
        startTime: moment($scope.startTime).format("YYYY-MM-DD HH:mm"),
        endTime: moment($scope.endTime).format("YYYY-MM-DD HH:mm")
    });
    request.sendRequest()
        .then(function(lastExportRecordId) {
            //使用localstorage存储最近一次导出的任务ID,再次进入页面时用来获取任务的执行状况
            orderMonitorStore.set("lastExportRecordIdOfFailedLockSeats", lastExportRecordId);
            loopCheckStatus(lastExportRecordId);
        }, function(data) {
            console.log(data);
            $scope.isExporting = false;
        });
    //loop检查任务执行状况，TODO 提升封装性
    function loopCheckStatus(lastExportRecordId) {
            var loopCount = 0;
            var nIntervId = setInterval(function() {
                if (loopCount < 100 && (!$scope.lastExportStatus || $scope.lastExportStatus == 'EXPORTING')) {
                    commonService.getExportRecord(lastExportRecordId)
                        .then(function(data) {
                            if (data) {
                                $scope.lastExportStatus = data[0].status;
                                $scope.lastExportName = data[0].fileName;
                                if ('FINISH' === $scope.lastExportStatus) {
                                    $scope.lastExportDownloadUrl = data[0].downloadUrl;
                                }
                            }
                            loopCount++;
                            $scope.lastExportProcess = $scope.lastExportStatus != 'EXPORTING' ?
                                100 : ((loopCount % 10) * 10 + 10);
                        }, function(data) {
                            $scope.lastExportStatus = "FAILED";
                        });
                } else {
                    $scope.$apply(function() { $scope.lastExportProcess = 100 });
                    clearInterval(nIntervId);
                }
            }, 1000);
        }
};
```

* 服务端
不同业务声明对于的数据加载逻辑

``` java
public interface IDataCollect<T> {
    public List<T> execute(Long maxId, Integer pageSize);

    public Long getId(T element);

    public Integer sizeOfLimit();
}

public Long exportFailedLockSeats(final Date beginTime, final Date endTime, final Long cinemaId) {
...
return excelExportService.exportExcel(fileName, Constant.ExportBizType.FAILED_LOCKSEATS_DETAIL, false,
        new ExcelExportServiceImpl.IDataCollect<TkeFailedLockSeatsStatisticsDO>() {
            //单次batch get
            @Override
            public List<TkeFailedLockSeatsStatisticsDO> execute(Long maxId, Integer pageSize) {
                return tkeFailedLockSeatsStatisticsDao
                        .batchGetFailedLockSeatsStatistics(beginTime, endTime, cinemaId, maxId, pageSize);
            }
            //游标
            @Override
            public Long getId(TkeFailedLockSeatsStatisticsDO element) {
                return element.getId();
            }
            //batch get size
            @Override
            public Integer sizeOfLimit() {
                return 1000;
            }
        }, excelColumnHeaders, excelColumnNames);
}
```

可复用的任务提交模块

``` java
@Service
public class ExcelExportServiceImpl{
    /**
     * 增量读数据库，全量写excel*
     *
     * @return
     */
    public <T> Long exportExcel(...) {
        ...
        //记录导出任务的任务编号
        Long id = exportRecordDao.addTkeExportRecord(tkeExportRecordDO);
        //提交任务，异步执行
        threadPoolTaskExecutor.execute(
                new ExportExcelExecutor(name, empId, nickNameCn, id, false, Constant.EXPORT_EXPIRE_SECONDS, dataCollect,
                        headers, fields));
        return id;
    }
}
```

可复用excel导出的任务执行逻辑，ps:excel支持的数据量在万级，如果数据量过大，建议采用拆分excel再合并成zip包的方式进行导出。

``` java
 /**
     * 异步导出
     */
    private class ExportExcelExecutor implements Runnable {
        final Logger logger = LoggerFactory.getLogger(ExportExcelExecutor.class);
        String                              name;
        Long                                exportRecordId;
        Integer                             expireSeconds;
        String[]                            headers;
        String[]                            fields;
        ExcelExportServiceImpl.IDataCollect dataCollect;

        ExportExcelExecutor(String name, String empId, String nickName, Long exportRecordId, Boolean isPublic,
                            Integer expireSeconds, ExcelExportServiceImpl.IDataCollect dataCollect, String[] headers,
                            String[] fields) {
            this.name = name;
            this.exportRecordId = exportRecordId;
            this.dataCollect = dataCollect;
            this.headers = headers;//excel表头
            this.fields = fields;//需要记入表格的对象属性
            this.expireSeconds = expireSeconds;//私有bucket有效期
        }

        @Override
        public void run() {
            Integer pageSize = dataCollect.sizeOfLimit();
            Long maxId = 0L;
            ByteArrayOutputStream outputStream = null;
            try {
                boolean hasMore = true;
                final List datas = new ArrayList();
                while (hasMore) {
                    List records = dataCollect.execute(maxId, pageSize);
                    if (records != null && records.size() != 0) {
                        maxId = dataCollect.getId(records.get(records.size() - 1));
                        datas.addAll(records);
                        if (datas.size() >= MAX_RECORD_SIZE_IN_SINGLE_EXCEL) {
                            //最多允许导出1w条数据，避免数据量过载
                            logger.warn("excel is too large, export abort,name:{},exportRecordId:{}", name,
                                    exportRecordId);
                            break;
                        }
                    }
                    if (records == null || records.size() < pageSize) {
                        hasMore = false;
                    }
                }
                //执行excel导出
                ...
                ExcelUtil.export2Excel(exportSetInfo);
                byte[] bytes = outputStream.toByteArray();
                OssServiceImpl.UploadResult uploadResult = null;
                if (bytes != null) {
                    //写oss 
                    uploadResult = ossService.uploadFile(name + Constant.EXCEL_FILE_SUFFIX, expireSeconds,
                            Constant.EXCEL_CONTENT_TYPE, bytes);
                }
                TkeExportRecordDO tkeExportRecordDO = new TkeExportRecordDO();
                tkeExportRecordDO.setId(exportRecordId);
                if (uploadResult != null) {
                    tkeExportRecordDO.setStatus(ExportStatus.FINISH.toString());
                    tkeExportRecordDO.setDownloadUrl(uploadResult.getDownloadUrl());
                    ...
                } else {
                    ...
                    tkeExportRecordDO.setStatus(ExportStatus.FAILED.toString());
                }
                exportRecordDao.updateTkeExportRecord(tkeExportRecordDO);
            } catch (Exception e) {
                ...
            } finally {
                ...
            }
        }
    }
```

### 预警事件
通过spring event，解耦了预警事件的判断发送等环节。
1.数据任务,触发ALARM_EVENT
=>2.执行alarm判断逻辑,判断是否需要触发ALARM_MSG_EVENT
=>3.发送alarm消息
其中2、3部可采用异步方式执行，避免预警等后置动作阻塞数据任务执行；
spring配置

``` xml
<!--预警异步处理线程池-->
<bean id="alarmThreadPoolTaskExecutor" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
    <property name="corePoolSize" value="5"/>
    <property name="keepAliveSeconds" value="300"/>
    <property name="maxPoolSize" value="30"/>
    <property name="queueCapacity" value="10000"/>
    <property name="rejectedExecutionHandler">
        <bean class="java.util.concurrent.ThreadPoolExecutor$CallerRunsPolicy"/>
    </property>
</bean>
<task:annotation-driven executor="alarmThreadPoolTaskExecutor" proxy-target-class="true"/>
```

事件触发

``` java
...
LockSeatsAlarmMsgSendEvent lockSeatsAlarmMsgSendEvent = new LockSeatsAlarmMsgSendEvent(
        getClass().getName());
...
applicationContext.publishEvent(lockSeatsAlarmMsgSendEvent);
```

注册事件监听

``` java
/**
 * 预警通知发送事件监听
 * @author - zhupin(kaiqiang.gkq@alibaba-inc.com)
 */
@Component
public class LockSeatsAlarmMsgSendEventListener implements ApplicationListener<LockSeatsAlarmMsgSendEvent> {
    @Autowired
    private MessageServiceImpl messageService;
    //异步执行
    @Async
    @Override
    public void onApplicationEvent(LockSeatsAlarmMsgSendEvent lockSeatsAlarmEvent) {
        ...
    }
}
```

