# qiankun中CSS沙箱的实现

上一篇文章中我们分析了`qiankun`中子应用渲染的基本流程，但是跳过了两个重要的模块：

1. CSS沙箱
2. JS沙箱

这篇文章中我们来分析`qiankun`中`CSS`沙箱的实现原理。



## 1. 为什么需要CSS沙箱

在`qiankun`中每个子应用的开发和部署都是完全独立的，最终通过`qiankun`把主子项目的资源整合在一起，其中一个比较大的问题就是**样式冲突**的问题。而且也是一个非常棘手的问题

`iframe`方案之所以还有一席之地，出了使用简单之外，最大的优势是**它是一个完全封闭的环境，内部运行的样式和脚本不会影响外面。**

而在`qiankun`中，我们必须去实现这么一个封闭的运行环境 —— 也就是沙箱。

**CSS沙箱本质上就是为了解决各应用之间的样式冲突问题。**



## 2. 工程化手段

> 有这样一个问题？
>
> 既然CSS沙箱是用来解决样式冲突的问题，那如果我能**保证每个样式选择器名称都是唯一的**，这样是不是就可以不需要CSS沙箱了？

确实，在现代化的前端工程中，我们可以使用工程化的手段来生成唯一的`CSS`类名，常见的方案有：

1. BEM (Block Element Modifier): 不同项目用不同的前缀 / 命名规则来避免冲突
2. CSS-module: 通过编译工具生成唯一的类名
3. CSS-in-JS: `CSS`和`JS`编码在一起，最终生成不冲突的选择器



但是这些方案都存在一些**问题**：

1. **历史包袱**：对于新的子应用工程化的手段大概率能解决问题。但是在老的项目中，特别是那些没有工程化的老项目，改造的成本是非常大的。
2. **第三方模块**：如果子应该中引入了第三方库的样式，而且第三方库包含了很多全局样式，这个是工程化的方案无法解决的问题。



显然，工程化手段只能解决一部分问题，CSS沙箱还是有实现的必要的。



## 3. CSS沙箱的实现

在`qiankun`中一共实现了三种CSS沙箱

1. **天然沙箱**：单实例模式下，子应用之间的样式天然隔离。
2. **shadowDOM 沙箱**：利用`shadowDOM`来实现的沙箱。
3. **作用域沙箱（scope）**：构造一个作用域，使子应用的样式只对作用域里面的元素生效

下面我们结合源码来分别看看三种CSS沙箱的实现



### 3.1 天然沙箱

天然沙箱是`qiankun`在加载子应用时就构造出来的CSS沙箱。并没有经过针对性的处理，所以有以下的限制条件：

1. **只能在单实例模式下起作用**
2. **只能用来隔离子应用之间的样式，无法隔离主应用和子应用之间的样式**

其实现的原理是**每个子应用退出的时候，其对应的子应用模板也会卸载，也就是说对应的css样式也会卸载**。子应用重新激活的时候再重新加载对应样式。

天然沙箱的限制条件比较多，所以在现实的使用中基本无法单独满足用户的需求。

> 对于新的子应用，使用**天然沙县+工程化手段**两种方案结合的方式，基本能够解决样式冲突的问题。



### 3.2 shadow DOM 沙箱

如果不考虑浏览器的兼容性，提到样式隔离我们首先想到的就是`shadow DOM`。

`Shadow DOM`是`Web Component`中提出的一项技术，**用来将子树和页面中的其他元素隔绝开来，内部的样式只会用到内部节点，不会影响到外部节点。**

> 关于shadow DOM的定义和用法可以看[这里](https://developer.mozilla.org/zh-CN/docs/Web/Web_Components/Using_shadow_DOM)



`qiankun`中也使用了`shadow DOM`的技术来实现`CSS`沙箱。源代码中对应的方法是`createElement`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/35c354cf137adc1eb159caee7d0bb042ec42edda/src/loader.ts#L65)

```typescript
function createElement(
  appContent: string,
  strictStyleIsolation: boolean,
  scopedCSS: boolean,
  appName: string,
): HTMLElement {
  const containerElement = document.createElement('div');
  containerElement.innerHTML = appContent;
  // appContent always wrapped with a singular div
  const appElement = containerElement.firstChild as HTMLElement;

  if (strictStyleIsolation) {
    // shadow DOM的方式
    if (!supportShadowDOM) {
      console.warn('...');
    } else {
      const { innerHTML } = appElement;
      appElement.innerHTML = '';
      let shadow: ShadowRoot;

      if (appElement.attachShadow) {
        shadow = appElement.attachShadow({ mode: 'open' });
      } else {
        shadow = (appElement as any).createShadowRoot();
      }
      shadow.innerHTML = innerHTML;
    }
  }

  if (scopedCSS) {
    // ...省略 scoped 方案 
    // 后面介绍
  }

  return appElement;
}
```

上面代码中可以看出，实现的代码也很简单，就是**将子应用模板包裹在`shadow DOM`中，确保其样式隔离**。



#### 3.2.1 shadow DOM沙箱的缺陷

上面的描述中看起来`shadow DOM`是一个比较完美的方案，但是`qiankun`内部还实现了一个作用域沙箱，也就意味着`shadow DOM`沙箱也是有缺陷的。

**因为`shadow DOM`内部的样式只作用在内部的节点中，但是如果内部的脚本越界创建`DOM`时，必定会导致越界的`DOM`样式丢失的情况。**

这种用法在React应用中是很常见的，`ReactDOM.createPortal`就是应用在这种场景中的API。

> 这种场景中，大部分可以通过修改节点的渲染位置来解决，也就是将节点渲染到内部子树中。但是在某些情况下是**不可调和**的，某些节点就无法渲染在内部子树中。



### 3.3 作用域沙箱 - scope

前面两个方案中，都存在一些比较致命的问题。为此`qiankun`提出了最终的解决方案——**作用域沙箱**。

作用域沙箱，顾名思义就是**限制子应用中样式的作用范围，让它只作用在子应用的节点中**。

**对于一个选择器，如果需要显示它的作用范围，可以使用组合选择器的方式**。在当前选择器A前面加一个选择器B，使得选择器A只作用在选择器B内部的节点。

`qiankun`中的作用域沙箱就是使用这个原理来实现的。**给子应用中所有的样式选择器都加上一个前缀选择器**。

源码中实现的入口方法也是`createElement`

```typescript
function createElement(
  appContent: string,
  strictStyleIsolation: boolean,
  scopedCSS: boolean,
  appName: string,
): HTMLElement {
  const containerElement = document.createElement('div');
  containerElement.innerHTML = appContent;
  const appElement = containerElement.firstChild as HTMLElement;

  
  if (strictStyleIsolation) {
    // ...省略 shadow DOM
  }

  if (scopedCSS) {
    // scope方案
    const attr = appElement.getAttribute(css.QiankunCSSRewriteAttr);
    if (!attr) {
      appElement.setAttribute(css.QiankunCSSRewriteAttr, appName);
    }

    const styleNodes = appElement.querySelectorAll('style') || [];
    forEach(styleNodes, (stylesheetElement: HTMLStyleElement) => {
      css.process(appElement!, stylesheetElement, appName);
    });
  }

  return appElement;
}
```

可以看到这里会遍历每个`style`文本，然后给内部的选择器添加前缀选择器。



#### 3.3.1 重写子应用样式选择器

