---

title: "状态机篇二：简易状态机实现"
slug: "状态机篇二：简易状态机实现"
description:
date: "2024-09-15T09:41:29+08:00"
lastmod: "2024-09-15T09:41:29+08:00"
image: cover.png
math:
license:
hidden: false
draft: false
categories: ["状态机"]
tags: ["简易状态机实现"]

---

> 封面为《凡人修仙传》118集元瑶出浴，现在已经被剪掉了


了解了基础的状态机概念后，来实现一个简单的状态机，包含几大组件：状态State、转变Transition、事件Event、动作Action、守护条件Condition等概念组件。

## 状态State
表示状态可以是int、string、enum等类型的值，所以定义一个接口，获取泛型的值。

```java
public interface FsmState<T> {

    T value();
    
}
```



## 事件Event
事件和State类似的，都可以用int、string、enum等表示。

```java
public interface FsmEvent<T> {

    T value();

}
```



## 事件和状态的三元组
定义一个三元组，来描述当前状态，发生了什么事情，目标状态是什么。

```java
@Getter
@AllArgsConstructor
public class FsmTransitionUnit<S,E> {

    /**
     * 现态
     */
    private FsmState<S> sourceState;

    /**
     * 关联的事件
     */
    private FsmEvent<E> event;

    /**
     * 次态
     */
    private FsmState<S> targetState;

}
```



## 状态流转上下文
当发生了事件后，会触发状态流转Transition，而Transition中需要感知到当前的上下文情况

+ 当前状态是什么sourceState
+ 发生了什么事情Event
+ 流转到的目标状态是什么targetState
+ 除了状态机本身需要的上下文外，还允许携带一些自定义的上下文数据对象

```java
public interface FsmStateContext<S, E, C> {

    /**
     * 当前状态
     */
    FsmState<S> sourceState();

    /**
     * 次态（目标状态）
     */
    FsmState<S> targetState();

    /**
     * 触发事件
     */
    FsmEvent<E> event();

    /**
     * 自定义上下文对象
     */
    C context();

}
```



## 状态流转Transition
状态流转Transition其实是多个动作的组合

+ onCondition：根据上下文判断是否可以执行Action，可以没有
+ onAction：执行具体的Action
+ afterAction：执行完Action之后可以做一些后置处理，可以没有

可以看到这三个方法的参数都是`FsmStateContext`，可以获取到状态机自身的状态情况，以及自定义的上下文数据对象。

```java
public interface FsmStateTransition<S, E, C> {

    /**
     * 执行动作之前，判断状态是否运行执行
     *
     * @return 是否可以执行transition
     */
    default boolean onCondition(FsmStateContext<S, E, C> fsmStateContext) {
        return true;
    }

    /**
     * 实际的动作执行
     */
    void onAction(FsmStateContext<S, E, C> fsmStateContext);

    /**
     * action执行完成之后的处理, 做一些收尾动作
     */
    default void afterAction(FsmStateContext<S, E, C> fsmStateContext) {
    }

}
```



## 状态机StateMachine
状态机实例主要负责两个功能

+ 提前配置好相关状态之间的流转关系
+ 触发某事件来进行状态流转

```java
public interface FsmStateMachine<S, E> {

    /**
     * 注册转换Transition
     */
    void registerTransition(FsmTransitionUnit<S, E> transitionUnit, FsmStateTransition<S, E, ?> transition);

    /**
     * 触发状态变更的事件
     */
    boolean fireEvent(FsmState<S> sourceState, FsmEvent<E> event, Object context);

}
```



一个简易状态机的实现类比较简单，主要是在状态机内部需要维护状态流转的关系，下面是几个设计关键点

+ 这里把<sourceState, event >拼接起来做为一个key，来唯一标识一个Transition
+ 而外维护一个<sourceState, event> 到 targetState的关系
+ 内部简单实现了FsmStateContext，不对外暴露具体的实现

具体代码如下

```java
public class DefaultStateMachine<S, E> implements FsmStateMachine<S, E> {

    private static final String SEPARATOR = "_TO_";
    ConcurrentHashMap<String, FsmStateTransition<S, E,?>> stateTransitionMap = new ConcurrentHashMap<>();
    ConcurrentHashMap<String, FsmState<S>> targetStateMap = new ConcurrentHashMap<>();


    @Override
    public synchronized void registerTransition(FsmTransitionUnit<S, E> transitionUnit, FsmStateTransition<S, E, ?> transition) {
        Objects.requireNonNull(transitionUnit, "transitionUnit cannot be null");
        Objects.requireNonNull(transition, "transition cannot be null");
        Objects.requireNonNull(transitionUnit.getSourceState(), "sourceState cannot be null");
        Objects.requireNonNull(transitionUnit.getEvent(), "event cannot be null");
        Objects.requireNonNull(transitionUnit.getTargetState(), "targetState cannot be null");

        String key = transitionUnit.getSourceState().value().toString() + SEPARATOR + transitionUnit.getEvent().value().toString();
        if (stateTransitionMap.containsKey(key)) {
            throw new IllegalArgumentException("transitionUnit already registered");
        }
        stateTransitionMap.put(key, transition);
        targetStateMap.put(key, transitionUnit.getTargetState());
    }

    @Override
    public boolean fireEvent(FsmState<S> sourceState, FsmEvent<E> event, Object context) {
        Objects.requireNonNull(sourceState, "sourceState cannot be null");
        Objects.requireNonNull(event, "event cannot be null");

        String key = sourceState.value().toString() + SEPARATOR + event.value().toString();
        FsmStateTransition stateTransition = stateTransitionMap.get(key);
        if (stateTransition == null) {
            return false;
        }
        FsmState<S> targetState = targetStateMap.get(key);
        DefaultStateContext<S, E, Object> contextImpl = new DefaultStateContext<>(sourceState,
                targetState, event, context);
        boolean conditionResult = stateTransition.onCondition(contextImpl);
        if (conditionResult) {
            stateTransition.onAction(contextImpl);
            stateTransition.afterAction(contextImpl);
            return true;
        }
        return false;
    }

    @AllArgsConstructor
    static class DefaultStateContext<S,E, C> implements FsmStateContext<S,E, C> {

        private FsmState<S> source;

        private FsmState<S> target;

        private FsmEvent<E> event;

        private C context;

        @Override
        public FsmState<S> sourceState() {
            return source;
        }

        @Override
        public FsmState<S> targetState() {
            return target;
        }

        @Override
        public FsmEvent<E> event() {
            return event;
        }

        @Override
        public C context() {
            return context;
        }

    }

}
```





## 实践demo
基于支付单的模型来，在上述设计上做个实践demo。



支付单的状态设计：初始、待支付、已支付、已退款、已取消

```java
public enum StateEnum implements FsmState<StateEnum> {
    
    INIT, WAIT_PAY, PAYED, REFUND, CANCEL;

    @Override
    public StateEnum value() {
        return this;
    }
}
```



支付单相关事件：创建事件、支付事件、取消时间、退款事件

```java
public enum EventEnum implements FsmEvent<EventEnum> {

    CREATE_EVENT,
    PAY_EVENT,
    CANCEL_EVENT,
    REFUND_EVENT;

    @Override
    public EventEnum value() {
        return this;
    }
}
```



定义各种Transition，这里以处理支付事件为例，其余的都类似，只是简单打印日志

```java
@Slf4j
public class PayedTransition implements FsmStateTransition<StateEnum, EventEnum, String> {

    @Override
    public void onAction(FsmStateContext<StateEnum, EventEnum, String> fsmStateContext) {
        log.info("orderNo={} pay success", fsmStateContext.context());
    }

}
```



接下来初始化StateMachine实例，并配置状态流转过程，简单的说就是配置4个元素

+ 当前状态
+ 发生的事件
+ 目标状态
+ 需要执行的动作

```java
FsmStateMachine<StateEnum, EventEnum> fsmStateMachine = new DefaultStateMachine<>();
fsmStateMachine.registerTransition(new FsmTransitionUnit<>(StateEnum.INIT, EventEnum.CREATE_EVENT, StateEnum.WAIT_PAY), new CreatePayOrderTransition());
fsmStateMachine.registerTransition(new FsmTransitionUnit<>(StateEnum.WAIT_PAY, EventEnum.PAY_EVENT, StateEnum.PAYED), new PayedTransition());
fsmStateMachine.registerTransition(new FsmTransitionUnit<>(StateEnum.WAIT_PAY, EventEnum.CANCEL_EVENT, StateEnum.CANCEL), new CancelTransition());
fsmStateMachine.registerTransition(new FsmTransitionUnit<>(StateEnum.PAYED, EventEnum.REFUND_EVENT, StateEnum.REFUND), new RefundTransition());

```



接下来触发事件

```java
// 正常的状态流转
fsmStateMachine.fireEvent(StateEnum.INIT, EventEnum.CREATE_EVENT, "12345678");
fsmStateMachine.fireEvent(StateEnum.WAIT_PAY, EventEnum.PAY_EVENT, "12345678");
fsmStateMachine.fireEvent(StateEnum.WAIT_PAY, EventEnum.CANCEL_EVENT, "12345678");
fsmStateMachine.fireEvent(StateEnum.PAYED, EventEnum.REFUND_EVENT, "12345678");

// 异常流转
boolean res = fsmStateMachine.fireEvent(StateEnum.INIT, EventEnum.REFUND_EVENT, "12345678");
if (!res) {
    log.error("This event={} should not occur in the state={}", EventEnum.REFUND_EVENT,StateEnum.INIT);
}
```



能够正常流转的状态，都通过Transition执行并打印了日志。

```java
10:37:56.810 [main] INFO com.example.stateMachine.test.CreatePayOrderTransition - orderNo=12345678 pay order create success
10:37:56.812 [main] INFO com.example.stateMachine.test.PayedTransition - orderNo=12345678 pay success
10:37:56.812 [main] INFO com.example.stateMachine.test.CancelTransition - orderNo=12345678 cancel success
10:37:56.812 [main] INFO com.example.stateMachine.test.RefundTransition - orderNo=12345678 refund success
10:37:56.812 [main] ERROR com.example.stateMachine.test.StateMachineTest - This event=REFUND_EVENT should not occur in the state=INIT
```



## 总结
基于有限状态机FSM的基础概念来实现一个简单的状态机，囊括了基本的功能：State、Event、Transition、Condition、Action等，可以实现基本的状态流转。这里还是缺少了一些东西，如InternalTransition，更多类型的Action等。在状态机配置上，缺少了一些语义化层面的东西，不够直观。后边会对SpringStateMachine、COLAStateMachine进行介绍。

## 附录

### 参考
