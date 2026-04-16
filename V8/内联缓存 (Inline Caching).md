---
tags:
  - v8
  - inline-caching
  - optimization
  - javascript
  - performance
created: 2026-04-15
updated: 2026-04-15
---
## 概述

内联缓存是 JavaScript 引擎用来加速**动态属性访问**的核心优化技术。

## 问题背景

JavaScript 是动态类型语言，对象属性访问需要：

1. 查找对象的隐藏类 (Map/Shape)
2. 在属性偏移表中查找
3. 根据偏移量读取值

每次访问都执行完整查找非常慢。

## 工作原理

```javascript
const obj = { x: 1, y: 2 };
obj.x; // 第一次访问：慢速查找
obj.x; // 第二次访问：使用缓存
```

### 缓存内容

- **对象 Map 指针** - 记录上次访问的对象形状
- **属性偏移量** - 记录属性在内存中的位置

### 执行流程

```
obj.x 访问
    ↓
检查缓存中的 Map 是否匹配
    ↓
匹配 → 直接使用偏移量读取 (快)
不匹配 → 重新查找 + 更新缓存 (慢)
```

## 多态级别

| 级别 | 说明 | 性能 |
|------|------|------|
| **Monomorphic** | 只见过 1 种 Map | 最快 |
| **Polymorphic** | 见过 2-4 种 Map | 中等 |
| **Megamorphic** | 见过 5+ 种 Map | 最慢 |

```javascript
// Monomorphic - 最优
function getX(obj) {
  return obj.x; // 总是同一种对象形状
}

// Polymorphic - 可接受
function getX(obj) {
  return obj.x; // 2-4 种不同形状
}

// Megamorphic - 最差
function getX(obj) {
  return obj.x; // 太多不同形状
}
```

## 实际示例

```javascript
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
}

const p1 = new Point(1, 2);
const p2 = new Point(3, 4);

// 内联缓存生效
function sumX(a, b) {
  return a.x + b.x; // Monomorphic: 两个 Point 实例共享同一 Map
}

sumX(p1, p2); // 快速路径
```

## 导致缓存失效的情况

```javascript
// 1. 属性添加顺序不同
const a = { x: 1 };
const b = {};
b.x = 1; // 不同 Map

// 2. 删除属性
const obj = { x: 1, y: 2 };
delete obj.y; // Map 改变

// 3. 动态属性名
obj[key]; // 无法内联缓存，只能用字典查找
```

## 性能影响

```javascript
// 慢 - Megamorphic
function getValue(obj) {
  return obj.value; // 10 种不同对象形状
}

// 快 - Monomorphic
function getValue(obj) {
  return obj.value; // 只有 1 种对象形状
}

// 性能差异可达 10-100 倍
```

## 调试方法

```bash
# 查看内联缓存状态
node --trace-ic script.js

# 输出示例:
# IC state: 0x12345678: [Property] in [Function]
#   MONOMORPHIC
```

## 最佳实践

1. **保持对象形状一致** - 相同属性、相同顺序初始化
2. **避免动态添加/删除属性**
3. **使用类或构造函数** - 确保实例共享同一 Map
4. **避免在热路径使用动态键名**

## 参考资料

- [JavaScript engine fundamentals: Shapes and Inline Caches](https://mathiasbynens.be/notes/shapes-ics)
- [V8 Inline Caching](https://v8.dev/blog/ic)
