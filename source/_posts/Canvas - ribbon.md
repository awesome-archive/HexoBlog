---
title: Canvas - ribbon
date: 2017-07-07 14：30
tags: ['Canvas', 'JavaScript']
categories: Canvas
---

这个效果第一次遇见还是在尤雨溪的 blog 上看到的，给人感觉非常好。
有种低调的装了逼的感觉 😶

<iframe width="100%" height="300" src="//jsfiddle.net/swarosky44/e6v17ohr/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

<!-- more -->
## 基本原理
在 codeopen 看到 ribbon 的代码，第一眼看到其实是一脸懵逼的，但是仔细想一下还是蛮容易的。
可以通过修改一下代码就能够立马理解啦 ~
我们把 `ctx.fillStyle` & `ctx.fill` 都改成 stroke 就明白原理了。
其实本质就是绘制一个有一定规律的折线，然后按照一个规律填充颜色就可以实现了。

## 实现
### 参数准备
```JavaScript
const pr = window.devicePixelRatio || 1;
const W = canvas.width = window.innerWidth * pr;
const H = canvas.height = window.innerHeight * pr;
const Hp = 0.7
const chartH = 90;
const pi = Math.PI * 2;
const cos = Math.cos;
let radius = 0;
let p;
```
- pr：当前显示设备的物理像素分辨率与CSS像素分辨率的比值;
- W、H：canvas 画布的宽高；
- Hp：彩带在大约在画布 Y 轴的位置（用百分比计算）；
- chartH：彩带高度的 1 / 2；
- pi：360 度的圆周值；
- cos：余弦函数的计算方法；
- radius：角度值；
- p：存放折现拐点的容器；

### 创建
```JavaScript
function create() {
  ctx.clearRect(0, 0, W, H);
  p = [
    { x: 0, y: H * Hp + chartH },
    { x: 0, y: H * Hp - chartH },
  ];
  while(p[1].x < W) {
    draw(p[0], p[1]);
  }
}
```

这里我们创建了两个初始点，都是 X 轴 0 点开始，高度分别是画布中绘制点的上下固定距离。
随后通过迭代，我们开始绘制，这里明显是分段式的绘制，直到绘制的拐点到了画布的边缘。

### 绘制
```JavaScript
function draw(p0, p1) {
  ctx.beginPath();
  ctx.moveTo(p0.x, p0.y);
  ctx.lineTo(p1.x, p1.y);
  const nx = calculateX(p1.x);
  const ny = calculateY(p1.y);
  ctx.lineTo(nx, ny);
  ctx.closePath();
  radius -= pi / -50;
  const color = '#' + (cos(radius) * 127 + 128 << 16 | cos(radius + pi / 3) * 127 + 128 << 8 | cos(radius + pi / 3 * 2) * 127 + 128).toString(16);
  ctx.fillStyle = color;
  ctx.fill();
  p[0] = p[1];
  p[1] = {
    x: nx,
    y: ny,
  };
}

function calculateX(x) {
  return x + (Math.random() * 2 - 0.25) * chartH;
}

function calculateY(y) {
  const cy = y + (Math.random() * 2 - 1.1) * chartH;
  return (cy > H || cy < 0) ? calculateY(y) : cy;
}
```
分段式的绘制过程中，我们用 p0 代表上一个拐点，p1 代表当前的拐点，然后绘制的过程中计算下一个拐点的位置。
这里我们仔细看一下图就明白了其实这个 ribbon 就是由多个三角形组成，所以每个分段中都需要绘制三个点。fill 时会连接第一个点和最后一个点。

至于 X 、Y 的计算其实可以自己随意点，不必太过纠结这里给出的公式。

最后就是颜色值的公式，这里看上去非常复杂。其实主要是通过线性计算 radius 的值，然后在将值作用于余弦函数上，就可以看到一个渐变的效果。
接着是 style 的表示方式，这里是用的 web 格式，web 格式的颜色值，其实是 16进制的三个数字组成的，分别代表了三原色。同理我们也可以用 rgb 的格式来计算和表示，我认为会相应的简单很多。

## 总结
其实对于平常看到的很多酷炫效果，都可以去 codeopen 搜一下，看上去非常复杂的东西，也许并没有我们想的那么复杂。
