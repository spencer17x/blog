---
title: Excel 小数计算精度问题
date: 2026-06-20 13:45:00
tags:
    - Front-end
---

# 背景

在做金额、折扣、税率、比例这类计算时，我们经常会默认认为：

```txt
0.1 + 0.2 - 0.3 = 0
19.99 * 100 = 1999
ROUND(1.255, 2) = 1.26
```

但是在 Excel 里，这些表达式并不总是符合“十进制心智模型”。问题通常不是 Excel 把公式写错了，而是 Excel 和大多数编程语言一样，底层使用有限精度的二进制浮点数保存和计算数字。

微软文档里也明确提到，Excel 围绕 IEEE 754 浮点规范设计，部分数字和公式会受到舍入或数据截断影响；Excel 的数字精度也限制在 15 位有效数字左右。

这篇文章记录一次复现：用 Spreadsheets 插件生成 `.xlsx` 测试文件，并使用它的 Excel 兼容公式引擎跑一组小数计算，再用 `decimal.js` 按十进制语义跑同一组计算，看看差异具体出现在哪里。

需要先说明测试边界：本文截图和表格里的计算结果来自 Spreadsheets 插件的公式引擎，不是 Microsoft Excel 桌面版或网页版的直接截图，所以这里不会把结果写成“某个 Excel 版本实测”。不过微软官方文档已经说明 Excel 使用 IEEE 754 浮点模型，且小数计算可能因为舍入或数据截断得到不符合直觉的结果；本文的复现结果也正好落在这个问题范畴里。

# 复现文件

我用 Spreadsheets 插件生成了一个测试工作簿：

```txt
/files/excel-decimal-precision/excel-decimal-precision-repro.xlsx
```

工作簿里有三类结果：

1. Spreadsheets 插件公式引擎计算值。
2. 直觉上期望得到的结果。
3. `decimal.js` 按十进制语义得到的结果。

截图如下：

![Excel 小数计算精度复现结果](/articleImgs/Excel-小数计算精度问题/decimal-precision-preview.png)

可以直接下载测试文件：

[excel-decimal-precision-repro.xlsx](/files/excel-decimal-precision/excel-decimal-precision-repro.xlsx)

# 问题复现

本次复现里比较有代表性的几组结果如下。

| 场景 | 公式引擎结果 | decimal.js 结果 | 说明 |
| --- | --- | --- | --- |
| `0.1 + 0.2 - 0.3` | `5.551115123125783e-17` | `0` | 出现非常小的正向残差 |
| `SUM(0.1 x 10) - 1` | `-1.1102230246251565e-16` | `0` | 累加后出现负向残差 |
| `4.1 - 4` | `0.09999999999999964` | `0.1` | 显示为 0.1 不代表底层值精确等于 0.1 |
| `INT(19.99 * 100)` | `1998` | `1999` | 金额转分后取整，结果少了 1 |
| `ROUND(1.255, 2)` | `1.25` | `1.26` | 十进制 half-up 语义下通常期望为 1.26 |
| `ROUND(2.675, 2)` | `2.68` | `2.68` | 不是所有小数都会出现同样问题 |
| `0.3 - 0.2 - 0.1` | `-2.7755575615628914e-17` | `0` | 同一组数字换个计算顺序，也可能出现不同方向的残差 |

其中最危险的不是 `0.1 + 0.2 - 0.3` 这种一眼能看出异常的残差，而是 `INT(19.99 * 100)` 这种业务场景：

```txt
19.99 元 * 100 = 1999 分
```

如果公式计算过程中内部临时值接近：

```txt
1998.9999999999998
```

再执行 `INT`，就会得到 `1998`。

这类问题在金额、积分、库存单位换算、费率结算里都比较危险，因为误差很小，但一旦接上取整、比较、分支判断，就可能变成可见的业务错误。

# decimal.js 对照

`decimal.js` 的定位是任意精度 Decimal 类型。它不会先把字符串形式的十进制小数转换成二进制浮点数再计算，所以更接近业务里常说的“按十进制算”。

对照代码大概是这样：

```js
import Decimal from 'decimal.js';

new Decimal('0.1').plus('0.2').minus('0.3').toString();
// '0'

new Decimal('19.99').times('100').floor().toString();
// '1999'

new Decimal('1.255').toDecimalPlaces(2, Decimal.ROUND_HALF_UP).toString();
// '1.26'
```

这里有个细节很重要：创建 `Decimal` 时，应该传字符串。

```js
new Decimal('1.255')
```

而不是：

```js
new Decimal(1.255)
```

后者在进入 `decimal.js` 之前，`1.255` 这个 JavaScript number 已经是二进制浮点近似值了。虽然 `decimal.js` 可以处理 number，但如果目标是保留十进制输入语义，字符串更稳。

# 原因

十进制里的有限小数，不一定能用二进制有限小数表示。

例如 `0.1` 在十进制里很短，但转成二进制后是循环小数。计算机只能用有限位数保存它，于是只能保存一个近似值。后续再做加减乘除，就可能把这个近似值的尾差带出来。

电子表格软件的行为还多了一层容易迷惑人的地方：显示格式不等于真实参与计算的值。

比如单元格显示：

```txt
0.1
```

它参与后续计算时，底层可能更接近：

```txt
0.09999999999999964
```

当只是展示给人看时，这通常没什么影响；但当后续还有 `INT`、`ROUND`、`IF`、`VLOOKUP`、等值比较、汇总校验时，误差就可能被放大成业务问题。

# 解决方案

## 用 ROUND 固定业务精度

如果计算仍然放在 Excel 表内，最基本的做法是：在关键节点按业务精度显式 `ROUND`。

例如金额转分时，不要直接：

```excel
=INT(19.99*100)
```

可以改成：

```excel
=INT(ROUND(19.99*100,0))
```

对于金额中间值，也应该尽早把结果归一到业务允许的精度：

```excel
=ROUND(A1*B1,2)
```

这不是为了让显示更好看，而是为了让后续计算拿到已经归一化的值。

不过 `ROUND` 不是万能药。像 `ROUND(1.255, 2)` 这种刚好卡在进位边界上的值，Excel 仍然可能因为输入值已经是二进制近似值而得到 `1.25`。测试工作簿里为了演示表内修正，使用了 `1.255 + 1E-12` 这种补偿写法，但这种方式不适合作为通用金额规则。只要涉及严格的十进制舍入，还是应该优先使用 decimal.js 或后端 Decimal 类型。

## 不要用显示格式代替计算修正

把单元格设置成两位小数，只会影响显示，不会改变真实参与计算的值。

比如把 `5.551115123125783e-17` 显示成 `0.00`，它在后续公式里仍然不是数学意义上的 0。

如果后续逻辑依赖这个结果，需要用公式修正：

```excel
=ROUND(A1,2)
```

而不是只设置单元格格式。

## 谨慎使用“以显示精度为准”

Excel 有一个“以显示精度为准”的选项，它会把工作簿里的数值强制改成当前显示精度。

这个选项可以减少一些浮点误差带来的影响，但代价很大：它会永久丢失未显示出来的精度。微软文档也提醒，开启前应保存工作簿副本，因为无法通过关闭选项恢复被丢掉的数据。

所以它更像是最后手段，不适合作为默认修复方案。

## 关键金额计算放到 decimal.js

如果业务逻辑在前端或者 Node.js 里，金额、费率、分摊、税额这类计算更适合放到 `decimal.js` 这类十进制库里。

```js
import Decimal from 'decimal.js';

function yuanToCent(yuan) {
  return new Decimal(yuan)
    .times(100)
    .toDecimalPlaces(0, Decimal.ROUND_HALF_UP)
    .toNumber();
}

yuanToCent('19.99');
// 1999
```

然后再把已经归一化的结果写入 Excel。

这样 Excel 更像是展示、导出、人工核对工具，而不是承担所有关键计算的唯一引擎。

# 排查建议

遇到 Excel 小数计算异常时，可以按这个顺序排查：

1. 把相关单元格格式调成足够多的小数位，先看真实残差。
2. 检查有没有 `INT`、`ROUNDDOWN`、`IF(A=B)`、`VLOOKUP` 精确匹配这类会放大误差的公式。
3. 在关键节点加 `ROUND`，不要只改显示格式。
4. 对金额、费率、分摊结果，用 decimal.js 或后端 Decimal 类型做一组对照。
5. 如果 Excel 只是导出结果，优先导出已经按业务精度归一化后的值。

# 总结

Excel 的小数问题，本质上是二进制浮点数和十进制业务语义之间的错位。

这次复现里，几个结果很有代表性：

```txt
0.1 + 0.2 - 0.3       -> 5.551115123125783e-17
SUM(0.1 x 10) - 1     -> -1.1102230246251565e-16
INT(19.99 * 100)      -> 1998
ROUND(1.255, 2)       -> 1.25
```

如果只是展示，影响通常很小；如果接上金额、取整、比较、校验，这个影响就可能变成真实问题。

我的建议是：

1. Excel 表内计算，用 `ROUND` 在关键节点固定业务精度。
2. 不要把“显示成两位小数”当成“值已经是两位小数”。
3. 金额和费率这类关键逻辑，尽量用 `decimal.js` 或后端 Decimal 类型先算好，再写入 Excel。

# 参考

- [Microsoft Learn: Floating-point arithmetic may give inaccurate results in Excel](https://learn.microsoft.com/en-us/troubleshoot/microsoft-365-apps/excel/floating-point-arithmetic-inaccurate-result)
- [Microsoft Support: Change formula recalculation, iteration, or precision in Excel](https://support.microsoft.com/en-us/excel/change-formula-recalculation-iteration-or-precision-in-excel)
- [Microsoft Support: Set rounding precision](https://support.microsoft.com/en-us/excel/set-rounding-precision)
- [Microsoft Support: Round a number to the decimal places I want in Excel](https://support.microsoft.com/en-us/excel/round-a-number-to-the-decimal-places-i-want-in-excel)
- [decimal.js API](https://mikemcl.github.io/decimal.js/)
