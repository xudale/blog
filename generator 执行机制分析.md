# generator 执行机制分析
本文以下面代码为例

```JavaScript
function* test() {
  yield 123456
}
let iterator = test()
iterator.next()
```

分析 generator 执行机制相关的源码，版本为 V8 7.7.1。
## 初始化——看不见的 yield










## iterator.next() 





## async/await 与 generator 的关系

## 3 种函数














