#### Node 中模块是什么

模块是一个对象，里面封装了实现某个具体功能的所有函数和属性。它的具体代码实现如下：
```javascript
// lib/internal/modules/cjs/loader.js
function Module(id = '', parent) {
  this.id = id;
  this.path = path.dirname(id);
  this.exports = {};
  this.parent = parent;
  updateChildren(parent, this, false);
  this.filename = null;
  this.loaded = false;
  this.children = [];
}
```
所有的模块都是 Module 的实例.

#### require 模块加载机制

我们通常都是使用 require 函数来加载模块，require函数内部其实调用的是 Module._load 方法. 
```javascript
// 源码位置：lib/internal/modules/cjs/loader.js
// 注：代码做了简化，只关注主要逻辑，详细实现请自读源码完成
Module._load = function(request, parent, isMain) {
  var filename = Module._resolveFilename(request, parent);

  var cachedModule = Module._cache[filename];
  if (cachedModule) {
    return cachedModule.exports;

  if (NativeModule.exists(filename)) {
    return NativeModule.require(filename);
  }
  
  var module = new Module(filename, parent);
  Module._cache[filename] = module;

  try {
    module.load(filename);
    hadException = false;
  } finally {
    if (hadException) {
      delete Module._cache[filename];
    }
  }

  return module.exports;
};
```
根据源码我们可以总结出 require 机制的流程如下：
  - 计算绝对路径
  - 查看是否有缓存，如果有缓存，直接返回 Module._cache[filename].exports 属性
  - 查看是否为内置模块，如果是内置模块，直接调用 NativeModule.require(filename) 进行加载
  - 生成模块实例，存入缓存
  - 调用 module.load(filename) 方法封装模块，输出 module.exports 属性

#### 模块动态更新

观点：不建议在线上对 Node.js 进程做模块的热加载，原因如下
  - 清除模块的缓存需要清除 Module._cache 中的记录，和所有父引用，但这里有个问题，就是我们很难清除全部的父引用。
  - 清除的模块中包含定时器、socket连接等资源的需要手动释放