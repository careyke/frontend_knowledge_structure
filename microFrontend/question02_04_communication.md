# qiankun中应用之间的通信方案

在使用`qiankun`拆分子应用的时候，建议是将功能相对独立完整的模块拆成子应用。如果两个模块交互频繁，数据相互依赖，这种情况下更适合作为一个子应用。

最理想的情况下是每个应用都是完全独立的，没有数据交互。然而在现实的场景中，应用之间不可能完全没有数据交互，最常见的就是子应用需要依赖主应用的数据来展示不同的内容。所以应用之间的数据通信是无可避免的。

`qiankun`内部使用**发布-订阅**的设计模式实现了一套应用之间的通信链路。



## 1. 通信方案的实现思路

整个方案在代码实现上是比较简单的，也很容易读懂。

> 对应的源代码目录是`/src/globalState.ts`，可以看[这里](https://github.com/careyke/qiankun/blob/littleknife/src/globalState.ts)

```typescript
import { cloneDeep } from 'lodash';

let globalState: Record<string, any> = {};

const deps: Record<string, OnGlobalStateChangeCallback> = {};

// 触发全局监听
function emitGlobal(state: Record<string, any>, prevState: Record<string, any>) {
  Object.keys(deps).forEach((id: string) => {
    if (deps[id] instanceof Function) {
      // 确保每个应用监听事件获取到的都是一个全新的副本
      deps[id](cloneDeep(state), cloneDeep(prevState));
    }
  });
}

export function initGlobalState(state: Record<string, any> = {}) {
  if (state === globalState) {
    console.warn('[qiankun] state has not changed！');
  } else {
    const prevGlobalState = cloneDeep(globalState);
    globalState = cloneDeep(state);
    emitGlobal(globalState, prevGlobalState);
  }
  return getMicroAppStateActions(`global-${+new Date()}`, true);
}

export function getMicroAppStateActions(id: string, isMaster?: boolean): MicroAppStateActions {
  return {
    onGlobalStateChange(callback: OnGlobalStateChangeCallback, fireImmediately?: boolean) {
      if (!(callback instanceof Function)) {
        console.error('[qiankun] callback must be function!');
        return;
      }
      if (deps[id]) {
        console.warn(`[qiankun] '${id}' global listener already exists before this, new listener will overwrite it.`);
      }
      deps[id] = callback;
      // const cloneState = cloneDeep(globalState);
      if (fireImmediately) {
        const cloneState = cloneDeep(globalState);
        callback(cloneState, cloneState);
      }
    },

    /**
     * setGlobalState 更新 store 数据
     *
     * 1. 对输入 state 的第一层属性做校验，只有初始化时声明过的第一层（bucket）属性才会被更改
     * 2. 修改 store 并触发全局监听
     */
    setGlobalState(state: Record<string, any> = {}) {
      if (state === globalState) {
        console.warn('[qiankun] state has not changed！');
        return false;
      }

      const changeKeys: string[] = [];
      const prevGlobalState = cloneDeep(globalState);
      globalState = cloneDeep(
        Object.keys(state).reduce((_globalState, changeKey) => {
          // 子应用只能修改state含有的属性，主应用可以添加属性
          if (isMaster || _globalState.hasOwnProperty(changeKey)) {
            changeKeys.push(changeKey);
            return Object.assign(_globalState, { [changeKey]: state[changeKey] });
          }
          console.warn(`[qiankun] '${changeKey}' not declared when init state！`);
          return _globalState;
        }, globalState),
      );
      if (changeKeys.length === 0) {
        console.warn('[qiankun] state has not changed！');
        return false;
      }
      emitGlobal(globalState, prevGlobalState);
      return true;
    },

    // 注销该应用下的依赖
    offGlobalStateChange() {
      delete deps[id];
      return true;
    },
  };
}
```

从实现上看是一个标准的`发布-订阅`的实现。

1. 全局维护一份状态（globalState）和订阅列表（deps）
2. 向外暴露添加订阅、发布和取消订阅的API



上面代码中我们可以看到，添加订阅和发布的API是包裹在`getMicroAppStateActions`方法中的，那么主应用和子应用是如何获取到这些API的呢？

1. 主应用 — 主应用是通过调用`initGlobalState`方法初始化`globalState`时同时获取到了订阅和发布的API

2. 子应用 — **子应用是在`load`阶段的时候调用`getMicroAppStateActions`生成API，然后在`mount`时将这些API传入到子应用中，在`unmount`时自动取消订阅。**

   ```typescript
   export async function loadApp<T extends ObjectType>(
     app: LoadableApp<T>,
     configuration: FrameworkConfiguration = {},
     lifeCycles?: FrameworkLifeCycles<T>,
   ): Promise<ParcelConfigObjectGetter> {
     // ...省略
     
     const {
       onGlobalStateChange,
       setGlobalState,
       offGlobalStateChange,
     }: Record<string, CallableFunction> = getMicroAppStateActions(appInstanceId);
     
     // async (props) => mount({ ...props, container: appWrapperGetter(), setGlobalState, onGlobalStateChange }),
     
     // async () => {offGlobalStateChange(appInstanceId);},
     
     // ...省略
   }
   ```



从代码实现上看，整个方案是比较简单的，但是其中包含了很多细节和规则。



## 2. 通信方案的实现细节

1. **每个应用只能注册一个订阅事件，多次添加订阅事件会覆盖。**



2. **`globalState`只能通过`发布API`来修改，且每次修改之后都会通过深拷贝的方式来重新生成一个全新的`globalState`。而且每个应用的订阅事件拿到的都是一个全新的深拷贝副本，确保每个应用拿到的globalState都不是同一个对象。**

   这样做能够**确保应用中直接修改`globalState`对象是无效的**，不会影响其他应用。强制规定了只有一种方式能够修改`globalState`，符合单一变量原则。



3. **子应用只能修改`globalState`中的属性，不能增加属性。主应用可以增加和修改属性。**

   这样做就要求开发者在使用的时候，初始化时就需要定义好`globalState`的结构。增加`globalState`数据的可回溯性和可维护性，如果子应用调用发布API可以增加属性的话，那`globalState`中的属性都不知道是哪个子应该添加的，长久下来比较难维护。

