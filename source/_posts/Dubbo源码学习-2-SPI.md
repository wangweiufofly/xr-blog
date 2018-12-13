---
title: dubbo源码学习-2-SPI
date: 2018-12-06 10:33:47
tags:  
    - Dubbo
---

## 分布式与集群

- 分布式：一个业务分拆多个子业务，部署在不同的服务器上
- 集群：同一个业务，部署在多个服务器上


## Dubbo 服务治理


<!-- more -->

## 一、Dubbo getAdaptiveExtension() 方法
- 2种情况
####  @Adaptive在扩展类上

``` java
@Adaptive
public class AddExtAdaptive implements AddExt {
    public String echo(URL url, String s) {
        AddExt addExt = ExtensionLoader.getExtensionLoader(AddExt.class).getExtension(url.getParameter("AddExt"));
        return addExt.echo(url, s);
    }
}
```
<!-- more -->
####  @Adaptive在标注在SPI接口的方法上
ExtensionLoader 会生成动态代理类：
 模板如下：
``` java
package <扩展点接口所在包>;
 
public class <扩展点接口名>$Adpative implements <扩展点接口> {
    public <有@Adaptive注解的接口方法>(<方法参数>) {
        if(是否有URL类型方法参数?) 使用该URL参数
        else if(是否有方法类型上有URL属性) 使用该URL属性
        # <else 在加载扩展点生成自适应扩展点类时抛异常，即加载扩展点失败！>
         
        if(获取的URL == null) {
            throw new IllegalArgumentException("url == null");
        }
 
              根据@Adaptive注解上声明的Key的顺序，从URL获致Value，作为实际扩展点名。
               如URL没有Value，则使用缺省扩展点实现。如没有扩展点， throw new IllegalStateException("Fail to get extension");
 
               在扩展点实现调用该方法，并返回结果。
    }
 
    public <有@Adaptive注解的接口方法>(<方法参数>) {
        throw new UnsupportedOperationException("is not adaptive method!");
    }
}
```
以Protocol生成的代理对象Protocol$Adpative为例子：

``` java
package com.alibaba.dubbo.rpc;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }
    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }
    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }
    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}
```
创建自适应扩展点实现类型和实例化就已经完成。


#### Adaptive机制总结
Adaptive机制是一个很好的设计，使用了代理模式，很好的解决多方案实现的适配问题。

## 二、Dubbo getExtension(String name) 方法
获取name对应的扩展类实例
``` java
Set<Class<?>> wrapperClasses = cachedWrapperClasses;
if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
    for (Class<?> wrapperClass : wrapperClasses) {
        instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
    }
}
```
- 使用了装饰者模式，为后面Filter，Listener扩展做了准备
#### Wrapper总结
装饰者模式可以帮我们实现很多的扩展。

## 三、Dubbo getActivateExtension(URL url, String[] values, String group) 方法
getActivateExtension方法主要获取当前扩展的所有可自动激活的实现。可根据入参(values)调整指定实现的顺序，在这个方法里面也使用到getExtensionClasses方法中收集的缓存数据。
``` java
 if (!names.contains(Constants.REMOVE_VALUE_PREFIX + Constants.DEFAULT_KEY)) {
            getExtensionClasses();
            for (Map.Entry<String, Object> entry : cachedActivates.entrySet()) {
                String name = entry.getKey();
                Object activate = entry.getValue();

                String[] activateGroup, activateValue;

                if (activate instanceof Activate) {
                    activateGroup = ((Activate) activate).group();
                    activateValue = ((Activate) activate).value();
                } else if (activate instanceof com.alibaba.dubbo.common.extension.Activate) {
                    activateGroup = ((com.alibaba.dubbo.common.extension.Activate) activate).group();
                    activateValue = ((com.alibaba.dubbo.common.extension.Activate) activate).value();
                } else {
                    continue;
                }
                if (isMatchGroup(group, activateGroup)) {
                    T ext = getExtension(name);
                    if (!names.contains(name)
                            && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)
                            && isActive(activateValue, url)) {
                        exts.add(ext);
                    }
                }
            }
            Collections.sort(exts, ActivateComparator.COMPARATOR);
        }
```
- 使用了装饰者模式，为后面Filter，Listener扩展做了准备
#### Wrapper总结
装饰者模式可以帮我们实现很多的扩展。

## 四、Dubbo的SPI总结和注意点

- 1、每个ExtensionLoader实例只负责加载一个特定扩展点实现
- 2、每个扩展点对应最多只有一个ExtensionLoader实例
- 3、对于每个扩展点实现，最多只会有一个实例
- 4、一个扩展点实现可以对应多个名称(逗号分隔)
- 5、对于需要等到运行时才能决定使用哪一个具体实现的扩展点，应获取其自使用扩展点实现(AdaptiveExtension)
- 6、@Adaptive注解要么注释在扩展点@SPI的方法上，要么注释在其实现类的类定义上
- 7、如果@Adaptive注解注释在@SPI接口的方法上，那么原则上该接口所- - 有方法都应该加@Adaptive注解(自动生成的实现中默认为注解的方法抛异常)
- 8、每个扩展点最多只能有一个被AdaptiveExtension
- 9、每个扩展点可以有多个可自动激活的扩展点实现(使用@Activate注解)
- 10、由于每个扩展点实现最多只有一个实例，因此扩展点实现应保证线程安全
- 11、如果扩展点有多个Wrapper，那么最终其执行的顺序不确定(内部使用ConcurrentHashSet存储)

我们在平时的代码架构中，可以尝试学习Dubbo，多想一下更加有扩展性的设计。
