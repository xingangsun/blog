# 按顺序完成异步操作

> 实际开发中，经常遇到一组异步操作，需要按照顺序完成。比如，展示页面中有上中下三个部分，每一部分通过一个接口获得数据后就展示该部分区域内容，要求这三部分要自上而下显示，避免下面部分先展示，然后上面部分突然“窜出”影响体验。

## 思考点

1. 接口调用应该并行发出请求，而不是按顺序继发。  
2. 接口请求可能出现异常，每个接口的异常处理不尽相同，应该分开处理。  
3. 如果接口依次返回结果，当然可以直接展示数据。但是，如果后面部分的接口先返回结果，应该等前面接口结果返回并展示后才能展示。  
4. 各部分接口的请求和处理最好放在一起。

## 尝试解

```javascript
// 模拟API请求接口
function fetch (api, ms, err = false) {
  return new Promise(function (resolve, reject) {
    console.log(`fetch-${api}-${ms} start`)
    console.timeEnd('fetch')
    setTimeout(function () {
      err ? reject(`reject-${api}-${ms}`) : resolve(`resolve-${api}-${ms}`)
    }, ms)
  })
}

// 解法一
function loadData () {
  const promises = [fetch('API1', 3000), fetch('API2', 2000, true), fetch('API3', 5000)]
  promises.reduce((chain, promise, index) => {
    return chain.then(() => promise).then(data => console.log(data)).catch(data => console.error(data))
  }, Promise.resolve())
}

// 解法二
async function loadData () {
  const promises = [fetch('API1', 3000), fetch('API2', 2000, true), fetch('API3', 5000)]
  for (const promise of promises) {
    try {
      const data = await promise
      console.log(data)
    } catch (data) {
      console.error(data)
    }
  }
}
```

这两种解法都是可以的，但是确不能很好地将各部分接口的请求和处理放在一起。
其实，并发请求就是`fetch`函数的同时调用，但是返回的`promise`确需要我们控制其按顺序执行`then`或`catch`。所以我们可以考虑使用`Generator`函数的暂停-恢复执行功能。

```javascript
function* load1 () {
  const promise = yield fetch('API1', 3000)
  promise.then(function (data) {
    console.log(data)
  }).catch(function (data) {
    console.error(data)
  })
}

function* load2 () {
  const promise = yield fetch('API2', 2000, true)
  promise.then(function (data) {
    console.log(data)
  }).catch(function (data) {
    console.error(data)
  })
}

function* load3 () {
  const promise = yield fetch('API3', 5000)
  promise.then(function (data) {
    console.log(data)
  }).catch(function (data) {
    console.error(data)
  })
}

async function loadData () {
  console.time('fetch')
  const promises = [load1(), load2(), load3()].map(gen => ({ gen, promise: gen.next().value }))
  for (const { gen, promise } of promises) {
    try {
      await promise
    } catch (err) {
      console.error('catch error')
      console.timeEnd('fetch')
    } finally {
      console.info('finally', gen.next(promise))
      console.timeEnd('fetch')
    }
  }
}
```

上述代码，执行了`loadData`函数后，在控制台上的一次结果如下：

```javascript
fetch-API1-3000 start
fetch: 19.390ms
fetch-API2-2000 start
fetch: 22.986ms
fetch-API3-5000 start
fetch: 26.002ms
// 并发请求

Uncaught (in promise) reject-API2-2000
// 2000ms后API2请求出错
// 但是要等到API1请求结果返回并处理后才能处理

finally Object {value: undefined, done: true}
fetch: 3023.055ms
resolve-API1-3000
// 3000ms后API1请求结束，处理结果

catch error
fetch: 3025.530ms
finally Object {value: undefined, done: true}
fetch: 3026.752ms
reject-API2-2000
// 紧接着处理API2结果

finally Object {value: undefined, done: true}
fetch: 5029.639ms
resolve-API3-5000
// 5000ms后API3请求结束，处理结果
```

执行结果就是我们想要的。

## 最终解

可以将`loadData`函数提取为一个公共的函数，供多次使用。完整代码：

```javascript
/**
 * 按顺序加载异步请求数据
 * @param {...GeneratorFunction()} args GeneratorFunction函数执行返回值
 * @return {Promise} 返回一个Promise对象p。只要请求出错，就执行p的catch回调，否则执行then回调，回调参数为各个请求结果组成的数组
 */
async function loadDataInOrder (...args) {
  const promises = [...args].map(gen => ({ gen, promise: gen.next().value }))
  const result = []
  let hasErr = false
  for (const { gen, promise } of promises) {
    try {
      result.push(await promise)
    } catch (err) {
      result.push(err)
      hasErr = true
    } finally {
      gen.next(promise)
    }
  }
  if (hasErr) {
    throw result
  }
  return result
}

// 模拟API请求接口
function fetch (api, ms, err = false) {
  return new Promise(function (resolve, reject) {
    setTimeout(function () {
      err ? reject(`reject-${api}-${ms}`) : resolve(`resolve-${api}-${ms}`)
    }, ms)
  })
}

// 请求接口1
function* load1 () {
  (yield fetch('API1', 3000)).then(function (data) {
    console.log(data)
  }).catch(function (data) {
    console.error(data)
  })
}

// 请求接口2
function* load2 () {
  (yield fetch('API2', 2000, true)).then(function (data) {
    console.log(data)
  }).catch(function (data) {
    console.error(data)
  })
}

// 请求接口3
function* load3 () {
  (yield fetch('API3', 5000)).then(function (data) {
    console.log(data)
  }).catch(function (data) {
    console.error(data)
  })
}

// 按顺序加载异步请求
loadDataInOrder(load1(), load2(), load3()).then(function (result) {
  console.log('ok', result)
}).catch(function (result) {
  console.error('error', result)
})
```

