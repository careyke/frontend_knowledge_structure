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

> 这种场景中，大部分可以通过修改节点的渲染位置来解决，也就是将节点渲染到内部子树中。但是在某些情况下是**不可调和**的，某些节点就无法渲染在内部子树中。（笔者曾经碰到过）



### 3.3 作用域沙箱 - scope

前面两个方案中，都存在一些比较致命的问题。为此`qiankun`提出了最终的解决方案——**作用域沙箱**。

作用域沙箱，顾名思义就是**限制子应用中样式的作用范围，让它只作用在子应用的节点中**。

**对于一个选择器，如果需要限制它的作用范围，可以使用组合选择器的方式**。在当前选择器A前面加一个选择器B，使得选择器A只作用在选择器B内部的节点。

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

上面方法中可以看出，**每个子应用的包裹`DOM`节点会添加一个自定义属性`data-qiankun`，而且属性值是唯一的，由此可以构造一个唯一的属性选择器来当做对应子应用的前缀选择器**。



对于子应用选择器的处理都发生在`css.ts`文件中，主要分成两个部分：

1. 构造前缀选择器
2. 重写所有选择器



#### 3.3.1 构造前缀选择器

对应的方法是`css.ts`文件中的`process`方法

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/bf03b50799b180f0f5b3e02c747d1fcdf5583348/src/sandbox/patchers/css.ts#L188)

```typescript
export const QiankunCSSRewriteAttr = 'data-qiankun';
export const process = (
  appWrapper: HTMLElement,
  stylesheetElement: HTMLStyleElement | HTMLLinkElement,
  appName: string,
): void => {
  // 单例模式
  if (!processor) {
    processor = new ScopedCSS();
  }

  // 不支持link外部引入样式
  if (stylesheetElement.tagName === 'LINK') {
    console.warn('Feature: sandbox.experimentalStyleIsolation is not support for link element yet.');
  }

  const mountDOM = appWrapper;
  if (!mountDOM) {
    return;
  }

  const tag = (mountDOM.tagName || '').toLowerCase();

  if (tag && stylesheetElement.tagName === 'STYLE') {
    // 构造前缀选择器
    const prefix = `${tag}[${QiankunCSSRewriteAttr}="${appName}"]`;
    processor.process(stylesheetElement, prefix);
  }
};
```

这里说一下作用域沙箱不支持`link`标签引入的样式，这也是在解析子应用静态资源时为什么把`link`样式转化为`style`样式的一个原因。



#### 3.3.2 重写所有选择器

选择器重写的过程都是在`ScopedCSS`这个类中完成，这里按照选择器添加的方式可以将**选择器分成三类**：

1. 当前`style`标签中存在的选择器
2. 动态添加到当前`style`标签中的选择器
3. 动态添加的`style`或者`link`标签中的选择器



##### 3.3.2.1 当前style标签中存在的选择器

对于这类选择器，直接解析样式文本，重写其中的选择器。对应的方法是`rewrite`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/bf03b50799b180f0f5b3e02c747d1fcdf5583348/src/sandbox/patchers/css.ts#L91)

```typescript
private rewrite(rules: CSSRule[], prefix: string = '') {
    let css = '';

    rules.forEach((rule) => {
        switch (rule.type) {
            case RuleType.STYLE:
                css += this.ruleStyle(rule as CSSStyleRule, prefix);
                break;
            case RuleType.MEDIA:
                css += this.ruleMedia(rule as CSSMediaRule, prefix);
                break;
            case RuleType.SUPPORTS:
                css += this.ruleSupport(rule as CSSSupportsRule, prefix);
                break;
            default:
                css += `${rule.cssText}`;
                break;
        }
    });

    return css;
}
```

可以看到，这里只对三种类型的规则进行了重写，其他的类型不处理。

> @keyframes, @font-face, @import, @page 将不会被重写。
>
> 能满足绝大部分场景



这里我们重点看一下对于普通样式规则的重写，对应的方法是`ruleStyle`

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/bf03b50799b180f0f5b3e02c747d1fcdf5583348/src/sandbox/patchers/css.ts#L119)

```typescript
private ruleStyle(rule: CSSStyleRule, prefix: string) {
    const rootSelectorRE = /((?:[^\w\-.#]|^)(body|html|:root))/gm;
    const rootCombinationRE = /(html[^\w{[]+)/gm;

    const selector = rule.selectorText.trim();

    let {
        cssText
    } = rule;
    // handle html { ... }
    // handle body { ... }
    // handle :root { ... }
    /**
     * 根节点选择器的处理
     */
    if (selector === 'html' || selector === 'body' || selector === ':root') {
        return cssText.replace(rootSelectorRE, prefix);
    }

    // handle html body { ... }
    // handle html > body { ... }
    /**
     * 根节点组合选择器的处理
     */
    if (rootCombinationRE.test(rule.selectorText)) {
        const siblingSelectorRE = /(html[^\w{]+)(\+|~)/gm;

        // since html + body is a non-standard rule for html
        // transformer will ignore it
        if (!siblingSelectorRE.test(rule.selectorText)) {
            cssText = cssText.replace(rootCombinationRE, '');
        }
    }

    // handle grouping selector, a,span,p,div { ... }
    /**
     * 其他选择器的处理
     */
    cssText = cssText.replace(/^[\s\S]+{/, (selectors) =>
        selectors.replace(/(^|,\n?)([^,]+)/g, (item, p, s) => {
            // handle div,body,span { ... }
            if (rootSelectorRE.test(item)) {
                return item.replace(rootSelectorRE, (m) => {
                    const whitePrevChars = [',', '('];

                    if (m && whitePrevChars.includes(m[0])) {
                        return `${m[0]}${prefix}`;
                    }

                    // replace root selector with prefix
                    return prefix;
                });
            }

            return `${p}${prefix} ${s.replace(/^ */, '')}`;
        }),
    );

    return cssText;
}
```

上面函数中将选择器分成**三种规则**来重写：

1. **对于子应用中的根选择器，比如html或者body，会将当前根选择器修改成前缀选择器**。也就是将包裹子应用的`DOM`当成是`body`节点，继承子应用的顶层样式。

   ```css
   // 子应用中
   body{
     color: red
   }
   body p{
     font-size: 14px
   }
   
   // 重写之后
   div[data-qiankun="react"]{
     color: red
   }
   div[data-qiankun="react"] p{
     font-size: 14px
   }
   
   // html选择器也是一样的
   ```

2. 对于子应用中的组合根选择器，比如`html body{}`，也会替换成前缀选择器

   ```css
   // 子应用中
   html body{
     color: red
   }
   
   // 重写之后
   div[data-qiankun="react"]{
     color: red
   }
   ```

3. **对于其他类型的选择器，前面增加前缀选择器**

   ```css
   // 子应用中
   .row{
     height: 200px
   }
   
   // 重写之后
   div[data-qiankun="react"] .row{
     height: 200px
   }
   ```

   

##### 3.3.2.2 动态添加到当前`style`标签中的选择器

对于这类选择器的重写规则其实是一样的，重要的是**需要监听`style`标签的变化，获取动态添加的内容。**

监听DOM元素的变化，可以使用`MutationObserver`，`qiankun`中也是使用这个API来监听`style`标签的变化

对应的代码在`ScopedCSS.process`方法中

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/bf03b50799b180f0f5b3e02c747d1fcdf5583348/src/sandbox/patchers/css.ts#L46)

```typescript
process(styleNode: HTMLStyleElement, prefix: string = '') {
		// ... 省略
    const mutator = new MutationObserver((mutations) => {
        /**
         * 针对动态添加样式类
         */
        for (let i = 0; i < mutations.length; i += 1) {
            const mutation = mutations[i];

            if (ScopedCSS.ModifiedTag in styleNode) {
                return;
            }

            if (mutation.type === 'childList') {
                const sheet = styleNode.sheet as any;
                const rules = arrayify < CSSRule > (sheet ? .cssRules ? ? []);
                const css = this.rewrite(rules, prefix);

                // eslint-disable-next-line no-param-reassign
                styleNode.textContent = css;
                // eslint-disable-next-line no-param-reassign
                (styleNode as any)[ScopedCSS.ModifiedTag] = true;
            }
        }
    });

    mutator.observe(styleNode, {
        childList: true
    });
}
```

可以看到，最终重写选择器调用的方法也是`rewrite`方法



##### 3.3.2.3 动态添加的`style`或者`link`标签中的选择器

这种情况下，我们要想获取动态添加的`style`或`link`标签里面的内容，可以**改写浏览器中增加DOM节点的API，从而来劫持动态添加的`style`和`link`标签。**

`qiankun`中也是使用这种方式来劫持动态添加的`style、link和script`标签。

> 对应的源代码可以看[这里](https://github.com/careyke/qiankun/blob/bf03b50799b180f0f5b3e02c747d1fcdf5583348/src/sandbox/patchers/dynamicAppend/common.ts#L149)

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
          let stylesheetElement: HTMLLinkElement | HTMLStyleElement = newChild as any;
          const { href } = stylesheetElement as HTMLLinkElement;
          if (excludeAssetFilter && href && excludeAssetFilter(href)) {
            return rawDOMAppendOrInsertBefore.call(this, element, refChild) as T;
          }

          const mountDOM = appWrapperGetter();

          if (scopedCSS) {
            // exclude link elements like <link rel="icon" href="favicon.ico">
            const linkElementUsingStylesheet =
              element.tagName?.toUpperCase() === LINK_TAG_NAME &&
              (element as HTMLLinkElement).rel === 'stylesheet' &&
              (element as HTMLLinkElement).href;
            if (linkElementUsingStylesheet) {
              const fetch =
                typeof frameworkConfiguration.fetch === 'function'
                  ? frameworkConfiguration.fetch
                  : frameworkConfiguration.fetch?.fn;
              // 先请求样式文本，再来重写选择器
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

          // eslint-disable-next-line no-shadow
          dynamicStyleSheetElements.push(stylesheetElement);
          const referenceNode = mountDOM.contains(refChild) ? refChild : null;
          return rawDOMAppendOrInsertBefore.call(mountDOM, stylesheetElement, referenceNode);
        }

        case SCRIPT_TAG_NAME: {
          // ...省略 script标签的处理 后面再介绍
        }

        default:
          break;
      }
    }

    return rawDOMAppendOrInsertBefore.call(this, element, refChild);
  };
}
```

上面方法可以看到，**不管是动态添加的style还是link标签，最终都会调用`css.process`方法来重写内部的选择器**



#### 3.3.3 作用域沙箱总结

至此，关于作用域沙箱的内容我们就讲完了。

整体看来作用域沙箱基本能满足用户的需求，但是目前还是一个实验期的特性，后续很多细节很可能会发生修改，但是主要的思路感觉不会改动。

作用域沙箱还有一个**比较大的缺点**，就是**运行时重写选择器是需要消耗时间的**，特别是对于比较大的子应用，这个可能是一个比较耗时的过程。



## 4. 总结

整个分析下来，我们清楚了为什么需要CSS沙箱，以及`qiankun`内部提供的三种沙箱的实现原理和优缺点。

在实际的工作中，我们需要根据具体的场景来选择沙箱。在单实例模式中，新建的子应用往往使用天然沙箱+工程化手段就能满足需求。

