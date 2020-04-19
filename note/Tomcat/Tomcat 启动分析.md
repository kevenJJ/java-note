# Tomcat 启动分析

## Lifecycle

Lifecycle 接口是一个公用的接口，定义了组件生命周期的一些方法，用于启动、停止Catalina组件。是一个非常重要的接口，组件的生命周期包括：inti、start、stop、destory，以及各种事件的常量、操作LifecycleListener 的API，是典型的观察者模式。

```java
public interface Lifecycle {

    // ----------------------- 定义各种EVENT事件 -----------------------

    public static final String BEFORE_INIT_EVENT = "before_init";
    public static final String AFTER_INIT_EVENT = "after_init";
    public static final String START_EVENT = "start";

    // 省略事件常量定义……

    /**
     * 注册一个LifecycleListener
     */
    public void addLifecycleListener(LifecycleListener listener);

    /**
     * 获取所有注册的LifecycleListener
     */
    public LifecycleListener[] findLifecycleListeners();

    /**
     * 移除指定的LifecycleListener
     */
    public void removeLifecycleListener(LifecycleListener listener);

    /**
     * 组件被实例化之后，调用该方法完成初始化工作，发会出以下事件
     * <ol>
     *   <li>INIT_EVENT: On the successful completion of component initialization.</li>
     * </ol>
     * @exception LifecycleException if this component detects a fatal error
     *  that prevents this component from being used
     */
    public void init() throws LifecycleException;

    /**
     * 在组件投入使用之前调用该方法，先后会发出以下事件：BEFORE_START_EVENT、START_EVENT、AFTER_START_EVENT
     * @exception LifecycleException if this component detects a fatal error
     *  that prevents this component from being used
     */
    public void start() throws LifecycleException;

    /**
     * 使组件停止工作
     */
    public void stop() throws LifecycleException;

    /**
     * 销毁组件时被调用
     */
    public void destroy() throws LifecycleException;

    /**
     * Obtain the current state of the source component.
     */
    public LifecycleState getState();

    /**
     * 获取state的文字说明
     */
    public String getStateName();

    /**
     * Marker interface used to indicate that the instance should only be used
     * once. Calling {@link #stop()} on an instance that supports this interface
     * will automatically call {@link #destroy()} after {@link #stop()}
     * completes.
     */
    public interface SingleUse {
    }
}
```

各大组件均实现了 Lifecycle 接口，类图如下所示：

![Tomcat-Lifecycle](C:\Users\Administrator\Desktop\note\img\Tomcat-Lifecycle-01.png)

* LifecycleBase：实现了Lifecycle 的 init、start、stop等主要逻辑，向注册在 LifecycleBase 内部的 LifecycleListener 发出对应的事件，并且预留了 initInternal、startInternal、stopInternal 等模板方法，便于子类完成自己的逻辑。
* MBeanRegistration：JMxEnabled