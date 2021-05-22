# qiankun中JS沙箱的实现

在上一篇文章中，我们分析了`qiankun`中CSS沙箱的实现，CSS沙箱主要是用来隔绝各应用的样式，防止样式冲突。

同样对于各应用的`js`文件来说，也需要一个独立的环境来执行，**防止全局对象发生属性读写冲突**，这个独立的执行环境叫做`JS沙箱`。

前端应用的`js`代码运行在浏览器中，所以这个全局对象指的是`window`对象。

js沙箱的实现思路是：**为每个子应用创建一个全局对象的替代对象，然后代理子应用中对于全局对象的操作，使其操作在替代对象中，不影响原来的全局对象。**

根据上面的思路，我们首先想到的是使用`proxy`来实现`js`沙箱，但是`proxy`还是有一些低版本浏览器的不支持。所以`qiankun`内部实现了两种`js`沙箱用来兼容一些老旧浏览器

1. snapshotSandbox: 快照沙箱
2. proxySandbox: 代理沙箱



## 1. snapshotSandbox — 快照沙箱

快照沙箱是对于那些不支持`proxy`的浏览器的一个向下兼容的方案。

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/58253a35b30e5534dc1b62ef5181901cc814ee17/src/sandbox/snapshotSandbox.ts#L20)

```typescript
function iter(obj: typeof window, callbackFn: (prop: any) => void) {
  for (const prop in obj) {
    if (obj.hasOwnProperty(prop)) {
      callbackFn(prop);
    }
  }
}

export default class SnapshotSandbox implements SandBox {
  constructor(name: string) {
    this.name = name;
    this.proxy = window;
    this.type = SandBoxType.Snapshot;
  }

  active() {
    // 记录当前快照
    this.windowSnapshot = {} as Window;
    iter(window, (prop) => {
      this.windowSnapshot[prop] = window[prop];
    });

    // 恢复之前的变更
    Object.keys(this.modifyPropsMap).forEach((p: any) => {
      window[p] = this.modifyPropsMap[p];
    });

    this.sandboxRunning = true;
  }

  inactive() {
    this.modifyPropsMap = {};

    iter(window, (prop) => {
      if (window[prop] !== this.windowSnapshot[prop]) {
        // 记录变更，恢复环境
        this.modifyPropsMap[prop] = window[prop];
        window[prop] = this.windowSnapshot[prop];
      }
    });

    if (process.env.NODE_ENV === 'development') {
      console.info(`[qiankun:sandbox] ${this.name} origin window restore...`, Object.keys(this.modifyPropsMap));
    }

    this.sandboxRunning = false;
  }
}
```

可以看出，整体的实现代码非常简单。

- 在沙箱激活之前，使用一个对象来保存当前`window`对象的快照。同时将**已经修改的属性**更新到`window`对象中，然后子应用**直接在`window`对象上操作**。
- 在沙箱失活的时候，对应当前`window`对象和之前保存的`window`快照对象，收集被修改的属性，然后使用快照还原`window`对象。

由此刚好形成一个闭环操作。通过在沙箱激活和失活的时候更新`window`对象来达到不影响子应用的效果（仅在单实例模式下）。



### 1.1 快照沙箱的缺点

从上面的快照沙箱的实现逻辑中可以看出，快照沙箱有以下缺点：

1. **只能支持单实例模式**：因为快照沙箱中子应用是直接在window对象上操作的，所以如果是多实例模式中，两个子应用都读写window对象中的同一属性，必然会出现相互覆盖的情况。
2. **沙箱切换的效率不高**：每次切换都需要遍历`window`中所有的属性，是一个不小的开销。



## 2. proxySandbox — 代理沙箱

代理沙箱的实现原理：**为每个实例创建一个全局对象（window）的替代对象（fakeWindow），并且使用`proxy`来代理对全局对象的操作，使其操作在替代对象中**。

代理沙箱对应的实现文件是`/src/sandbox/proxySandbox.ts`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/58253a35b30e5534dc1b62ef5181901cc814ee17/src/sandbox/proxySandbox.ts#L129)

```typescript
export default class ProxySandbox implements SandBox {
  // 记录fakeWindow中更新的属性
  private updatedValueSet = new Set<PropertyKey>();
	// 沙箱的名字
  name: string;
	// 沙箱类型
  type: SandBoxType;
	// fakeWindow的代理对象
  proxy: WindowProxy;
	// 当前沙箱是否在运行
  sandboxRunning = true;
	// 最后一个修改的属性，用来获取子应用的生命周期函数
  latestSetProp: PropertyKey | null = null;

  active() {
    if (!this.sandboxRunning) activeSandboxCount++;
    this.sandboxRunning = true;
  }

  inactive() {
    if (--activeSandboxCount === 0) {
      variableWhiteList.forEach((p) => {
        if (this.proxy.hasOwnProperty(p)) {
          // @ts-ignore
          delete window[p];
        }
      });
    }

    this.sandboxRunning = false;
  }

  constructor(name: string) {
    this.name = name;
    this.type = SandBoxType.Proxy;
    const { updatedValueSet } = this;

    const rawWindow = window;
    // 创建fakeWindow
    const { fakeWindow, propertiesWithGetter } = createFakeWindow(rawWindow);
    // 记录属性描述对象的来源，也就是属性的来源（window or fakeWindow）
    const descriptorTargetMap = new Map<PropertyKey, SymbolTarget>();
    const hasOwnProperty = (key: PropertyKey) => fakeWindow.hasOwnProperty(key) || rawWindow.hasOwnProperty(key);

    const proxy = new Proxy(fakeWindow, {
      // ...省略
    });

    this.proxy = proxy;

    activeSandboxCount++;
  }
}
```

从上面代码中可以将代理沙箱的实现分成三个部分

1. **创建全局对象的替代对象**
2. **使用`proxy`来代理全局对象的操作**
3. **沙箱的生命周期**

> 属性解释：
>
> 1. variableWhiteList：白名单的属性值修改需要反馈到`window`对象中，其他js文件可能需要访问
> 2. activeSandboxCount：激活沙箱的个数



### 2.1 创建替代对象 — fakeWindow

对应的方法是`createFakeWindow`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/58253a35b30e5534dc1b62ef5181901cc814ee17/src/sandbox/proxySandbox.ts#L64)

```typescript
function createFakeWindow(global: Window) {
  const propertiesWithGetter = new Map<PropertyKey, boolean>();
  const fakeWindow = {} as FakeWindow;

  /*
   window中无法配置的属性copy一份存在fakeWindow对象中
   see https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy/handler/getOwnPropertyDescriptor
   > A property cannot be reported as non-configurable, if it does not exists as an own property of the target object or if it exists as a configurable own property of the target object.
   */
  Object.getOwnPropertyNames(global)
    .filter((p) => {
      const descriptor = Object.getOwnPropertyDescriptor(global, p);
      return !descriptor?.configurable;
    })
    .forEach((p) => {
      const descriptor = Object.getOwnPropertyDescriptor(global, p);
      if (descriptor) {
        const hasGetter = Object.prototype.hasOwnProperty.call(descriptor, 'get');

        // ...省略 一些代理规则处理
        
        if (hasGetter) propertiesWithGetter.set(p, true);

        // 将属性描述对象冻结，防止被zone.js修改
        rawObjectDefineProperty(fakeWindow, p, Object.freeze(descriptor));
      }
    });

  return {
    fakeWindow,
    propertiesWithGetter,
  };
}
```

上面方法主要做了两个事情：

1. 创建`fakeWindow`对象

2. 将`window`对象中**不可配置**的属性复制到`fakeWindow`中，主要是为了遵守`Object.getOwnPropertyDescriptor`的代理规则。

   > 详细的规则可以看[这里](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy/Proxy/getOwnPropertyDescriptor)

`propertiesWithGetter`用来记录这些不可配置属性中哪些拥有`getter`，也就是可以被访问。

> 至于fakeWindow是如何在运行时来替换window，简单来说就是通过**函数作用域**和**this绑定**来实现的。
>
> 具体的执行可以看`import-html-entry`中的实现，可以看[这里](https://github.com/kuitos/import-html-entry/blob/3765762fe7fc3d960b981c8a2dc6181e37a23ee6/src/index.js#L52)



### 2.2 使用proxy代理全局对象的操作

整个代理对象中实现了多个拦截器，这里我们主要来分析一下`set`和`get`拦截器，其他的拦截器代码比较简单



#### 2.2.1 set

```typescript
set: (target: FakeWindow, p: PropertyKey, value: any): boolean => {
    if (this.sandboxRunning) {
      	// 沙箱激活时才能修改全局属性
        if (!target.hasOwnProperty(p) && rawWindow.hasOwnProperty(p)) {
          	// window中独有的属性
            const descriptor = Object.getOwnPropertyDescriptor(rawWindow, p);
            const {
                writable,
                configurable,
                enumerable
            } = descriptor!;
            if (writable) {
                Object.defineProperty(target, p, {
                    configurable,
                    enumerable,
                    writable,
                    value,
                });
            }
        } else {
            // @ts-ignore
            target[p] = value;
        }

        if (variableWhiteList.indexOf(p) !== -1) {
            // @ts-ignore
            rawWindow[p] = value; // 白名单中的属性需要存在在真实的window对象中
        }

        updatedValueSet.add(p);

        this.latestSetProp = p;

        return true;
    }

    // 在 strict-mode 下，Proxy 的 handler.set 返回 false 会抛出 TypeError，在沙箱卸载的情况下应该忽略错误
    return true;
},
```

上面代码可以看出，**除了白名单中的属性之后，全局对象中所有属性的修改都会保存在`fakeWindow`中，并不会污染`window`对象**。

> 白名单中的属性如果发生修改同样会修改`window`中对应的属性，但是在**所有的**沙箱都失活的时候才会删除`window`中这些属性



#### 2.2.2 get

```typescript
get(target: FakeWindow, p: PropertyKey): any {
    // ... 省略一些特殊属性的处理
    
  	// 动态创建标签或者执行脚本时
    if (p === 'document' || p === 'eval') {
        setCurrentRunningSandboxProxy(proxy);
        nextTick(() => setCurrentRunningSandboxProxy(null)); // 临时方案
        switch (p) {
            case 'document':
                return document;
            case 'eval':
                // eslint-disable-next-line no-eval
                return eval;
                // no default
        }
    }
		
  	// propertiesWithGetter中的属性从window或者fakeWindow中获取都可以
    const value = propertiesWithGetter.has(p) ?
        (rawWindow as any)[p] :
        p in target ?
        (target as any)[p] :
        (rawWindow as any)[p];
  	// 对于Function属性的特殊处理，绑定this为window
    return getTargetValue(rawWindow, value);
},
```

`get`拦截器中有很多对于特殊属性的`hack`代码，这里就不展开分析了。我们重点看一下上面贴出来的代码。

当访问`document`属性时，会调用`setCurrentRunningSandboxProxy`方法来**设置当前操作是在哪个沙箱实例中**进行的，这一步对后面劫持`document.createElement`方法起到关键的作用。

子应用动态创建某些标签的时候需要插入额外逻辑，执行完成之后需要还原执行上下文（后面会详细分析），防止影响主应用的使用。这里创建一个微任务来还原执行上下文。

```typescript
export function nextTick(cb: () => void): void {
  Promise.resolve().then(cb);
}
```

> **注意：**这里还原的方案是一个临时方案，某些场景无法覆盖。
>
> 考虑一种情况：当子应用和主应用同步创建`script`标签的时候，主应用创建的`script`标签对应的代码会运行在沙箱环境中。这个是不合适的。（qiankun团队暂时还没有解决这个问题）



### 2.3 沙箱的生命周期：active 和 inactive

`ProxySandbox`类中构造代理对象之外，还提供了两个方法用来描述沙箱实例的生命周期。

1. active - 沙箱激活。

   ```typescript
   active() {
       if (!this.sandboxRunning) activeSandboxCount++;
       this.sandboxRunning = true;
   }
   ```

   **沙箱激活之后才能在修改全局对象的属性。在失活状态，修改全局属性会失败，但是可以读取属性**。（看前面的set和get方法）

2. Inactive - 沙箱失活。

   ```typescript
   inactive() {
       if (--activeSandboxCount === 0) {
           variableWhiteList.forEach((p) => {
               if (this.proxy.hasOwnProperty(p)) {
                   // @ts-ignore
                   delete window[p];
               }
           });
       }
   
       this.sandboxRunning = false;
   }
   ```

   如果所有的沙箱都失活，会删除window对象中存在在白名单中的属性



## 3. qiankun中沙箱的使用

`qiankun`中会为每一个子应用创建一个`js`沙箱，对应的入口方法是`createSandboxContainer`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/2b42f2156e3865215e8b912e449acbf0fc4eff10/src/sandbox/index.ts#L32)

```typescript
export function createSandboxContainer(
  appName: string,
  elementGetter: () => HTMLElement | ShadowRoot,
  scopedCSS: boolean,
  useLooseSandbox?: boolean,
  excludeAssetFilter?: (url: string) => boolean,
) {
  let sandbox: SandBox;
  if (window.Proxy) {
    sandbox = useLooseSandbox ? new LegacySandbox(appName) : new ProxySandbox(appName);
  } else {
    sandbox = new SnapshotSandbox(appName);
  }

  // some side effect could be be invoked while bootstrapping, such as dynamic stylesheet injection with style-loader, especially during the development phase
  const bootstrappingFreers = patchAtBootstrapping(appName, elementGetter, sandbox, scopedCSS, excludeAssetFilter);
  // mounting freers are one-off and should be re-init at every mounting time
  let mountingFreers: Freer[] = [];

  let sideEffectsRebuilders: Rebuilder[] = [];

  return {
    instance: sandbox,

    /**
     * 沙箱被 mount
     * 可能是从 bootstrap 状态进入的 mount
     * 也可能是从 unmount 之后再次唤醒进入 mount
     */
    async mount() {
      /* ------------------------------------------ 因为有上下文依赖（window），以下代码执行顺序不能变 ------------------------------------------ */

      /* ------------------------------------------ 1. 启动/恢复 沙箱------------------------------------------ */
      sandbox.active();

      const sideEffectsRebuildersAtBootstrapping = sideEffectsRebuilders.slice(0, bootstrappingFreers.length);
      const sideEffectsRebuildersAtMounting = sideEffectsRebuilders.slice(bootstrappingFreers.length);

      // must rebuild the side effects which added at bootstrapping firstly to recovery to nature state
      if (sideEffectsRebuildersAtBootstrapping.length) {
        sideEffectsRebuildersAtBootstrapping.forEach((rebuild) => rebuild());
      }

      /* ------------------------------------------ 2. 开启全局变量补丁 ------------------------------------------*/
      // render 沙箱启动时开始劫持各类全局监听，尽量不要在应用初始化阶段有 事件监听/定时器 等副作用
      // mount阶段也会执行沙箱的补丁
      mountingFreers = patchAtMounting(appName, elementGetter, sandbox, scopedCSS, excludeAssetFilter);

      /* ------------------------------------------ 3. 重置一些初始化时的副作用 ------------------------------------------*/
      // 存在 rebuilder 则表明有些副作用需要重建
      if (sideEffectsRebuildersAtMounting.length) {
        sideEffectsRebuildersAtMounting.forEach((rebuild) => rebuild());
      }

      // clean up rebuilders
      sideEffectsRebuilders = [];
    },

    /**
     * 恢复 global 状态，使其能回到应用加载之前的状态
     */
    async unmount() {
      // 卸载补丁，
      sideEffectsRebuilders = [...bootstrappingFreers, ...mountingFreers].map((free) => free());

      sandbox.inactive();
    },
  };
}
```

上面代码中可以看出，子应用的js沙箱可以分成两个部分：

1. **创建js沙箱**
2. **补丁系统**



### 3.1 创建JS沙箱

创建js沙箱的代码比较简单

```typescript
let sandbox: SandBox;
if (window.Proxy) {
    sandbox = useLooseSandbox ? new LegacySandbox(appName) : new ProxySandbox(appName);
} else {
    sandbox = new SnapshotSandbox(appName);
}
```

根据当前的运行环境选择实例化不同类型的沙箱，这里除了前面介绍的两种沙箱`SnapshotSandbox`和`ProxySandbox`，还有一种沙箱叫做`LegacySandbox`。这种沙箱是一个向下兼容的`proxySandbox`，慢慢会被抛弃，所以就不展开分析了。

这里有一个点需要注意：

**子应用的js沙箱是在`bootstrap`阶段创建，然后持久化的存在在子应用的各个生命周期中，子应用卸载的时候也只是使沙箱失活，并不会注销这个沙箱。**

> `qiankun`中并没有实现`single-spa`中注销子应用的接口，后续如果实现的话应该需要同步注销对应的js沙箱



### 3.2 补丁系统

在创建好js沙箱之后，在子应用`mount`之前，`qiankun`给沙箱实例添加了一些补丁，用来解决不同的问题，当子应用卸载的时候，这些补丁也会卸载。

`qiankun`暂时添加了4种补丁：

1. dynamicAppend：用来解决子应用动态添加`link、style或script`标签的情况
2. interval：用来解决子应用卸载时内部`setInterval`可能没有卸载的问题
3. windowListener: 用来解决子应用卸载时内部绑定在window上的事件可能没有卸载的问题
4. historyListener: 用来解决和`umi`一起使用时的问题（这个不展开分析）



#### 3.2.1 dynamicAppend 补丁

子应用中动态添加的样式或者脚本资源默认也需要运行在对应的`CSS沙箱`和`JS沙箱`中，`dynamicAppend`就是用来解决这个问题。

那么它是如何来解决的呢？我们看一下具体的实现，该补丁的入口方法是`patchStrictSandbox`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/2b42f2156e3865215e8b912e449acbf0fc4eff10/src/sandbox/patchers/dynamicAppend/forStrictSandbox.ts#L53)

```typescript
let bootstrappingPatchCount = 0; // bootstrap阶段添加该补丁的次数
let mountingPatchCount = 0; // mount阶段添加该补丁的次数

export function patchStrictSandbox(
  appName: string,
  appWrapperGetter: () => HTMLElement | ShadowRoot,
  proxy: Window,
  mounting = true,
  scopedCSS = false,
  excludeAssetFilter?: CallableFunction,
): Freer {
  let containerConfig = proxyAttachContainerConfigMap.get(proxy);
  if (!containerConfig) {
    // 子应用的基本配置信息
    containerConfig = {
      appName,
      proxy,
      appWrapperGetter,
      dynamicStyleSheetElements: [],
      strictGlobal: true,
      excludeAssetFilter,
      scopedCSS,
    };
    proxyAttachContainerConfigMap.set(proxy, containerConfig);
  }
  // 动态添加的style rules ，都会被缓存起来
  const { dynamicStyleSheetElements } = containerConfig;

  // 代理document.create方法
  const unpatchDocumentCreate = patchDocumentCreateElement();

  // 代理动态插入DOM元素的方法
  const unpatchDynamicAppendPrototypeFunctions = patchHTMLDynamicAppendPrototypeFunctions(
    (element) => elementAttachContainerConfigMap.has(element),
    (element) => elementAttachContainerConfigMap.get(element)!,
  );

  if (!mounting) bootstrappingPatchCount++;
  if (mounting) mountingPatchCount++;

  // 补丁卸载方法
  return function free() {
    // bootstrap patch just called once but its freer will be called multiple times
    if (!mounting && bootstrappingPatchCount !== 0) bootstrappingPatchCount--;
    if (mounting) mountingPatchCount--;

    const allMicroAppUnmounted = mountingPatchCount === 0 && bootstrappingPatchCount === 0;
    // 必须所有子应用都销毁时才能卸载补丁
    if (allMicroAppUnmounted) {
      unpatchDynamicAppendPrototypeFunctions();
      unpatchDocumentCreate();
    }

    // 缓存动态添加的样式元素
    recordStyledComponentsCSSRules(dynamicStyleSheetElements);

    // 重建补丁产生的副作用，也就是缓存的样式元素
    return function rebuild() {
      rebuildCSSRules(dynamicStyleSheetElements, (stylesheetElement) => {
        const appWrapper = appWrapperGetter();
        if (!appWrapper.contains(stylesheetElement)) {
          rawHeadAppendChild.call(appWrapper, stylesheetElement);
          return true;
        }
        return false;
      });
    };
  };
}
```

上面代码可以看出，这个补丁主要做了以下三个操作来实现对动态添加资源的劫持

1. 劫持`document.createElement`方法
2. 劫持`appendChild、insertBefore和removeChild`方法
3. 缓存和恢复 动态添加`style`元素和对应的`CSSRules`



##### 3.2.1.1 劫持document.createElement方法

对应的方法是`patchDocumentCreateElement`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/2b42f2156e3865215e8b912e449acbf0fc4eff10/src/sandbox/patchers/dynamicAppend/forStrictSandbox.ts#L21)

```typescript
function patchDocumentCreateElement() {
  if (Document.prototype.createElement === rawDocumentCreateElement) {
    Document.prototype.createElement = function createElement<K extends keyof HTMLElementTagNameMap>(
      this: Document,
      tagName: K,
      options?: ElementCreationOptions,
    ): HTMLElement {
      const element = rawDocumentCreateElement.call(this, tagName, options);
      if (isHijackingTag(tagName)) {
        // 如果是需要劫持的标签
        const currentRunningSandboxProxy = getCurrentRunningSandboxProxy();
        if (currentRunningSandboxProxy) {
          // 从proxy-containerConfig map中获取对应子应用的基本配置信息
          const proxyContainerConfig = proxyAttachContainerConfigMap.get(currentRunningSandboxProxy);
          if (proxyContainerConfig) {
            // 如果是在沙箱中调用，在element-containerConfig map中储存起来
            elementAttachContainerConfigMap.set(element, proxyContainerConfig);
          }
        }
      }

      return element;
    };
  }

  // 返回一个函数 用来还原配置，qiankun内部有很多这种类似的方法
  return function unpatch() {
    Document.prototype.createElement = rawDocumentCreateElement;
  };
}
```

代码逻辑比较简单，采用**重写**的形式来劫持`createElement`方法。

针对动态创建的`hijackingTag`元素做的额外操作就是**记录新元素`element`和子应用配置信息`containerConfig`之间的映射关系，**后面插入元素时需要使用。

> **注意：**
>
> 这种重写的劫持方式需要处理好边界，是有可能会影响到其他应用的。这个前面`get`方法那里有讲过。



##### 3.2.1.2 劫持DOM插入和删除的方法

DOM插入和删除对应的方法是`appendChild、insertBefore和removeChild`，对应的劫持方法是`patchHTMLDynamicAppendPrototypeFunctions`

> 对应的源代码可以看[这里](https://github.com/umijs/qiankun/blob/387a50899f6b21ed94be3a19f6c464d5791136a3/src/sandbox/patchers/dynamicAppend/common.ts#L300)

```typescript
export function patchHTMLDynamicAppendPrototypeFunctions(
  isInvokedByMicroApp: (element: HTMLElement) => boolean,
  containerConfigGetter: (element: HTMLElement) => ContainerConfig,
) {
  // Just overwrite it while it have not been overwrite
  if (
    HTMLHeadElement.prototype.appendChild === rawHeadAppendChild &&
    HTMLBodyElement.prototype.appendChild === rawBodyAppendChild &&
    HTMLHeadElement.prototype.insertBefore === rawHeadInsertBefore
  ) {
    HTMLHeadElement.prototype.appendChild = getOverwrittenAppendChildOrInsertBefore({
      rawDOMAppendOrInsertBefore: rawHeadAppendChild,
      containerConfigGetter,
      isInvokedByMicroApp,
    }) as typeof rawHeadAppendChild;
    HTMLBodyElement.prototype.appendChild = getOverwrittenAppendChildOrInsertBefore({
      rawDOMAppendOrInsertBefore: rawBodyAppendChild,
      containerConfigGetter,
      isInvokedByMicroApp,
    }) as typeof rawBodyAppendChild;

    HTMLHeadElement.prototype.insertBefore = getOverwrittenAppendChildOrInsertBefore({
      rawDOMAppendOrInsertBefore: rawHeadInsertBefore as any,
      containerConfigGetter,
      isInvokedByMicroApp,
    }) as typeof rawHeadInsertBefore;
  }

  // Just overwrite it while it have not been overwrite
  if (
    HTMLHeadElement.prototype.removeChild === rawHeadRemoveChild &&
    HTMLBodyElement.prototype.removeChild === rawBodyRemoveChild
  ) {
    HTMLHeadElement.prototype.removeChild = getNewRemoveChild(
      rawHeadRemoveChild,
      (element) => containerConfigGetter(element).appWrapperGetter,
    );
    HTMLBodyElement.prototype.removeChild = getNewRemoveChild(
      rawBodyRemoveChild,
      (element) => containerConfigGetter(element).appWrapperGetter,
    );
  }

  return function unpatch() {
    HTMLHeadElement.prototype.appendChild = rawHeadAppendChild;
    HTMLHeadElement.prototype.removeChild = rawHeadRemoveChild;
    HTMLBodyElement.prototype.appendChild = rawBodyAppendChild;
    HTMLBodyElement.prototype.removeChild = rawBodyRemoveChild;

    HTMLHeadElement.prototype.insertBefore = rawHeadInsertBefore;
  };
}
```

可以看到，这里也是使用重写的方式来劫持原生的方法，所以会污染全局，使用的时候需要处理好边界条件。



这里我们主要来看看劫持插入元素的方法额外做了哪些操作。对应的方法是`getOverwrittenAppendChildOrInsertBefore`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/2b42f2156e3865215e8b912e449acbf0fc4eff10/src/sandbox/patchers/dynamicAppend/common.ts#L149)

```typescript
function getOverwrittenAppendChildOrInsertBefore(opts: {
  rawDOMAppendOrInsertBefore: <T extends Node>(newChild: T, refChild?: Node | null) => T;
  isInvokedByMicroApp: (element: HTMLElement) => boolean;
  containerConfigGetter: (element: HTMLElement) => ContainerConfig;
}) {
  return function appendChildOrInsertBefore<T extends Node>(
    this: HTMLHeadElement | HTMLBodyElement,
    newChild: T,
    refChild?: Node | null,
  ) {
    let element = newChild as any;
    const { rawDOMAppendOrInsertBefore, isInvokedByMicroApp, containerConfigGetter } = opts;
    if (!isHijackingTag(element.tagName) || !isInvokedByMicroApp(element)) {
      // 边界判断不过，走原生逻辑
      return rawDOMAppendOrInsertBefore.call(this, element, refChild) as T;
    }

    if (element.tagName) {
      const containerConfig = containerConfigGetter(element);
      const {
        appName,
        appWrapperGetter,
        proxy,
        strictGlobal,
        dynamicStyleSheetElements,
        scopedCSS,
        excludeAssetFilter,
      } = containerConfig;

      switch (element.tagName) {
        case LINK_TAG_NAME:
        case STYLE_TAG_NAME: {
          // ...省略
        }

        case SCRIPT_TAG_NAME: {
          // ...省略
        }

        default:
          break;
      }
    }

    return rawDOMAppendOrInsertBefore.call(this, element, refChild);
  };
}
```

内部对动态添加的样式资源和脚本资源都做了额外处理。

1. **样式资源**：动态添加的`style`或者`link`标签

   ```typescript
   {
       let stylesheetElement: HTMLLinkElement | HTMLStyleElement = newChild as any;
       const {
           href
       } = stylesheetElement as HTMLLinkElement;
       if (excludeAssetFilter && href && excludeAssetFilter(href)) {
         	// 白名单的资源不加入子应用沙箱中，会添加在head标签内部
           return rawDOMAppendOrInsertBefore.call(this, element, refChild) as T;
       }
   
       const mountDOM = appWrapperGetter();
   
       if (scopedCSS) {
         	// 需要使用作用域CSS沙箱
           const linkElementUsingStylesheet =
               element.tagName ? .toUpperCase() === LINK_TAG_NAME &&
               (element as HTMLLinkElement).rel === 'stylesheet' &&
               (element as HTMLLinkElement).href;
           if (linkElementUsingStylesheet) {
               const fetch =
                   typeof frameworkConfiguration.fetch === 'function' ?
                   frameworkConfiguration.fetch :
                   frameworkConfiguration.fetch ? .fn;
               // 将link解析成style
               stylesheetElement = convertLinkAsStyle(
                   element,
                   (styleElement) => css.process(mountDOM, styleElement, appName),
                   fetch,
               );
               dynamicLinkAttachedInlineStyleMap.set(element, stylesheetElement);
           } else {
               css.process(mountDOM, stylesheetElement, appName);
           }
       }
   
       // 记录动态添加到子应用沙箱中的style和link
       dynamicStyleSheetElements.push(stylesheetElement);
       const referenceNode = mountDOM.contains(refChild) ? refChild : null;
       return rawDOMAppendOrInsertBefore.call(mountDOM, stylesheetElement, referenceNode);
   }
   ```

   主要做了两个操作：

   1. 缓存动态添加到**子应用沙箱环境**的`style和link`元素，为后续子应用再次由`unmount`变成`mount`时恢复沙箱环境做准备。

   2. 如果子应用开启了`作用域CSS沙箱`，需要将动态添加的`link`转化成`style`，然后给里面的选择器添加前缀选择器

      

2. **脚本资源**： 动态添加的`script`元素

   ```typescript
   {
       const {src,text} = element as HTMLScriptElement;
       // some script like jsonp maybe not support cors which should't use execScripts
       if (excludeAssetFilter && src && excludeAssetFilter(src)) {
           return rawDOMAppendOrInsertBefore.call(this, element, refChild) as T;
       }
   
       const mountDOM = appWrapperGetter();
       const {fetch} = frameworkConfiguration;
       const referenceNode = mountDOM.contains(refChild) ? refChild : null;
   
       if (src) {
           execScripts(null, [src], proxy, {
               fetch,
               strictGlobal,
               beforeExec: () => {
                   Object.defineProperty(document, 'currentScript', {
                       get(): any {
                           return element;
                       },
                       configurable: true,
                   });
               },
               success: () => {
                   manualInvokeElementOnLoad(element);
                   element = null;
               },
               error: () => {
                   manualInvokeElementOnError(element);
                   element = null;
               },
           });
   
         	// 创建CommentElement来注释script元素插入
           const dynamicScriptCommentElement = document.createComment(`dynamic script ${src} replaced by qiankun`);
           dynamicScriptAttachedCommentMap.set(element, dynamicScriptCommentElement);
           return rawDOMAppendOrInsertBefore.call(mountDOM, dynamicScriptCommentElement, referenceNode);
       }
   
       // inline script never trigger the onload and onerror event
       execScripts(null, [`<script>${text}</script>`], proxy, {
           strictGlobal
       });
       const dynamicInlineScriptCommentElement = document.createComment('dynamic inline script replaced by qiankun');
       dynamicScriptAttachedCommentMap.set(element, dynamicInlineScriptCommentElement);
       return rawDOMAppendOrInsertBefore.call(mountDOM, dynamicInlineScriptCommentElement, referenceNode);
   }
   ```

   **对于动态添加到子应用沙箱环境的`script`标签，直接由`qiankun`内部请求并在沙箱环境中执行，不插入DOM树中，而且插入一个`CommentElement`来注释这个`script`标签。**

> 注意：
>
> 这里有一个概念需要解释一下，**动态添加到子应用沙箱环境的资源指的是`excludeAssetFilter`白名单之外的资源，这些资源可以子应用的沙箱关联在一起，执行的时候需要使用到CSS沙箱或者JS沙箱。**（CSS沙箱开启的时候）



##### 3.2.1.3 缓存和恢复动态添加`style`元素和对应的`CSSRules`

上面代码中可以看出，当`dynamicAppend`补丁卸载的时候，会缓存子应用沙箱环境中动态添加的`style`元素的`CSSRules`，对应的方法是`recordStyledComponentsCSSRules`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/ef328fe0972ebf2f1c9b7a1bc4898490db3fd085/src/sandbox/patchers/dynamicAppend/common.ts#L116)

```typescript
export function recordStyledComponentsCSSRules(styleElements: HTMLStyleElement[]): void {
  styleElements.forEach((styleElement) => {
    if (styleElement instanceof HTMLStyleElement && isStyledComponentsLike(styleElement)) {
      // 没有textContent，只有sheet的style元素
      if (styleElement.sheet) {
        // record the original css rules of the style element for restore
        styledComponentCSSRulesMap.set(styleElement, (styleElement.sheet as CSSStyleSheet).cssRules);
      }
    }
  });
}
```

然后在**子应用由`unmount`变成`mount`的时候，重新将这些样式资源存在沙箱环境中，将子应用的沙箱环境恢复到`unmount`之前的环境。**

对应的方法是`rebuildCSSRules`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/ef328fe0972ebf2f1c9b7a1bc4898490db3fd085/src/sandbox/patchers/dynamicAppend/common.ts#L365)

```typescript
export function rebuildCSSRules(
  styleSheetElements: HTMLStyleElement[],
  reAppendElement: (stylesheetElement: HTMLStyleElement) => boolean,
) {
  styleSheetElements.forEach((stylesheetElement) => {
    // 将之前缓存的样式元素重新添加在沙箱环境中
    const appendSuccess = reAppendElement(stylesheetElement);
    if (appendSuccess) {
      if (stylesheetElement instanceof HTMLStyleElement && isStyledComponentsLike(stylesheetElement)) {
        const cssRules = getStyledElementCSSRules(stylesheetElement);
        if (cssRules) {
          // 这里不会导致规则重复添加吗？？？
          for (let i = 0; i < cssRules.length; i++) {
            const cssRule = cssRules[i];
            const cssStyleSheetElement = stylesheetElement.sheet as CSSStyleSheet;
            cssStyleSheetElement.insertRule(cssRule.cssText, cssStyleSheetElement.cssRules.length);
          }
        }
      }
    }
  });
}
```

也就是说，**子应用中动态添加到沙箱环境的样式资源被认为是沙箱环境的一部分，当子应用重新激活的时候，需要将沙箱环境恢复到卸载之前的状态。**

所以如果想重新激活之后没有之前动态添加的样式，需要手动清除



> 这里在恢复样式的时候有一个问题？
>
> 子应用卸载的时候`style`元素内部的`CSSRules`也被缓存了一份，而且恢复的时候重新追加到对应的`style`元素中，这里感觉会导致样式规则重复。
>
> **经验证是不会的。**
>
> 
>
> 先看`recordStyledComponentsCSSRules`方法中收集`CSSRule`的逻辑，这里调用`isStyledComponentsLike`方法
>
> ```typescript
> export function isStyledComponentsLike(element: HTMLStyleElement) {
>   return (
>     !element.textContent &&
>     ((element.sheet as CSSStyleSheet)?.cssRules.length || getStyledElementCSSRules(element)?.length)
>   );
> }
> ```
>
> 这里方法用来过滤符合条件的`style`元素
>
> 1. **textContent为空**。也就是style标签内没有内容
> 2. **sheet.cssRules不为空**。也就是说这个`style`标签的样式是直接通过`CSSStyleSheet API`来直接添加的，也就是style标签没有内容时也是有可能包含样式规则的。样式规则是存在`sheet.cssRules`中的，`styled-component`就是通过这种方式添加样式的
>
> 
>
> 这里有一个细节需要注意一下
>
> **`style`元素的`sheet`属性是由浏览器解析生成的，也就是说只有当style元素插入到DOM树中，才会生成样式表并关联在sheet属性中；如果style元素从DOM树中移除，相应的sheet属性也会去掉。**
>
> **对于有内容的style元素来说，样式表中的规则可以在元素重新插入页面的时候根据内容重新生成。但是对于没有内容的style元素来说，元素重新插入页面时无法还原之前的样式规则。**
>
> 这也是为什么在子应用卸载的时候需要对没有内容的`style`元素**手动存储**对应的`cssRules`





当`dynamicAppend`补丁卸载的时候，还有一个细节需要注意一下：

当`dynamicAppend`补丁卸载的时候，按照正常的思路就是需要将之前劫持的方法还原，但是前面提到过，这些方法是通过重写的方式来劫持的，所以修改的时候会影响其他子应用，所以需要等到**所有子应用的`dynamicAppend`补丁都卸载的时候才能还原被劫持的方法**。



#### 3.2.2 interval 补丁

这个补丁的代码实现比较简单，直接看代码就可以

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/ef328fe0972ebf2f1c9b7a1bc4898490db3fd085/src/sandbox/patchers/interval.ts#L12)

```typescript
export default function patch(global: Window) {
  let intervals: number[] = [];

  global.clearInterval = (intervalId: number) => {
    intervals = intervals.filter((id) => id !== intervalId);
    return rawWindowClearInterval(intervalId);
  };

  global.setInterval = (handler: CallableFunction, timeout?: number, ...args: any[]) => {
    const intervalId = rawWindowInterval(handler, timeout, ...args);
    intervals = [...intervals, intervalId];
    return intervalId;
  };

  return function free() {
    intervals.forEach((id) => global.clearInterval(id));
    global.setInterval = rawWindowInterval;
    global.clearInterval = rawWindowClearInterval;

    return noop;
  };
}
```

这个补丁是通过代理的方式来实现的，在`fakeWindow`中增加`setInterval和clearInterval`方法来代理`window`中的方法。不会影响到其他的应用，只在当前沙箱起作用。

从以上代码分析来看，**`interval`补丁的作用就是用来防止开发者在子应用卸载的时候没有清除`setInterval`定时器而导致的内存泄漏问题。**



#### 3.2.3 windowListener 补丁

比较简单，直接看代码

> 对应的源代码可以看[这里]()

```typescript
export default function patch(global: WindowProxy) {
  const listenerMap = new Map<string, EventListenerOrEventListenerObject[]>();

  global.addEventListener = (
    type: string,
    listener: EventListenerOrEventListenerObject,
    options?: boolean | AddEventListenerOptions,
  ) => {
    const listeners = listenerMap.get(type) || [];
    listenerMap.set(type, [...listeners, listener]);
    return rawAddEventListener.call(window, type, listener, options);
  };

  global.removeEventListener = (
    type: string,
    listener: EventListenerOrEventListenerObject,
    options?: boolean | AddEventListenerOptions,
  ) => {
    const storedTypeListeners = listenerMap.get(type);
    if (storedTypeListeners && storedTypeListeners.length && storedTypeListeners.indexOf(listener) !== -1) {
      storedTypeListeners.splice(storedTypeListeners.indexOf(listener), 1);
    }
    return rawRemoveEventListener.call(window, type, listener, options);
  };

  return function free() {
    listenerMap.forEach((listeners, type) =>
      [...listeners].forEach((listener) => global.removeEventListener(type, listener)),
    );
    global.addEventListener = rawAddEventListener;
    global.removeEventListener = rawRemoveEventListener;

    return noop;
  };
}
```

实现思路上和`interval`是一样的，也是通过在`fakeWindow`中代理的形式来实现的。

从代码上来看，**`windowListener`补丁的作用是防止子应用卸载的时候，内部绑定在`window`对象中的事件没有及时销毁而导致的内存泄漏或其他影响。**



## 4. 总结

至此，`qiankun`中JS沙箱的实现和应用基本就分析完了。

简单来说，可以总结以下几点：

1. `qiankun`中的提供了两种JS沙箱：`snapshotSandbox`和`proxySandbox`，目前大家基本使用的都是`proxySandbox`。该沙箱使用`fakeWindow`来替代`window`对象，并且代理了全局对象`fakeWindow`的操作

2. `qiankun`中每个子应用都会对应一个JS沙箱实例，这个沙箱实例会持久化的伴随着子应用，用来保存子应用的执行上下文信息，直到当前`qiankun`应用销毁。

3. `qiankun`给`JS`沙箱增加了补丁系统，用来完善沙箱的功能

