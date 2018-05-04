---
title: 状态机实践
date: 2018-04-25 09:07:09
tags: [JAVA,状态机]
categories: 架构
---
物联网的关键是设备产生事件之后，以IOT组件作为管道，发送至服务端，由服务端根据事件cache管理设备的状态。同时，服务端根据外部事件或者新状态，向设备下发新的指令（事件），由设备向最终用户提供服务。 目前，IOT组件可以使用 LVS+NETTY, MQTT，也有许多云端 NB-IOT平台。IOT平台不是本文阐述重点，本文重点是状态机。

状态机的问题在于状态通常和业务逻辑纠缠在一起，如果不能很好的解耦，那么代码会变得复杂，混沌。

我提出的解决方案是，状态实例化，每个状态对象里封装了适应他们的动作行为，以及切换规则。设备以内部事件的形式，响应外部事件。通过内部事件，执行相对应动作。解耦业务逻辑，设备状态，事件。

代码如下：

- 设备
```
public class Device implements Cloneable  {

    private ConcurrentHashMap<Class<? extends DeviceStatus>,DeviceStatus> statusMap = new ConcurrentHashMap<>() ;

    private String deviceId  ;
    private String deviceName ;
    private String data ;
    private DeviceStatus deviceStatus;
    private DeviceType deviceType;

    public Device(String id,String name){
        this.deviceId = id ;
        this.deviceName = name ;
    }

    /**
     * 初始化设备状态
     * @param statusClass   设备状态
     * @return  this
     */
    public Device initStatus(Class<? extends DeviceStatus> statusClass){
        Optional.ofNullable( statusMap.get(statusClass) ).ifPresent(status->setDeviceStatus(status)) ;
        return this ;
    }

    /**
     * 注册设备类型
     * @param deviceType    设备类型
     * @return  this
     */
    public Device deviceType(DeviceType deviceType){
        this.deviceType = deviceType;
        return this ;
    }

    /**
     * 绑定事件处理器
     *
     * @param deviceEvent   事件
     * @param handler       处理器
     * @return              this
     */
    public Device bindHandler(Class<? extends Event> deviceEvent, Consumer<Device> handler){
        Optional.ofNullable(deviceType).ifPresent(type->type.bind(deviceEvent,handler));
        return this ;
    }

    /**
     * 注册设备状态
     * @param deviceStatus  设备状态
     * @return      this
     */
    public Device deviceStatus(DeviceStatus deviceStatus){
        statusMap.putIfAbsent(deviceStatus.getClass(), deviceStatus) ;
        return this ;
    }

    /**
     * 绑定旧状态执行完逻辑后的新状态
     *
     * @param oldStatus     旧状态
     * @param deviceEvent   事件
     * @param newStatus     新状态
     * @return              this
     */
    public Device bindStatusTrans(Class<? extends DeviceStatus> oldStatus, Class<? extends Event>   deviceEvent, Class<? extends DeviceStatus>  newStatus){
        statusMap.get(oldStatus).registerStatusTrans(deviceEvent,statusMap.get(newStatus));
        return this ;
    }

    /**
     * 执行事件处理器
     * @param deviceEvent       事件
     * @throws OperationNoPermittedException    当前状态遇到非法事件
     * @throws ExecuteException                 执行异常
     */
    public void on(Class<? extends Event> deviceEvent) throws OperationNoPermittedException,ExecuteException {
        Optional<DeviceStatus> deviceNewStatusOptional = Optional.ofNullable(deviceStatus.transition(deviceEvent)) ;
        if(!deviceNewStatusOptional.isPresent()){
            //抛异常.这种状态不能处理这种事件
            throw new OperationNoPermittedException() ;
        }
        deviceType.execute(deviceEvent,this);
        setDeviceStatus(deviceNewStatusOptional.get()) ;
    }

    /**
     * 清空
     *
     */
    public void clear(){
        statusMap.values().forEach(status->status.clear());
        statusMap.clear();
    }

    //...省略
}
```

- 设备状态
```
/**
 * 设备状态
 * @author shenx
 * @version 1.0.0 2018/4/12
 */
public abstract class DeviceStatus {

    public enum DeviceStatusEnumClass {

        Busy,Error,Idle,Init,Nothing
    }

    private ConcurrentHashMap<Class<? extends Event>,DeviceStatus> statusTransMap = new ConcurrentHashMap<>() ;

    /**
     * 取设备状态
     * @return  设备状态
     */
    public abstract DeviceStatusEnumClass getDeviceStatus() ;

    /**
     * 注册事件执行成功后的状态
     * @param eventClass    事件
     * @param nextStatus    成功后状态
     */
    public void registerStatusTrans(Class<? extends Event> eventClass , DeviceStatus nextStatus){
        statusTransMap.putIfAbsent(eventClass,nextStatus) ;
    }

    /**
     * 状态变换
     * @param deviceEvent   事件
     * @return              新状态
     */
    public DeviceStatus transition(Class<? extends Event> deviceEvent) {
        return statusTransMap.getOrDefault(deviceEvent,null) ;
    }

    public String toString(){
        return getDeviceStatus().toString() ;
    }

    public void clear(){
        statusTransMap.clear();
    }
}
```
- 设备类型
```
public abstract class DeviceType {

    public enum DeviceTypeEnumClass {
        Box
    }

    private ConcurrentHashMap<Class<? extends Event>,Consumer<Device>> executeHandlerMap = new ConcurrentHashMap<>() ;

    /**
     * 取设备类型
     * @return  设备类型
     */
    public abstract DeviceTypeEnumClass getDeviceType() ;

    /**
     * 绑定事件与执行处理器
     * @param eventClass        事件
     * @param executeHandler    处理器
     */
    public void bind(Class<? extends Event> eventClass , Consumer<Device> executeHandler){
        executeHandlerMap.putIfAbsent(eventClass,executeHandler) ;
    }

    /**
     * 运行事件的处理器
     * @param deviceEvent       事件
     * @param device            设备
     * @throws ExecuteException 执行异常
     */
    public void execute(Class<? extends Event> deviceEvent, Device device) throws ExecuteException {
        try {
            Optional.ofNullable(executeHandlerMap.get(deviceEvent)).ifPresent(e->e.accept(device));
        }
        catch (Exception ex){
            throw new ExecuteException(ex) ;
        }
    }
    public String toString(){
        return getDeviceType().toString() ;
    }
}
```
- Context

```
/**
 * Device运行Context
 *
 * @author shenx
 * @version 1.0.0 2018/4/13
 */
public class DeviceContext {

    private RowLock<String> rowLock = new RowLock<>() ;

    private ConcurrentHashMap<String , Device> deviceConcurrentHashMap = new ConcurrentHashMap<>() ;

    private DeviceEventSource onRegistDeviceSource ;
    private DeviceEventSource onEventEventSource ;
    private DeviceEventSource onRemoveEventSource ;

    /**
     * 初始化注册，执行，取消注册队列
     */
    public DeviceContext(){
        // 初始化 创建设备
        Flux<Event<Device>> registDeviceFlowBridge = Flux.create(sink -> this.onRegistDeviceSource = new DeviceEventSource() {
                @Override
                public void onRegister(Event< Device > workFlowEvent) {
                    sink.next(workFlowEvent) ;
                }
            }
        ) ;
        registDeviceFlowBridge.publishOn(Schedulers.elastic()).subscribe(this::registerDevice) ;
        //  处理 设备事件
        Flux<Event<Device>> eventDeviceFlowBridge = Flux.create(sink -> this.onEventEventSource = new DeviceEventSource() {
                    @Override
                    public void onEvent(Event< Device > workFlowEvent) {
                        sink.next(workFlowEvent) ;
                    }
                }
        ) ;
        eventDeviceFlowBridge.publishOn(Schedulers.elastic()).subscribe(this::eventDevice) ;

        //  移除 设备
        Flux<Event<Device>> removeDeviceFlowBridge = Flux.create(sink -> this.onRemoveEventSource = new DeviceEventSource() {
                    @Override
                    public void onRemove(Event< Device > workFlowEvent) {
                        sink.next(workFlowEvent) ;
                    }
                }
        ) ;
        removeDeviceFlowBridge.publishOn(Schedulers.elastic()).subscribe(this::removeDevice) ;
    }

    ///

    /**
     * 注册设备
     * @param device    设备
     */
    public void onRegister(Device device){
        onRegistDeviceSource.onRegister(new Event<>(device));
    }

    /**
     * 取消注册设备
     * @param device    设备
     */
    public void onRemove(Device device){
        onRemoveEventSource.onRemove(new Event<>(device));
    }

    /**
     * 执行设备事件
     * @param deviceEvent   设备
     */
    public void onEvent(Event<Device> deviceEvent){
        onEventEventSource.onEvent(deviceEvent);
    }

    /**
     * 注册
     * @param deviceEvent   事件
     */
    private void registerDevice(Event<Device> deviceEvent){
        Device device = deviceEvent.getData() ;
        deviceConcurrentHashMap.putIfAbsent(device.getDeviceId(),device) ;
    }

    ///

    /**
     * 取消设备
     * @param deviceEvent   事件
     */
    private void removeDevice(Event<Device> deviceEvent){
        Device device = deviceEvent.getData() ;
        String deviceId = device.getDeviceId() ;
        logicWithLock(deviceId,device1 ->  {
            device1.clear();
            deviceConcurrentHashMap.remove(device1.getDeviceId());
        }) ;
    }

    /**
     * 执行设备事件
     * @param deviceEvent   事件
     */
    private void eventDevice(Event<Device> deviceEvent){
        Device device = deviceEvent.getData() ;
        String deviceId = device.getDeviceId() ;
        logicWithLock(deviceId,device1 ->  device1.on(deviceEvent.getClass())) ;
    }

    public interface DeviceConsumer<T, E extends Exception> {
        void accept(T t) throws E;
    }

    /**
     * 执行逻辑.对deviceId加锁.
     * @param deviceId      设备编号
     * @param lockLogic     执行逻辑
     */
    private void logicWithLock(String deviceId , DeviceConsumer<Device,Exception> lockLogic){
        try {
            Device deviceMapInstance = deviceConcurrentHashMap.get(deviceId) ;
            rowLock.lock(deviceId);
            lockLogic.accept(deviceMapInstance);
        }
        catch (RowLockException | OperationNoPermittedException | ExecuteException  ex){
            ex.printStackTrace();
        }
        catch (Exception e){
            e.printStackTrace();
        }
        finally {
            rowLock.unlock(deviceId);
        }
    }
}

```

- 注册设备
```
		/**
		* 构建盒子对象实例
		* @return  盒子
		*/
	 public Device buildBoxDevice(){
			 return new Device("box001","售货机")
							 .deviceType(new Box())
							 .bindHandler(BootStartup.class,device -> log.info("bootStartUp:{}",device))
							 .bindHandler(BootStartupSuccess.class,device -> log.info("bootStartUp Success:{}",device))
							 .bindHandler(Open.class,device -> log.info("open:{}",device))
							 .bindHandler(OpenSuccess.class,device -> log.info("open Success:{}",device))
							 .bindHandler(Close.class,device -> log.info("close:{}",device))
							 .bindHandler(CloseSuccess.class,device -> log.info("close Success:{}",device))
							 .deviceStatus(new Init()).deviceStatus(new Nothing()).deviceStatus(new Idle())
							 .deviceStatus(new Error()).deviceStatus(new Busy())
							 .initStatus(Nothing.class)
							 .bindStatusTrans(Nothing.class,BootStartup.class,Nothing.class)
							 .bindStatusTrans(Nothing.class,BootStartupSuccess.class,Idle.class)
							 .bindStatusTrans(Idle.class,Open.class,Idle.class)
							 .bindStatusTrans(Idle.class,OpenSuccess.class,Busy.class)
							 .bindStatusTrans(Busy.class,Close.class,Busy.class)
							 .bindStatusTrans(Busy.class,CloseSuccess.class,Idle.class)
			 ;
	 }
```

- 模拟IOT事件调用
```
	@Test
    public void buildContextAndRunDevice() throws  Exception{
        DeviceContext context = new DeviceContext() ;
        DeviceHelper helper = new DeviceHelper() ;
        Device device = helper.buildBoxDevice() ;
        context.onRegister(device);

        BootStartup bootStartup = new BootStartup(device) ;
        context.onEvent(bootStartup);

        BootStartupSuccess bootStartupSuccess = new BootStartupSuccess(device) ;
        context.onEvent(bootStartupSuccess);

        Open open = new Open(device) ;
        context.onEvent(open);

        OpenSuccess openSuccess = new OpenSuccess(device) ;
        context.onEvent(openSuccess);

        Close close = new Close(device) ;
        context.onEvent(close);

        CloseSuccess closeSuccess = new CloseSuccess(device) ;
        context.onEvent(closeSuccess);

        //由于异步的关系.有可能上诉两条未完成，马上执onRemove导致异常
        //解决的方式.
        // 可在Device实例上加锁，
        // 可让TestContext接收Device的事件.事件流是  Device->DeviceContext->TestContext.
        Thread.sleep(1000L);
        assert device.getDeviceStatus().getDeviceStatus().equals(DeviceStatus.DeviceStatusEnumClass.Idle) ;
        context.onRemove(device);
    }
```
