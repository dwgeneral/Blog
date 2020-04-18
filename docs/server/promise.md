> 最近在看 Kyle Simpson 在 FrontMasters 上的 JavaScript: The Recent Parts 视频，
> 对，就是《You Don't Know JS》的作者。为了巩固对知识的理解，打算再次重写一下 Promise实现。

```javascript
/**
 * AwesomePromise to implement Promise A+ rules
 */
const PENDING = 'pending'
const FULLFILLED = 'fullfilled'
const REJECTED = 'rejected'

module.exports = class AwesomePromise {
  constructor(executor) {
    this.state = PENDING    // promise 状态属性，初始值为 pending，可选值为 [fullfilled, rejected]
    this.value = undefined  // promise 存储结果数据属性，默认为 undefined
    this.callbacks = []     // promise 存储指定的回调函数的数组，数组元素为 { onResolved() {}, onRejected() {} }

    function resolve(value) {
      if (this.state != PENDING) return
      this.state = FULLFILLED
      this.value = value
      if (!this.callbacks.length) return
      setTimeout(() => {
        this.callbacks.forEach(callbacksObject => callbacksObject.onResolved(this.value))
      })
    }

    function reject(reason) {
      if (this.state != PENDING) return
      this.state = REJECTED
      this.value = reason
      if (!this.callbacks.length) return
      setTimeout(() => {
        this.callbacks.forEach(callbacksObject => callbacksObject.onRejected(this.value))
      })
    }

    try {
      executor(resolve, reject)
    } catch (error) {
      reject(error)         // 如果 executor 抛出异常，promise对象变为rejected状态
    }
  }

  then(onResolved, onRejected) {
    if(typeof onResolved != 'function') onResolved = value => value
    if(typeof onRejected != 'function') onRejected = reason => { throw reason }
    return new AwesomePromise((resolve, reject) => {
      function handle(callback) {
        try {
          const result = callback(this.value)
          if (result instanceof AwesomePromise) result.then(resolve, reject)
          else resolve(result)
        } catch (error) {
          reject(error)
        }
      }

      if (this.state === PENDING) {
        this.callbacks.push({ onResolved(value) {handle(onResolved)}, onRejected(reason) {handle(onRejected)} })
      }
      else if (this.state === FULLFILLED) setTimeout(() => handle(onResolved));
      else setTimeout(() => handle(onRejected))
    })
  }

  catch(onRejected) {
    return this.then(undefined, onRejected)
  }

  static resolve(value) {
    return new AwesomePromise((resolve, reject) => {
      if (value instanceof AwesomePromise) value.then(resolve, reject)
      else resolve(value)
    })
  }

  static reject(reason) {
    return new AwesomePromise((resolve, reject) => reject(reason))
  }

  static all(promises) {

  }

  static race(promises) {

  }

  static resolveDelay(value, delay) {

  }

  static rejectDalay(reson, delay) {

  }
}
```

参考资料：
- https://es6.ruanyifeng.com/
- https://frontendmasters.com/courses/js-recent-parts/
- https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise