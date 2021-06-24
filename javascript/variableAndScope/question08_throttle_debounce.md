## 防抖和节流
在实际的场景中，**有一些事件会高频率的触发，就会导致事件回调代码不断的执行，但是实际上并不需要执行得如此频繁。比如resize事件，会造成很大的资源浪费。**

防抖和节流就是用来处理这种场景的，虽然**不能减少事件触发的次数，但是可以减少事件回调执行的次数。**
### 防抖（debounce）
**事件触发之后,延时一段时间n以后执行，如果这期间内这个事件还触发，则重新计时。**

代码实现:
```js
/**
 * 可以在开始时立马执行一次
 * @param {*} fn 
 * @param {*} delay 延时时间
 * @param {*} immediate 是否立刻触发一次
 */
function myDebounce(fn,delay,immediate = false){
  let timer=null;
  let called = false;
  function debounce(...args){
    const context = this;
    let result;
    if(immediate && !called){
      called = true;
      return fn.apply(context,args);
    }
    if(timer){
      clearTimeout(timer);
    }
    timer = setTimeout(()=>{
      result = fn.apply(context,args);
      timer = null;
    },delay)
    return result;
  }
  return debounce;
}
```
可以看到，使用的就是闭包和setTimeout函数来实现的。

### 节流（throttle）
**在事件频繁触发的过程中，使得事件回调函数每隔一段时间n触发一次。**

代码实现：
```js
/**
 * 头尾都可以执行，但是尾必执行
 * 因为使用的是setTimeout所以最后必执行一次
 * 如果使用的是时间戳方式计时，则开头必执行一次
 * @param {*} fn 
 * @param {*} delay 
 * @param {*} immediate 
 */
function myThrottle(fn,delay,immediate=false){
  let timer = null;
  let called = false;
  function throttle(...args){
    const context = this;
    let result;
    if(immediate && !called){
      called = true;
      return fn.apply(context, args);
    }
    if(timer){
      return result;
    }
    timer = setTimeout(()=>{
      result = fn.apply(context, args);
      timer = null;
    },delay);
    return result;
  }
  return throttle;
}
```























