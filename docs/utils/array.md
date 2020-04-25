```javascript
/**
 * 数组降维
 */
const reduceDimension = (arr, result = []) => {
  const toArr = (arr) => {
    arr.forEach(element => {
      element instanceof Array ? toArr(element) : result.push(element)  
    });
  }
  toArr(arr)
  return result
}
```