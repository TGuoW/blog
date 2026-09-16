# JavaScript 价格处理与精度问题

在前端开发中，处理“单价”（Unit Price）和金额时，经常会遇到精度丢失和格式化的问题。

## 1. 精度问题 (0.1 + 0.2 !== 0.3)

JavaScript 使用 IEEE 754 双精度浮点数，这导致某些小数运算不精确。

```javascript
console.log(0.1 + 0.2); // 0.30000000000000004
console.log(19.9 * 100); // 1989.9999999999998
```

### 解决方案

#### 方案 A：转换为整数运算
将金额乘以 100（变成“分”），运算后再除以 100。

```javascript
function add(num1, num2) {
  const m = Math.pow(10, 2); // 假设保留2位小数
  return (Math.round(num1 * m) + Math.round(num2 * m)) / m;
}

console.log(add(0.1, 0.2)); // 0.3
```

#### 方案 B：使用第三方库
推荐使用 `decimal.js`、`bignumber.js` 或 `currency.js`。

## 2. 价格格式化

### 使用 `Intl.NumberFormat`
这是原生 API，性能好且支持多语言。

```javascript
const price = 123456.789;

// 人民币格式
const cny = new Intl.NumberFormat('zh-CN', {
  style: 'currency',
  currency: 'CNY'
}).format(price);

console.log(cny); // "¥123,456.79"
```

### 使用 `toFixed` 的坑
`toFixed` 返回的是字符串，且遵循“四舍六入五成双”或者具体的浏览器实现差异，并不总是标准的四舍五入。

```javascript
(1.335).toFixed(2) // "1.33" (在某些环境下)
```
