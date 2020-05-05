```javascript
/**
 * 闭包
 */
{
  function add() {
    var a = 1
    return function() {
      return ++a
    }
  }
  var acc = add()
  console.log(acc())  // 2
  console.log(acc())  // 3
  console.log(acc())  // 4
}
```