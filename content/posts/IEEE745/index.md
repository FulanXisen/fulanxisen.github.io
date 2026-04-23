---
title: "IEEE 745 floating point representation"
date: 2022-07-22T10:44:05+08:00
draft: false
math: true
---

32-bit 表示一个单精度浮点数 float

$V=(-1)^s \times M \times 2^E$
{{< figure src="float.png" title="Bit Segments" >}}

32-bit 分为3部分： S, E, M

- S: 符号位，1bit，0 表示正数， -1 表示负数
- E: 指数位，8bit，取值[-127, 128].
- M: 尾数位，23bit，取值[0, $2^{23}-1$].

指数的底固定为2，指数有正数也有负数，但指数部分没有符号位，取一个中间值，小于中间值的表示负数，等于中间值的表示0，大于中间值的表示正数，中间值的定义如下：Bias=$2^{k-1}$-1, k表示E的bit数。实际的E为E-Bias.

尾数部分隐藏了小数点前的1，实际为1.M。M转换为十进制从高位开始计算： $1+b_{22}\times\frac{1}{2}+b_{21}\times\frac{1}{4}+...+b_{0}\times\frac{1}{2^{23}}$


规格化的表示，E不是全0，也不是全1，M任意取值

E全为0时，表示+0.0, -0.0和接近0.0的值

E全为1，M全为0时，表示无穷，符号位区分正无穷和负无穷

E全为1，M不全为0时，表示NaN，不是一个数。