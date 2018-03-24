## JS做动画小Tips

### 原理

动画其实都是通过一帧一帧绘制展现出来的。

当页面刷新在60帧的时候，人眼其实分辨不出这个刷新的过程，会以为这是一个连续的过程。（比如说人眼看不出显示屏闪动，但实际上用相机的拍摄屏幕的时候，会发现屏幕其实在闪动）

所谓的60帧其实就是1s内执行60次，也就是没间隔16.67ms就要进行一次周期性的画面改变，才能“骗过”人眼，让人眼误认为这是一个连续的过程。

所以同样的，这也就是实现动画的原理。如果我们能够在每个16.67ms都去改变物体的位置，当浏览器每帧绘制的时候，就有看到所谓的“动画”。

一般都是用`setTImeout`或`setInterval`来做动画，但是其实由于JS的异步队列，设置的时间间隔并不能精确到具体数值，只能是尽量靠近。

并且其实每个浏览器每帧重新绘制的时间并不一样，有的大于16.67ms，有的小于它。

用`setTimeout`和`setInterval`来做动画，如果时间间隔小于每帧重新绘制的时间间隔，会出现跳帧的情况。也就是说出现的某一帧的动画状态被忽略了，它直接从一个状态跳到另外一个状态。同样的，如果设置的时间间隔大于每帧重新绘制的事件间隔（或者出现线程阻塞），就会给人一种卡顿的感觉。

使用`requestAnimationFrame`而不是`setTimeout`和`setInterval`，保证动画不会出现跳帧或掉帧。

`requestAnimationFrame`会根据不同类型的浏览器，让传入的回调函数在重新绘制画面的时候执行一次，来保证动画的正常运作。

## 函数的计算

建立`S-t`图像，用函数来计算位移。
```
S'(t) --> v(t)
v'(t) --> a(t) 
a'(t) --> f(t) 若求导得结果在范围内恒大于0，a递增；反之a递减。判断v是加速还是减速，还得看a(t)原本符号。
f'(t) --> 若求导得结果在范围内恒大于0，说明a递增，反之递减。
```
### 匀速运动

设总运动时间为T，S(t)为任意时间点的位移，p为t和T的比值。

```javascript
t / T = p
S(t) = vt
v = s(t) / t = S / T //因为是匀速
S(t) = S / T * t = S * p
```

### 匀加速运动

```javascript
t / T = p
S(t) = 1/2 * a * t^2
S = 1/2 * a * T^2 => a = 2S / T^2
S(t) = 1/2 * 2 * S / T^2 * t^2 = S * p^2
```

### 匀减速

设初始速度为v0。

```javascript
t / T = p
S(t) = v0t + 1/2 * a * t^2 (2)
-1/2 * a * T^2 = S (3)
v0 = -aT (4)
(3)(4)代入(2)得 S(t) = 2 * S * t/T = S * t^2 / T^2
(1)代入(4)得 S(t) = S * (2 * p - p^2);
```

### 变加速运动（加速度不断增大的加速运动）

使用幂函数`S = t ^ 3, t ∈ (0, 1)`

### 变减速运动（加速度不断增大的减速运动）

使用正弦函数`S = sin(t), t ∈ (0, π/2)`

### 变减速运动（加速度不断减小的减速运动）

使用对数函数`ln(t), t ∈ (1, e)`

### easeInOut（先加速后减速）

把位移分隔成两段，前半段加速，后半段减速。

因为前半段位移距离可能小于原位移的一半，所以后半段位移应该是总位移减去前半段真正走的位移。

## 代码实现

```javascript
function speedFn(node, exp, distance, direction, duration) {
    node.addEventListener('click', function () {
        let self = this;
        let startTime = new Date();
        var T = duration;
        requestAnimationFrame(function step() {
            var p = Math.min(1.0, (Date.now() - startTime) / T);
            self.style.transform = (direction === 'x' ? 'translateX(' : 'translateY(') + exp.call(null, distance, p) + 'px)';
            if (p < 1.0) {
                requestAnimationFrame(step);
            }
        });
    })
}
```

通过时间点和总时间的比例得到`p`，结合函数计算出对应的位移S(t)。

变量说明如下：

```javascript
node // 传入的node节点
exp // 使用计算的函数
distance // 总变化量，也就是位移
duration // 持续时间
```

`easeInOut`函数实现如下：

```javascript
function speedEaseFn(node, exp, distance, direction, duration) {
    node.addEventListener('click', function () {
        let self = this;
        let startTime = new Date();
        let T = duration;
        let p = null;
        let tagDistance = null;
        requestAnimationFrame(function step() {
            if (Date.now() - startTime <= T / 2) {
                p = Math.min(1.0, (Date.now() - startTime) / (T / 2));
                tagDistance = exp[0].call(null, distance / 2, p);
                self.style.transform = (direction === 'x' ? 'translateX(' : 'translateY(') + tagDistance + 'px)';
            } else {
                p = Math.min(1.0, (Date.now() - startTime) / (T / 2) - 1);
                self.style.transform = (direction === 'x' ? 'translateX(' : 'translateY(') + (tagDistance + exp[1].call(null, distance - tagDistance, p)) + 'px)';
            }
            if (Date.now() - startTime < T) {
                requestAnimationFrame(step);
            }
        });
    })
}
```
现在分成两段来执行，`exp`需要传入一个长度为2的数组，分别是加速和减速函数。

## 此外

### 关于样式的获取

如果CSS样式不是直接卸载标签的`style`属性中，是无法通过`dom.style.xxx`进行访问的。

此时必须需用全局函数`getComputedStyle(node)[attr]`来访问，这个属性只是**只读**的，它会计算相关DOM最终的样式。

IE下要使用`node.currentStyle[attr]`来访问。

