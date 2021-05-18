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
- 在沙箱卸载的时候，对应当前`window`对象和之前保存的`window`快照对象，收集被修改的属性，然后使用快照还原`window`对象。

由此刚好形成一个闭环操作。通过在沙箱激活和卸载的时候更新`window`对象来达到不影响子应用的效果（仅在单实例模式下）。



### 1.1 快照沙箱的缺点

从上面的快照沙箱的实现逻辑中可以看出，快照沙箱有以下缺点：

1. **只能支持单实例模式**：因为快照沙箱中子应用是直接在window对象上操作的，所以如果是多实例模式中，两个子应用都读写window对象中的同一属性，必然会出现相互覆盖的情况。
2. **沙箱切换的效率不高**：每次切换都需要遍历`window`中所有的属性，是一个不小的开销。



## 2. proxySandbox — 代理沙箱

代理沙箱的实现原理是：**每个实例创建一个全局对象（window）的替代对象（fakeWindow），并且使用`proxy`来代理对全局对象的操作，使其操作在替代对象中**。

代理沙箱对应的实现文件是`/src/sandbox/proxySandbox.ts`，可以分成两个部分

1. **创建全局对象的替代对象**
2. **使用`proxy`来代理全局对象的操作**



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





### 2.2 使用proxy代理全局对象的操作

