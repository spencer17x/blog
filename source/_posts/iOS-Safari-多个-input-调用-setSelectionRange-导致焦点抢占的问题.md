---
title: iOS Safari 多个 input 调用 setSelectionRange 导致焦点抢占的问题
date: 2026-06-20 12:58:00
tags:
    - Front-end
---

# 背景

在移动端 H5 表单里，经常会遇到这样的需求：用户输入手机号、银行卡号、验证码、金额等内容时，前端需要格式化输入值，同时把光标恢复到合适的位置。

常见做法是调用 `input.setSelectionRange(start, end)` 来恢复选区或者光标位置。

这个 API 在大多数桌面浏览器上表现很稳定，但在 iOS Safari 上需要特别小心。一次排查中发现：如果页面上有多个 `input`，并且代码批量对这些输入框调用 `setSelectionRange`，在 iOS 16.4 Safari 上可能会出现焦点被后面的输入框抢走的问题。

最终现象是：

1. 当前并不想聚焦某个输入框。
2. 代码只是想设置多个输入框的选区。
3. iOS 16.4 Safari 会在调用过程中触发 `focusin`。
4. 最后一个被调用 `setSelectionRange` 的输入框拿到焦点。
5. 键盘被唤起，页面滚动位置也可能跟着变化。

# 最小复现

我写了一个极简 demo：

```
/demos/ios-setselectionrange-minimal.html
```

核心代码如下：

```js
const inputs = Array.from(document.querySelectorAll('input'));

function batchSetSelection() {
  inputs.forEach(input => {
    const start = Math.min(2, input.value.length);
    const end = input.value.length;

    input.setSelectionRange(start, end);

    console.log(
      input.dataset.name,
      `setSelectionRange(${start}, ${end})`,
      'active=',
      document.activeElement.dataset.name || document.activeElement.tagName
    );
  });
}
```

页面中有四个输入框：

```html
<input data-name="input A" value="A-13800138000">
<input data-name="input B" value="B-24681012">
<input data-name="input C" value="C-hello-ios-safari">
<input data-name="input D" value="D-last-input">
```

点击按钮以后，代码会按 A、B、C、D 的顺序依次调用 `setSelectionRange`。

在 iOS 16.4 Safari 中，日志类似这样：

```txt
focusin: input B; active=input B
focusin: input C; active=input C
focusin: input D; active=input D
input D setSelectionRange(2, 12); active=input D
end; active=input D
```

可以看到，代码没有主动调用 `focus()`，但 `setSelectionRange` 仍然让输入框依次获得了焦点。最后焦点停在 `input D`，键盘也被唤起。

复现截图如下。点击按钮触发批量 `setSelectionRange` 后，页面焦点最终落在 `input D`，同时 iOS 键盘被唤起：

![iOS 16.4 Safari setSelectionRange 抢占焦点复现截图](/articleImgs/iOS-Safari-多个-input-调用-setSelectionRange-导致焦点抢占的问题/ios-16-4-focus-steal.jpg)

# 版本对比

我分别在几个 iOS Simulator Safari 上验证了这个 demo。

| 环境 | 结果 |
| --- | --- |
| iPhone 14 / iOS 16.4 / Safari | 可以复现，焦点最终停在 `input D` |
| iPhone 16 / iOS 18.5 / Safari | 未复现 |
| iPhone 17 / iOS 26.5 / Safari | 未复现 |

所以目前可以先把这个问题归类为：**iOS 16 Safari 中，`setSelectionRange` 对非当前输入框调用时，可能存在隐式聚焦行为；较新的 iOS Safari 行为已经变化，至少在 iOS 18.5 和 iOS 26.5 Simulator 中没有复现。**

# 为什么会出问题

`setSelectionRange` 的语义是设置输入框里的选区范围。

但是在移动端浏览器里，选区、焦点、虚拟键盘、页面滚动经常是耦合在一起的。尤其是 iOS Safari，输入框获得焦点后，浏览器还要处理键盘弹出、输入框可见区域、文本选区菜单等事情。

从 iOS 16.4 的现象看，可以理解为：

1. 对非当前 `activeElement` 的输入框调用 `setSelectionRange`。
2. Safari 为了展示或者维护这个输入框的选区，让这个输入框进入焦点状态。
3. 多个输入框连续调用时，焦点一路从前面的输入框跳到后面的输入框。
4. 最后一个输入框成为最终焦点。

这个行为对业务代码非常隐蔽，因为调用方可能只是想“恢复光标”，并没有意识到它会改变当前页面焦点。

# 常见错误写法

比较容易踩坑的是在组件初始化、列表渲染、表单更新后，批量处理所有输入框：

```js
function restoreAllSelections(fields) {
  fields.forEach(field => {
    const input = field.ref;

    if (!input) return;

    input.setSelectionRange(field.selectionStart, field.selectionEnd);
  });
}
```

这段代码在桌面端看起来没什么问题，但在 iOS 16.4 Safari 上，可能会导致最后一个 `input` 抢走焦点。

还有一种情况是输入格式化组件内部没有判断当前焦点：

```js
function formatAndRestore(input, value, selectionStart) {
  input.value = formatValue(value);
  input.setSelectionRange(selectionStart, selectionStart);
}
```

如果这个函数被多个输入框的更新流程调用，也会有同样风险。

# 解决方案

## 只处理当前 activeElement

最核心的原则是：**不要对非当前焦点输入框调用 `setSelectionRange`。**

可以封装一个安全方法：

```js
function safeSetSelectionRange(input, start, end, direction) {
  if (!input) return;
  if (document.activeElement !== input) return;

  input.setSelectionRange(start, end, direction);
}
```

这样即使页面上有多个输入框，也只有当前正在编辑的输入框会被恢复光标。

在同一台 iOS 16.4 Simulator 上触发安全写法后，页面仍然保持 `activeElement = body`，没有输入框被抢占焦点，键盘也没有弹出：

![iOS 16.4 Safari 使用安全写法后未抢占焦点](/articleImgs/iOS-Safari-多个-input-调用-setSelectionRange-导致焦点抢占的问题/ios-16-4-safe-active-only.png)

## 在用户输入事件里恢复光标

如果需求是“用户输入后格式化并恢复光标”，建议把恢复动作限制在当前事件目标上：

```js
function handleInput(event) {
  const input = event.currentTarget;
  const rawValue = input.value;
  const nextValue = formatValue(rawValue);
  const nextCursor = getNextCursor(rawValue, nextValue, input.selectionStart);

  input.value = nextValue;

  requestAnimationFrame(() => {
    if (document.activeElement !== input) return;

    input.setSelectionRange(nextCursor, nextCursor);
  });
}
```

这里有两个关键点：

1. `input` 来自当前用户事件，而不是从全局列表里取。
2. 真正调用 `setSelectionRange` 前，再次检查 `document.activeElement === input`。

## 不要在初始化阶段恢复 selection

很多时候，页面初始化时并不需要设置光标。用户还没有编辑输入框，光标位置没有实际意义。

所以这类代码要避免：

```js
mounted(() => {
  inputs.forEach(input => {
    input.setSelectionRange(0, input.value.length);
  });
});
```

更好的方式是：只保存每个输入框的 selection 状态，在它真正获得焦点或者发生输入时再恢复。

```js
function restoreWhenFocused(input, start, end) {
  input.addEventListener('focus', () => {
    requestAnimationFrame(() => {
      if (document.activeElement !== input) return;

      input.setSelectionRange(start, end);
    });
  });
}
```

## 处理中文输入法组合态

如果输入框涉及中文输入法，还要避开 `compositionstart` 到 `compositionend` 之间的输入组合阶段。

```js
let composing = false;

input.addEventListener('compositionstart', () => {
  composing = true;
});

input.addEventListener('compositionend', event => {
  composing = false;
  handleInput(event);
});

input.addEventListener('input', event => {
  if (composing) return;

  handleInput(event);
});
```

组合输入过程中频繁改 value 和 selection，本来就容易导致候选词、光标和实际输入状态不一致。在 iOS Safari 上更应该谨慎。

# 推荐封装

可以把判断收敛到一个工具函数里：

```js
function restoreActiveInputSelection(input, start, end, direction = 'none') {
  if (!input) return false;
  if (document.activeElement !== input) return false;

  try {
    input.setSelectionRange(start, end, direction);
    return true;
  } catch (error) {
    return false;
  }
}
```

使用时只对当前输入框调用：

```js
function handleInput(event) {
  const input = event.currentTarget;
  const cursor = calculateCursor(input.value, input.selectionStart);

  requestAnimationFrame(() => {
    restoreActiveInputSelection(input, cursor, cursor);
  });
}
```

这个封装不会解决所有光标问题，但可以避免最危险的一类问题：**批量调用 `setSelectionRange` 导致非目标输入框抢焦点。**

# 排查建议

遇到类似问题时，可以先打这些日志：

```js
document.addEventListener('focusin', event => {
  console.log('focusin:', event.target);
});

document.addEventListener('focusout', event => {
  console.log('focusout:', event.target);
});

function logActive(label) {
  console.log(label, document.activeElement);
}
```

然后在每次调用 `setSelectionRange` 前后打印当前 `activeElement`：

```js
logActive('before setSelectionRange');
input.setSelectionRange(start, end);
logActive('after setSelectionRange');
```

如果发现 `setSelectionRange` 后 `activeElement` 变了，就说明当前环境存在隐式聚焦行为。

# 总结

这次问题的核心不是 `setSelectionRange` 本身不能用，而是它不能被无差别地批量调用。

在 iOS 16.4 Safari 上，对多个非当前输入框连续调用 `setSelectionRange`，会导致输入框依次抢占焦点，最后一个输入框成为最终焦点，并触发键盘弹出。

比较稳妥的规则是：

1. 不要在初始化、渲染、批量更新阶段调用所有输入框的 `setSelectionRange`。
2. 只对当前 `document.activeElement` 调用 `setSelectionRange`。
3. 用户输入后的光标恢复，要绑定到当前事件目标。
4. 调用前再次判断焦点，尤其是在 `requestAnimationFrame`、`setTimeout`、框架 `nextTick` 之后。
5. 涉及中文输入法时，避开 composition 过程中强制修改 selection。

一句话总结：**移动端恢复光标时，先确认输入框仍然是当前焦点，再调用 `setSelectionRange`。**
