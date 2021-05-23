# qiankun多实例模式的实现

前面的文章大多都是以单实例模式为例来分析的，`qiankun2.0`之后支持了多实例模式，使得`qiankun`的覆盖场景更多也更加灵活。

下面笔者会结合`qiankun`的源码来分析一下多实例模式的实现以及和单实例模式的差别。



## 1. 多实例模式的实现

多实例模式加载子应用的方法是`loadMicroApp`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/49079fd796ca7f31a2196a5f805dd6d0ffdc1584/src/apis.ts#L56)

```typescript
export function loadMicroApp<T extends ObjectType>(
  app: LoadableApp<T>,
  configuration?: FrameworkConfiguration,
  lifeCycles?: FrameworkLifeCycles<T>,
): MicroApp {
  const { props, name } = app;

  const getContainerXpath = (container: string | HTMLElement): string | void => {
    const containerElement = getContainer(container);
    if (containerElement) {
      return getXPathForElement(containerElement, document);
    }

    return undefined;
  };

  const wrapParcelConfigForRemount = (config: ParcelConfigObject): ParcelConfigObject => {
    return {
      ...config,
      // 防止子应用的bootstrap周期调用多次
      bootstrap: () => Promise.resolve(),
    };
  };

  const memorizedLoadingFn = async (): Promise<ParcelConfigObject> => {
    const userConfiguration = configuration ?? { ...frameworkConfiguration, singular: false };
    // $$cacheLifecycleByAppName是一个过期配置项 后面会删除
    const { $$cacheLifecycleByAppName } = userConfiguration;
    const container = 'container' in app ? app.container : undefined;

    if (container) {
			// ...省略 即将过期的逻辑
      const xpath = getContainerXpath(container);
      if (xpath) {
        const parcelConfigGetterPromise = appConfigPromiseGetterMap.get(`${name}-${xpath}`);
        if (parcelConfigGetterPromise) return wrapParcelConfigForRemount((await parcelConfigGetterPromise)(container));
      }
    }

    const parcelConfigObjectGetterPromise = loadApp(app, userConfiguration, lifeCycles);

    if (container) {
      if ($$cacheLifecycleByAppName) {
        appConfigPromiseGetterMap.set(name, parcelConfigObjectGetterPromise);
      } else {
        const xpath = getContainerXpath(container);
        // 缓存子应用的配置上下文，不需要每次都重新生成，导致内存状态丢失
        if (xpath) appConfigPromiseGetterMap.set(`${name}-${xpath}`, parcelConfigObjectGetterPromise);
      }
    }

    return (await parcelConfigObjectGetterPromise)(container);
  };

  // 这里并没有用提供给single-spa的挂载节点，而且由qiankun来提供挂载节点
  return mountRootParcel(memorizedLoadingFn, { domElement: document.createElement('div'), ...props });
}
```

上面代码可以分成两个阶段来分析

1. 第一次注册：子应用第一次注册
2. 第多次注册：子应用第多次注册



### 1.1 第一次注册

上面代码中可以看到，和单实例模式一样，多实例模式中，子应用生命周期方法的生成都是调用`loadApp`方法来完成。

也就是说**多实例和单实例中子应用资源加载的方式和运行环境都是一样的，都是使用`html-enrty`的方式加载资源，每个子应用都分配了独立的JS沙箱和CSS沙箱**

> 这里有一点需要提一下，在多实例模式下，建议不要使用天然的CSS沙箱，极大可能会出现样式冲突的问题。



当子应用的配置生成完成之后，依然是调用`single-spa`的`mountRootParcel`来注册子应用。

> `mountRootParcel`的参数可以看[这里](https://zh-hans.single-spa.js.org/docs/parcels-api)
>
> 这个文档中`mountRootParcel`的返回值和`loadMicroApp`的返回值有一些差异，我们以`loadMicroApp`为主。
>
> （感觉像是文档没有更新）

这里有一个细节需要注意一下，在调用`mountRootParcel`时，传入的`domElement`并不是子应用真正挂载的节点。这里`qiankun`是为了将子应用挂载的控制权掌握在自己手里，并没有使用`single-spa`内部的挂载逻辑，避免产生冲突。所以传给`mountRootParcel`方法的`domElement`是一个临时生成的`Element`对象，甚至还没有插到DOM树。

> 笔者并没有分析过`single-spa`的源码，上面是笔者根据逻辑猜测的结果。以后有时间可以去验证一下



这里有必要来分析一下`loadMicroApp`的返回值，是一个`Parcel`结构体

1. mount(): Promise<null>; 子应用的挂载方法。子应用第一次加载的时候`single-spa`内部会调用`mount`方法来挂载子应用
2. unmount(): Promise<null>; 子应用的卸载方法。多实例模式中需要手动卸载子应用
3. update(customProps: object): Promise<any>; 子应用的更新方法。多实例模式中子应用才会有这个生命周期
4. getStatus()： 获取当前子应用的状态。枚举值可以看[这里](https://zh-hans.single-spa.js.org/docs/parcels-api#getstatus)
5. loadPromise: Promise<null>; load生命周期的钩子`Promise`，可以用来添加回调函数
6. bootstrapPromise: Promise<null>; bootstrap生命周期的钩子`Promise`
7. mountPromise: Promise<null>; mount生命周期的钩子`Promise`
8. unmountPromise: Promise<null>; unmount生命周期的钩子`Promise`

这里可以看到，**在多实例模式中，子应用的渲染和监控都是交给主应用来控制的**。这种方式更加灵活，可以应对多实例这种复杂的情况，但是需要开发者做的事情也会增多。



### 1.2 第多次注册

在第一次注册的时候，会将生成好的子应用的配置对象缓存起来。

```typescript
const xpath = getContainerXpath(container);
// 缓存子应用的配置上下文，不需要每次都重新生成，导致内存状态丢失
if (xpath) appConfigPromiseGetterMap.set(`${name}-${xpath}`, parcelConfigObjectGetterPromise);
```

这里每个子应用缓存的`key`中包含了挂载节点`contianer`的`XPath`。**`XPath`是一种用来定位DOM元素位置的标识，每个DOM元素对应唯一的`XPath`。**

> XPath的介绍可以看[这里](https://developer.mozilla.org/zh-CN/docs/Web/XPath)



当子应用第多次注册时，会先从缓存列表中读取换缓存，如果没有命中缓存然后才重新生成子应用的配置对象。

在`single-spa`中，是没有去缓存子应用的配置对象的，所以**子应用每次注册都相当于第一次注册，需要重新初始化所有的资源**。

`qiankun`中增加了**应用缓存**的功能，在使用`single-spa`时，这一步通常由开发者自己来完成。

```typescript
const wrapParcelConfigForRemount = (config: ParcelConfigObject): ParcelConfigObject => {
    return {
        ...config,
        // 防止子应用的bootstrap周期调用多次
        bootstrap: () => Promise.resolve(),
    };
};
```

增加缓存的好处有以下两点：

1. **注册过的子应用对应的静态资源不需要重新请求，极大提高了子应用加载的速度。**
2. **缓存中记录了子应用卸载时的执行环境，当子应用第二次注册的时候能够恢复到卸载前的执行环境。**（闭包中保存）

> 这里说的注册指的就是调用`loadMicroApp`来加载应用，而不是直接调用子应用的`mount`方法



上面代码中有一个细节需要注意一下：对于第二次注册的子应用，这里替换掉了生成的`bootstrap`方法。

这么做的意义是**`qiankun`中将第二次注册的子应用看成是子应用的重新挂载，相当于是调用mount方法重新挂载子应用，不应该再去执行子应用提供的`bootstrap`生命周期方法。**

实际上`single-spa`在每次注册都会去执行，所以这里使用一个简单函数替换掉了子应用的`bootstrap`方法。



## 2. 单实例模式 VS 多实例模式

下面来对比一下单实例模式和多实例模式中子应用的异同点。

1. **资源的加载方式相同**

   都是通过`html-entry`的方式来加载资源

   

2. **子应用运行时的执行环境相同**

   都使用JS沙箱和CSS沙箱，通过闭包的形式来记录当前子应用的执行环境上下文。



3. **子应用的控制方不同**（也可以理解为开发的成本不同）
   - 单实例模式中，子应用的渲染周期由框架来控制，开发者只需要提供对应的生命周期钩子函数即可，开发成本低。
   - 多实例模式中，子应用的渲染周期需要由主应用开发者来控制，而且需要手动来触发子应用的生命周期，开发成本较高。



所以，在实际的场景中，推荐先考虑单实例模式来实现，开发成本很低，接入也很快；但是如果需要支持多实例的场景，开发时需要注意控制子应用渲染周期的逻辑，这块比较容易出错。

