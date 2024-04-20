# 结合qiankun源码分析基本执行流程

这篇文章开始，笔者会开始分析`qiankun`的源码，学习内部的实现原理。

在微前端架构中，对于子应用的加载可以分成两种方式：

1. 单实例：**同时只渲染一个子应用**：基于路由配置来**自动**加载子应用
2. 多实例：**同时渲染多个子应用**：由开发者**手动**来加载子应用

这里我们以单实例为例子来分析`qiankun`的执行流程

> 前面我们提到过，`qiankun`是基于`single-spa`来实现的，内部会涉及到一些`single-spa`的API调用，对应的API文档可以看[这里](https://zh-hans.single-spa.js.org/docs/getting-started-overview)

> 笔者fork的`qiankun`版本是`v2.4.0`和`v2.4.1`之间的`master`分支，fork的仓库可以看[这里](https://github.com/careyke/qiankun/tree/littleknife)



## 1. 注册子应用 — registerMicroApps

加载**基于路由**的子应用第一步先要注册所需要的子应用，`qiankun`中提供的对应的`api`是`registerMicroApps`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/35c354cf137adc1eb159caee7d0bb042ec42edda/src/apis.ts#L21)

```typescript
// 全局维护一个已注册的子应用列表
let microApps: Array<RegistrableApp<Record<string, unknown>>> = [];

export function registerMicroApps<T extends ObjectType>(
  apps: Array<RegistrableApp<T>>,
  lifeCycles?: FrameworkLifeCycles<T>,
) {
  // 保证每个子应用只注册一次
  const unregisteredApps = apps.filter((app) => !microApps.some((registeredApp) => registeredApp.name === app.name));

  microApps = [...microApps, ...unregisteredApps];

  unregisteredApps.forEach((app) => {
    const { name, activeRule, loader = noop, props, ...appConfig } = app;

    // single-spa的注册api
    registerApplication({
      name,
      app: async () => {
        loader(true);
        await frameworkStartedDefer.promise;

        const { mount, ...otherMicroAppConfigs } = (
          await loadApp({ name, props, ...appConfig }, frameworkConfiguration, lifeCycles)
        )();

        return {
          mount: [async () => loader(true), ...toArray(mount), async () => loader(false)],
          ...otherMicroAppConfigs,
        };
      },
      activeWhen: activeRule,
      customProps: props,
    });
  });
}
```

上面这个方法中有**两个关键点**：

1. `qiankun`内部维护了一个已注册的子应用列表，**每个子应用只允许注册一次**。也就是说`qiankun`不允许修改子应用的配置，也无法销毁。

   > 不允许修改其实是可以理解的，涉及到内部对于子应该的缓存处理，如果可以修改一个已经加载的子应用，还要清除内部的缓存，这是一个没有必要的操作。`single-spa`中也没有提供动态修改子应用配置的操作。
   >
   > 但是销毁子应用配置其实是有应用场景的，`single-spa`中提供了`unloadApplication`来实现这个操作，但是`qiankun`中并没有提供这种api。

2. 方法内部是调用`single-spa`的`registerApplication`方法来注册子应用。`qiankun`主要是在处理子应用的资源加载和运行过程，这个后面会介绍



这里我们大概介绍一些`single-spa`的`registerApplication`方法，这个方法的参数如下：

- name：子应用的名字，唯一标识
- app：子应用的资源加载方法，**每个子应用只会在第一次加载的时候调用一次**。返回值是子应用对应的生命周期函数
- activeWhen： 子应用激活的路由规则
- customProps：主应用传给子应用的额外数据，**一次性数据，无法动态修改**。



## 2. 启动应用 — start

在设计上，子应用的注册和启动是分开来执行的，这样设计可以让开发者更加合理地来控制自己的应用。

启动应用对应的方法是`start`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/35c354cf137adc1eb159caee7d0bb042ec42edda/src/apis.ts#L121)

```typescript
export function start(opts: FrameworkConfiguration = {}) {
  frameworkConfiguration = { prefetch: true, singular: true, sandbox: true, ...opts };
  const { prefetch, sandbox, singular, urlRerouteOnly, ...importEntryOpts } = frameworkConfiguration;

  if (prefetch) {
    // 预加载资源
    doPrefetchStrategy(microApps, prefetch, importEntryOpts);
  }

  // 根据运行环境采用对应的沙箱
  if (sandbox) {
    if (!window.Proxy) {
      console.warn('[qiankun] Miss window.Proxy, proxySandbox will degenerate into snapshotSandbox');
      frameworkConfiguration.sandbox = typeof sandbox === 'object' ? { ...sandbox, loose: true } : { loose: true };
      if (!singular) {
        console.warn(
          '[qiankun] Setting singular as false may cause unexpected behavior while your browser not support window.Proxy',
        );
      }
    }
  }

  // single-spa中的start方法
  startSingleSpa({ urlRerouteOnly });

  // 加载微应用静态资源的开关
  // 资源加载的时候会判断frameworkStartedDefer是否完成
  // 见registerMicroApps方法
  frameworkStartedDefer.resolve();
}
```

这个方法做了以下三个事情：

1. 开启子应用资源的预加载
2. 根据运行时环境配置沙箱
3. 调用single-spa的api启动应用

这里我们主要来分析一下`qiankun`内部如何实现资源的预加载



### 2.1 子应用资源预加载 — prefetch

资源预加载的入口方法是`doPrefetchStrategy`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/35c354cf137adc1eb159caee7d0bb042ec42edda/src/prefetch.ts#L106)

```typescript
// 预加载策略
export function doPrefetchStrategy(
  apps: AppMetadata[],
  prefetchStrategy: PrefetchStrategy,
  importEntryOpts?: ImportEntryOpts,
) {
  const appsName2Apps = (names: string[]): AppMetadata[] => apps.filter((app) => names.includes(app.name));

  // 预加载的两种策略
  if (Array.isArray(prefetchStrategy)) {
    prefetchAfterFirstMounted(appsName2Apps(prefetchStrategy as string[]), importEntryOpts);
  } else if (isFunction(prefetchStrategy)) {
    (async () => {
      // critical rendering apps would be prefetch as earlier as possible
      const { criticalAppNames = [], minorAppsName = [] } = await prefetchStrategy(apps);
      prefetchImmediately(appsName2Apps(criticalAppNames), importEntryOpts);
      prefetchAfterFirstMounted(appsName2Apps(minorAppsName), importEntryOpts);
    })();
  } else {
    switch (prefetchStrategy) {
      case true:
        prefetchAfterFirstMounted(apps, importEntryOpts);
        break;

      case 'all':
        prefetchImmediately(apps, importEntryOpts);
        break;

      default:
        break;
    }
  }
}
```

上面方法中主要使用了**两种策略**来预加载子应用的资源

1. **延迟预加载**：第一个子应用mount之后开始预加载其他子应用的资源
2. **立即预加载**：启动应用之后立即开始预加载子应用的资源



#### 2.1.1 延迟预加载

对应的方法是`prefetchAfterFirstMounted`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/35c354cf137adc1eb159caee7d0bb042ec42edda/src/prefetch.ts#L83)

```typescript
function prefetchAfterFirstMounted(apps: AppMetadata[], opts?: ImportEntryOpts): void {
  // single-spa中定义的自定义事件
  window.addEventListener('single-spa:first-mount', function listener() {
    const notLoadedApps = apps.filter((app) => getAppStatus(app.name) === NOT_LOADED);
    
    notLoadedApps.forEach(({ entry }) => prefetch(entry, opts));

    window.removeEventListener('single-spa:first-mount', listener);
  });
}
```

可以看到，这里是借助`single-spa`提供的**自定义事件**来监听第一个子应用何时挂载完成。



#### 2.1.2 立即预加载

对应的方法是`prefetchImmediately`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/35c354cf137adc1eb159caee7d0bb042ec42edda/src/prefetch.ts#L98)

```typescript
export function prefetchImmediately(apps: AppMetadata[], opts?: ImportEntryOpts): void {
  apps.forEach(({ entry }) => prefetch(entry, opts));
}
```

这个方法非常简单，就是调用`prefetch`来预加载资源。

下面来看看`prefetch`方法，看看`qiankun`是如何实现预加载的



#### 2.1.3 实现预加载 — requestIdleCallback

```typescript
const requestIdleCallback =
  window.requestIdleCallback ||
  // 模拟requestIdleCallback实现
  function requestIdleCallback(cb: CallableFunction) {
    const start = Date.now();
    return setTimeout(() => {
      cb({
        didTimeout: false,
        timeRemaining() {
          // 每帧最多只能给50ms的执行时间
          return Math.max(0, 50 - (Date.now() - start));
        },
      });
    }, 1);
  };

const isSlowNetwork = navigator.connection
  ? navigator.connection.saveData ||
    (navigator.connection.type !== 'wifi' &&
      navigator.connection.type !== 'ethernet' &&
      /(2|3)g/.test(navigator.connection.effectiveType))
  : false;

// 预加载的实现方法
function prefetch(entry: Entry, opts?: ImportEntryOpts): void {
  if (!navigator.onLine || isSlowNetwork) {
    // 网速慢的时候不预加载
    return;
  }

  // 利用requestIdleCallback 在空余时间预加载微应用资源
  requestIdleCallback(async () => {
    const { getExternalScripts, getExternalStyleSheets } = await importEntry(entry, opts);
    requestIdleCallback(getExternalStyleSheets);
    requestIdleCallback(getExternalScripts);
  });
}
```

可以看到`qiankun`内部借助`requestIdleCallback`来实现对资源的预加载。



## 3. 加载子应用资源 — import-html-entry

和`single-spa`的`JS Entry`不一样，`qiankun`加载资源的方式是`HTML Entry`。

对应的方法是`loadApp`，这个方法内容比较多，大致可以分成两个部分：

1. 资源加载
2. 资源执行

这里我们先来分析资源加载的部分。

```typescript
export async function loadApp<T extends ObjectType>(
  app: LoadableApp<T>,
  configuration: FrameworkConfiguration = {},
  lifeCycles?: FrameworkLifeCycles<T>,
): Promise<ParcelConfigObjectGetter> {
  const { entry, name: appName } = app;
  const appInstanceId = `${appName}_${+new Date()}_${Math.floor(Math.random() * 1000)}`;

  const markName = `[qiankun] App ${appInstanceId} Loading`;

  // 这里在 start 方法中已经给singular赋过缺省值
  // 手动加载微应用时不会走start
  const { singular = false, sandbox = true, excludeAssetFilter, ...importEntryOpts } = configuration;

  // 加载子应用的资源
  const { template, execScripts, assetPublicPath } = await importEntry(entry, importEntryOpts);
  
  // ...省略
}
```

可以看到这里加载资源的方法是`importEntry`，这个是`qiankun`的作者开源出来的一个npm包 — `import-html-entry`，对应的源代码可以看[这里](https://github.com/kuitos/import-html-entry/blob/1720f1a7ce/src/index.js)

这个库的代码不多，推荐大家花点时间去看看，这里我们来分析一下入口方法 — `importHTML`

```js
export default function importHTML(url, opts = {}) {
	let fetch = defaultFetch;
	let autoDecodeResponse = false;
	let getPublicPath = defaultGetPublicPath;
	let getTemplate = defaultGetTemplate;

	// 允许使用者自定义fetch、getPublicPath 和 getTemplate方法
	if (typeof opts === 'function') {
		fetch = opts;
	} else {
		// fetch option is availble
		if (opts.fetch) {
			// fetch is a funciton
			if (typeof opts.fetch === 'function') {
				fetch = opts.fetch;
			} else { // configuration
				fetch = opts.fetch.fn || defaultFetch;
				autoDecodeResponse = !!opts.fetch.autoDecodeResponse;
			}
		}
    // 这里涉及到资源publicPath的处理，可以详细看看默认的处理逻辑
		getPublicPath = opts.getPublicPath || opts.getDomain || defaultGetPublicPath;
		getTemplate = opts.getTemplate || defaultGetTemplate;
	}

	return embedHTMLCache[url] || (embedHTMLCache[url] = fetch(url) // 缓存已加载资源
		.then(response => readResAsString(response, autoDecodeResponse))
		.then(html => {
			
			const assetPublicPath = getPublicPath(url); // 静态资源文件夹路径
			const { template, scripts, entry, styles } = processTpl(getTemplate(html), assetPublicPath);

			return getEmbedHTML(template, styles, { fetch }).then(embedHTML => ({
				template: embedHTML,
				assetPublicPath,
				getExternalScripts: () => getExternalScripts(scripts, fetch),
				getExternalStyleSheets: () => getExternalStyleSheets(styles, fetch),
				execScripts: (proxy, strictGlobal, execScriptsHooks = {}) => {
					if (!scripts.length) {
						return Promise.resolve();
					}
					return execScripts(entry, scripts, proxy, {
						fetch,
						strictGlobal,
						beforeExec: execScriptsHooks.beforeExec,
						afterExec: execScriptsHooks.afterExec,
					});
				},
			}));
		}));
}
```

这里我们主要看一下这个方法的返回值，是一个`promise`对象，该`promise`执行完成的决议值有以下**属性**：

- template: `html`文件解析出来的字符串，默认是`head`和`body`标签里面的内容（不包含`head`和`body`标签）。
- assetPublicPath：当前子应用**静态资源文件夹**的路径
- getExternalScripts：获取子应用`html`文件中引入的`js`文件的**文本内容**，包括行内`script`。
- getExternalStyleSheets：获取子应用`html`文件中引入的`css`文件的**文本内容**，包括行内`style`
- execScripts：直接执行所依赖的`js`，可以执行的环境，也就是沙箱。



这里的`getEmbedHTML`方法需要分析一下，涉及到后续的分析

> 对应的源代码可以看[这里](https://github.com/kuitos/import-html-entry/blob/1720f1a7ce06a6f948327b3d66921a724375ded5/src/index.js#L36)

```js
function getEmbedHTML(template, styles, opts = {}) {
	const { fetch = defaultFetch } = opts;
	let embedHTML = template;

	return getExternalStyleSheets(styles, fetch)
		.then(styleSheets => {
			embedHTML = styles.reduce((html, styleSrc, i) => {
				html = html.replace(genLinkReplaceSymbol(styleSrc), `<style>/* ${styleSrc} */${styleSheets[i]}</style>`);
				return html;
			}, embedHTML);
			return embedHTML;
		});
}
```

可以看到，这个方法中**将外部引入的样式文件转换成了使用`<style>`标签的内联样式**。

> TODO：这样做的好处是什么？
>
> 1. 这样做可以减少一次请求
> 2. 作用域沙箱不支持`link`样式



### 3.1 JS Entry VS HTML Entry

#### 3.1.1 JS Entry

`JS Entry` 指的是以子应用的`js`文件作为渲染的入口。代表框架是`single-spa`

这种做法有一个要求，就是需要**将子应用中所有的资源都打包到同一个`js`文件**中，这种做法的**缺点**十分明显。

1. 构建时很多优化无法进行，比如按需加载，css独立打包等
2. 单一文件巨大，加载耗时长，资源并行加载的特性也无法使用
3. 子应用需要修改打包配置，接入成本大
4. 主应用需要和子应用**强约定**挂载的`DOM`节点标志



> 优点暂时感觉不明显
>
> 硬要说的话，主应用和子应该的资源可以一起打包，方便构建时优化一些重复资源。（不能说服自己）



#### 3.1.2 HTML Entry

`HTML Entry` 指的是以子应用的`html`文件作为渲染的入口。最大程度上**保留子应用的完整环境上下文**，使得接入子应该像使用`iframe`一样简单。

**优点**：

1. 子应用可以完全独立，开发、测试和部署，具有和独立应用一样的开发体验
2. 接入成本小，像使用iframe一样简单
3. 主子应用不需要约定挂载节点的信息



缺点：

1. 多请求一次`html`文件资源
2. 各子应用的资源或多或少存在重复



## 4. 执行子应用资源

接着看`loadApp`中关于资源执行的内容

```typescript
export async function loadApp<T extends ObjectType>(
  app: LoadableApp<T>,
  configuration: FrameworkConfiguration = {},
  lifeCycles?: FrameworkLifeCycles<T>,
): Promise<ParcelConfigObjectGetter> {
  // ... 省略 资源加载相关代码
  
  // ... 省略 CSS沙箱相关代码
  
  // ... 省略 构建JS沙箱相关代码
  
  const { beforeUnmount = [], afterUnmount = [], afterMount = [], beforeMount = [], beforeLoad = [] } = mergeWith(
    {},
    getAddOns(global, assetPublicPath), // 注入子应用 运行的引擎信息和子应用资源的publicPath
    lifeCycles,
    (v1, v2) => concat(v1 ?? [], v2 ?? []),
  );

  await execHooksChain(toArray(beforeLoad), app, global);

  // 获取子应用生命周期函数
  const scriptExports: any = await execScripts(global, !useLooseSandbox);
  // 子应用导出的生命周期
  const { bootstrap, mount, unmount, update } = getLifecyclesFromExports(
    scriptExports,
    appName,
    global,
    sandboxContainer?.instance?.latestSetProp,
  );

  // 应用之间通信相关的代码
  const {
    onGlobalStateChange,
    setGlobalState,
    offGlobalStateChange,
  }: Record<string, CallableFunction> = getMicroAppStateActions(appInstanceId);

  // FIXME temporary way
  const syncAppWrapperElement2Sandbox = (element: HTMLElement | null) => (initialAppWrapperElement = element);

  const parcelConfigGetter: ParcelConfigObjectGetter = (remountContainer = initialContainer) => {
    let appWrapperElement: HTMLElement | null = initialAppWrapperElement;
    const appWrapperGetter = getAppWrapperGetter(
      appName,
      appInstanceId,
      !!legacyRender,
      strictStyleIsolation,
      scopedCSS,
      () => appWrapperElement,
    );

    // 子应用的配置集合
    const parcelConfig: ParcelConfigObject = {
      name: appInstanceId,
      bootstrap,
      mount: [
        async () => {
          if ((await validateSingularMode(singular, app)) && prevAppUnmountedDeferred) {
            return prevAppUnmountedDeferred.promise;
          }

          return undefined;
        },
        // 添加 mount hook, 确保每次应用加载前容器 dom 结构已经设置完毕
        async () => {
          const useNewContainer = remountContainer !== initialContainer;
          if (useNewContainer || !appWrapperElement) {
            // element will be destroyed after unmounted, we need to recreate it if it not exist
            // or we try to remount into a new container
            appWrapperElement = createElement(appContent, strictStyleIsolation, scopedCSS, appName);
            syncAppWrapperElement2Sandbox(appWrapperElement);
          }
					// 将子应用的模板添加到主应用的挂载节点中
          render({ element: appWrapperElement, loading: true, container: remountContainer }, 'mounting');
        },
        mountSandbox,
        // exec the chain after rendering to keep the behavior with beforeLoad
        async () => execHooksChain(toArray(beforeMount), app, global),
        async (props) => mount({ ...props, container: appWrapperGetter(), setGlobalState, onGlobalStateChange }), // 执行子应用的mount方法
        // finish loading after app mounted
        async () => render({ element: appWrapperElement, loading: false, container: remountContainer }, 'mounted'),
        async () => execHooksChain(toArray(afterMount), app, global),
        // initialize the unmount defer after app mounted and resolve the defer after it unmounted
        async () => {
          if (await validateSingularMode(singular, app)) {
            prevAppUnmountedDeferred = new Deferred<void>();
          }
        },
      ],
      unmount: [
        async () => execHooksChain(toArray(beforeUnmount), app, global),
        async (props) => unmount({ ...props, container: appWrapperGetter() }),
        unmountSandbox,
        async () => execHooksChain(toArray(afterUnmount), app, global),
        async () => {
          // 移除子应用模板
          render({ element: null, loading: false, container: remountContainer }, 'unmounted');
          offGlobalStateChange(appInstanceId);
          // for gc
          appWrapperElement = null;
          syncAppWrapperElement2Sandbox(appWrapperElement);
        },
        async () => {
          if ((await validateSingularMode(singular, app)) && prevAppUnmountedDeferred) {
            prevAppUnmountedDeferred.resolve();
          }
        },
      ],
    };

    if (typeof update === 'function') {
      parcelConfig.update = update; // 子应用暴露的update生命周期
    }

    return parcelConfig;
  };

  return parcelConfigGetter;
}
```

上面方法中可以看出，**在`single-spa`中，子应用的生命周期方法可以是一个函数数组。利用这一点，`qiankun`在`single-spa`的基础上给子应用额外增加了很多生命周期钩子函数。**

`qiankun`中子应用**完整的生命周期方法**如下：

> 针对于**单实例**加载方式来说

1. beforeLoad: Array<Function>
2. bootstrap: Function
3. beforeMount: Array<Function>
4. mount: Function
5. afterMount: Array<Function>
6. update: Function
7. beforeUnmount: Array<Function>
8. unmount: Function
9. afterUnmount: Array<Function>



### 4.1 子应用的渲染流程

子应该的渲染流程宏观上可以分成三个阶段：

1. 子应用模板的挂载
2. 子应用内容的渲染和销毁
3. 子应用模板的销毁

> 模板：指的是子应用`html`文件中解析出来的模板节点
>
> 内容：指的是子应用渲染出来挂载在模板中的内容



#### 4.1.1 子应用模板的挂载

子应用模板的挂载调用的方法是`render`，这个方法由`getRender`方法返回。

上面代码中可以看出，`render`方法在子应用的`single-spa-mount`阶段会被调用。

> `single-spa-mount` 表示的是single-spa中定义的子应用的mount阶段

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/35c354cf137adc1eb159caee7d0bb042ec42edda/src/loader.ts#L166)

```typescript
function getRender(appName: string, appContent: string, legacyRender?: HTMLContentRender) {
  const render: ElementRender = ({ element, loading, container }, phase) => {
    // ...省略 老版本兼容代码
    const containerElement = getContainer(container!);

    if (phase !== 'unmounted') {
      const errorMsg = (() => {
        switch (phase) {
          case 'loading':
          case 'mounting':
            return `[qiankun] Target container with ${container} not existed while ${appName} ${phase}!`;

          case 'mounted':
            return `[qiankun] Target container with ${container} not existed after ${appName} ${phase}!`;

          default:
            return `[qiankun] Target container with ${container} not existed while ${appName} rendering!`;
        }
      })();
      assertElementExist(containerElement, errorMsg);
    }

    if (containerElement && !containerElement.contains(element)) {
      while (containerElement!.firstChild) {
        rawRemoveChild.call(containerElement, containerElement!.firstChild);
      }

      if (element) {
        // 插入模板节点
        rawAppendChild.call(containerElement, element);
      }
    }

    return undefined;
  };

  return render;
}
```

代码逻辑比较清晰，无需额外分析。



#### 4.1.2 子应用内容的挂载和销毁

子应该内容的挂载和销毁分别发生在子应用的`mount`和`unmount`生命周期中，具体的销毁逻辑由子应用自己来实现。

比如在React应用中：

- mount: 调用`ReactDOM.render()`来渲染内容
- unmount：调用`ReactDOM.unmountComponentAtNode()` 来销毁内容



#### 4.1.3 子应用模板的销毁

上面生命周期的代码中可以看出，子应用模板的销毁调用的方法仍是`render`，发生在`single-spa-unmount`阶段中。



## 5. 总结

至此，对于`qiankun`中单实例场景的基本流程就分析完了，整体流程来说还是比较简单明了的。这篇文章主要是用来了解`qiankun`的基本流程，后续笔者会详细分析内部重要模块的实现细节。包括**CSS沙箱**和**JS沙箱**的实现



## 6. 参考文章

1. [HTML Entry 源码分析](https://juejin.cn/post/6885212507837825038#heading-0)

