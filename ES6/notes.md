## ES6回顾

### 1.let和var区别

`let` 只在当前花括号内可见，`var` 会逃到函数作用域，循环结束后外面还能读到它。

```js
for (let round = 0; round < 4; round++) {
  console.log(round)
}
// 这里再打印 round 会抛出未定义
```

### 2.Es6解构

数组按顺序拆：

```js
const pair = [3, 8]
const [left, right] = pair
console.log(left, right)
```

对象按键名拆，冒号右侧是新变量：

```js
const profile = {
  nick: '周宁',
  years: 24,
  city: '宁波',
  wave() {
    console.log('已签到')
  }
}
const { nick: label, years } = profile
console.log(label, years)
```

### 3.函数参数默认值

缺省参数写在形参上，调用时省略就走默认值。

```js
function toll(km, price = 3) {
  return km * price
}
console.log(toll(5))
```

### 4.箭头函数

只有一个表达式时可以省掉花括号和 `return`。它不绑定自己的 `this`，适合做回调。

```js
const toll2 = (km, price = 3) => km * price
console.log(toll2(5, 4))
```

### 5.map操作

`map` 返回等长新数组，原数组保持不动。

```js
const texts = ['9', '4']
const nums = texts.map(cell => Number(cell))
console.log(nums)
```

### 6.reduce操作

第二个参数是累加起点。下面从 50 开始把两笔账加进去。

```js
const fees = [7, 3]
const sum = fees.reduce((acc, cur) => acc + cur, 50)
console.log(sum)
```

### 7.promise

异步结果用 `resolve` / `reject` 交出去，调用方再用 `then` 或 `await` 接住。

```js
const job = new Promise((resolve, reject) => {
  setTimeout(() => {
    Math.random() > 0.3 ? resolve('ok') : reject('boom')
  }, 200)
})
```
