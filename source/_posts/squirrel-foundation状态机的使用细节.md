---
title: squirrel-foundation状态机的使用细节
date: 2017-06-21 16:59:11
categories: 
- 编程
- 系统设计
- 状态机引擎
tags:
- FSM
- 状态机引擎
- finite state machine
- squirrel-foundation
- squirrel
- spring
---
上一篇[文章](http://www.timguan.net/2017/06/19/%E7%8A%B6%E6%80%81%E6%9C%BA%E5%BC%95%E6%93%8E%E9%80%89%E5%9E%8B/)介绍了stateless4j、spring-statemachine以及squirrel-foundation三款状态机引擎的实现原理，以及我为何选择squirrel-foundation作为解决方案。本文主要介绍一下项目中如何使用squirrel-foundation的一些细节以及如何与spring进行集成。在阅读本文前，建议先阅读官方的[使用手册](http://hekailiang.github.io/squirrel/)。
<!-- more -->

# 生命周期

## 状态机创建过程
* StateMachine: StateMachine实例由StateMachineBuilder创建不被共享，对于使用annotation方式(或fluent api)定义的StateMachine，StateMachine实例即根据此定义创建，相应的action也由本实例执行，与spring的集成最终要的就是讲spring的bean实例注入给由builder创建的状态机实例；
* StateMachineBuilder: 本质上是由StateMachineBuilderFactory创建的动态代理。被代理的StateMachineBuilder默认实现为StateMachineBuilderImpl，内部描述了状态机实例创建细节包括State、Event、Context类型信息、constructor等，同时也包含了StateMachine的一些全局共享资源包括StateConverter、EventConverter、MvelScriptManager等。StateMachineBuilder可被复用，使用中可被实现为singleton；
* StateMachineBuilderFactory: 为StateMachineBuilder创建的动态代理实例；

## 事件处理过程
* 状态正常迁移
TransitionBegin--(exit->transition->entry)-->TransitionComplete-->TransitionEnd
* 状态迁移异常
TransitionBegin--(exit->transition->entry)-->TransitionException-->TransitionEnd
* 状态迁移事件拒绝
TransitionBegin-->TransitionDeclined-->TransitionEnd

{% qnimg statemachine-lifecycle.jpeg extend:?imageView2/2/w/600 title:"statemachine lifecycle" %}

# spring集成
从statemachine的生命流程上可以看到，StateMachineBuilder可以单例方式由spring container管理，而stateMachine的instance的生命周期伴随着请求(或业务)。
从这两点出发，集成spring需要完成两件事：
* (1).通过Spring创建StateMachineBuilder实例；
* (2).业务函数中通过(1)的StateMachineBuilder实例创建StateMachine实例，并向StateMachine暴露SpringApplicationContext；

泛型参数+覆盖默认构造函数隐藏StateMachineBuilder创建细节，实现ApplicationContextAware接口，接受applicationContext注入，并注入给stateMachine实例。
``` java
public abstract class AbstractStateMachineEngine<T extends UntypedStateMachine> implements ApplicationContextAware {
    protected UntypedStateMachineBuilder stateMachineBuilder = null;
    @SuppressWarnings("unchecked")
    public AbstractStateMachineEngine() {
    	//识别泛型参数
        Class<T> genericType = (Class<T>)GenericTypeResolver.resolveTypeArgument(getClass(),
            AbstractStateMachineEngine.class);
        stateMachineBuilder = StateMachineBuilderFactory.create(genericType, ApplicationContext.class);
    }
    //注入applicationContext，并在创建StateMachine实例时注入
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
    //delegate fire
    public void fire(int rmaId, State initialState, Trigger trigger, StateMachineContext context) {
    	T stateMachine = stateMachineBuilder.newUntypedStateMachine(
    						initialState
                            //暂时开启debug进行日志trace
                            StateMachineConfiguration.create().enableDebugMode(true).enableAutoStart(true),
                            //注入applicationContext
                            applicationContext);
        stateMachine.fire(trigger, context);
    }
    ...
}
@Service
class DiscountRefundStateMachineEngine extends AbstractStateMachineEngine<DiscountRefundStateMachine> {
}
@Service
public class ReturnGoodsStateMachineEngine extends AbstractStateMachineEngine<ReturnGoodsStateMachine> {
}
```
StateMachine定义，接受SpringContext注入
``` java
@StateMachineParameters(stateType = State.class, eventType = Trigger.class,
    //StateMachineContext 自定义上下文，用来传递数据
    contextType = StateMachineContext.class)
@States({
    @State(name = "PENDING", initialState = true),
    @State(name = "CONFIRMING"),
    @State(name = "REJECTED"),
    @State(name = "REFUND_APPROVING"),
    @State(name = "REFUND_APPROVED"),
    @State(name = "REFUND_FINISHED")
})
@Transitions({
    @Transit(from = "PENDING", to = "CONFIRMING", on = "APPLY_CONFIRM",
        callMethod = "doSomething"),
    @Transit(from = "CONFIRMING", to = "REJECTED", on = "REJECT"),
    @Transit(from = "CONFIRMING", to = "REFUND_APPROVING", on = "APPLY_APPROVED"),
    @Transit(from = "REFUND_APPROVING", to = "REFUND_APPROVED", on = "REFUND_APPROVED"),
    @Transit(from = "REFUND_APPROVED", to = "REFUND_FINISHED", on = "REFUND_FINISH_CONFIRM")
})
public class DiscountRefundStateMachine extends AbstractUntypedStateMachine {
	protected ApplicationContext applicationContext;
	//定义构造函数接受ApplicationContext注入([参看New State Machine Instance](http://hekailiang.github.io/squirrel/))
    public DiscountRefundStateMachine(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }
    public void doSomething(State fromState, State toState, Trigger event,
                         StateMachineContext stateMachineContext) {
         DemoBean demoBean = this.applicationContext.get("demoBean");
         //do something
    }
    ...
}
```

# 状态持久化
从StateMachine的事件响应流程中可以看到，TransitionBegin--(exit->transition->entry)-->TransitionComplete-->TransitionEnd，在TransitionComplete发生一个状态已从source迁移到了target状态，所以我选择了在这个时间点进行了状态的持久化（没有选择TransitionEnd做持久化，因为某些场景在持久化完成后还会存在一些外部动作的触发，例如通知第三方系统当前状态已完成变更）。

``` java
public class DiscountRefundStateMachine extends AbstractUntypedStateMachine {
    ..
    @Override
    protected void afterTransitionCompleted(Object fromState, Object toState, Object event, Object context) {
        if (context instanceof StateMachineContext && toState instanceof State) {
            StateMachineContext stateMachineContext = (StateMachineContext)context;
            //从上下文中获取需要持久化的数据，例如订单ID等
            Rma rma = stateMachineContext.get(MessageKeyEnum.RMA);
            //持久化
            rma.setStatus((State)toState);
            this.applicationContext.get("rmaRepository").updateRma(rma);
        } else {
            throw new Exception("type not support, context expect " + StateMachineContext.class.getSimpleName() + ", actually "
                    + context.getClass().getSimpleName() + ", state expect " + State.class.getSimpleName()
                    + ", actually "
                    + toState.getClass().getSimpleName());
        }
    }
}
```

# 分布式锁+事务
由于StateMachine实例不是由Spring容器创建，所以这个过程中无法通过注解方式开启事务(Spring没有机会去创建事务代理)，我采用了编程式事务，在AbstractStateMachineEngine的fire函数中隐式的实现。
AbstractStateMachineEngine#fire
``` java
public abstract class AbstractStateMachineEngine<T extends UntypedStateMachine> implements ApplicationContextAware {
    ...
    public void fire(int rmaId, State initialState, Trigger trigger, StateMachineContext context) {
        JedisLock jedisLock = jedisLockFactory.buildLock(rmaId);
        //争用分布式锁
        if (jedisLock.tryLock()) {
            try {
                T stateMachine = stateMachineBuilder.newUntypedStateMachine(
                                    initialState
                                    //暂时开启debug进行日志trace
                                    StateMachineConfiguration.create().enableDebugMode(true).enableAutoStart(true),
                                    //注入applicationContext
                                    applicationContext);
                DataSourceTransactionManager transactionManager = applicationContext.get("transactionManager")
                DefaultTransactionDefinition def = new DefaultTransactionDefinition();
                def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
                TransactionStatus status = transactionManager.getTransaction(def);
                try {
                    stateMachine.fire(trigger, context)
                    transactionManager.commit(status);
                } catch (Exception ex) {
                    transactionManager.rollback(status);
                    throw ex;
                }
            } finally {
                jedisLock.release();
            }
        } 
        ...
    }
}
```

# 使用graphviz生成状态拓扑图
squirrel statemachine提供了DotVisitor、SCXMLVisitor两种实现方式用于生成状态机描述文件，项目里我选择了graphviz用来做状态拓扑
[graphviz gui工具下载](http://ortsyq47e.bkt.clouddn.com/qnsource/images/graphviz-2.40.1.pkg)
PS:由于squirrel默认的DotVisitorImpl对带中文描述属性的State/Event枚举不友好，我在原有代码上做了一些调整，有类似需求的可以看[这里](https://github.com/TimGuan/squirrel/tree/developer/timguan)
{% qnimg returnGoods.jpg title:"退货流程" %}

以上是我在落地状态机改造过程中的一些细节，欢迎大家留意讨论。