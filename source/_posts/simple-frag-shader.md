---
title: 基础的片元着色器技巧
date: 2018/11/6
---

整理一些通用的非常非常基础的片元着色器技巧。

从来没有接触过片元着色器的朋友可以去看看thebookofshaders这个网站，非常不错//。顺便，它有一个交互的编辑器日常可以玩玩。 http://editor.thebookofshaders.com/SLWRZ/ (maybe需要科学上网).  。

### 随机性

一般来说shader里高效的生成随机数，这个随机性一般来自对一个超高频的函数进行采样。创造一个超高频的函数很简单，首先需要有周期性，所以三角函数一般是需要的，你可以把一个sin，振幅给的非常大，然后取小数，这就是一般shader里随机数生成的常用做法，我自己尝试了直接用一个超高频的sin，效果也不错，但可能再结合其他函数还是会暴露周期性，所以可能有时候不适用。

```c
float rand(float n){return fract(sin(n) * 43758.5453123);}
float rand(float n){return (sin(n * 1234567.0));}
```

二维随机数: 和一维的一样，需要把二维降到一维，这里可以用一个点积。因为点积的结果，在xy两个轴分别看上去，其实线性的，另外一个维度就是截距的不同，所以可以直接用一维的方法，并不会在某个方向产生不均匀的问题（假设你一维的方法能够保证随机性）。

```c
float rand(vec2 n) { 
	return fract(sin(dot(n, vec2(12.9898, 4.1414))) * 43758.5453);
}
```

有时候为了更好的质量，可以再乘一个dot过的sin在里边。

### 简单噪声

噪声同时包含了随机性的连续性。纯粹的随机是完全离散的，除了概率毫无规律的，跳跃的，当我们试图在一个规整的，几何的形态上，引入随机性的因素，可以创造出有机的自然的形态。下面这个basic的noise，是在整数位上做随机，然后中间直接插值。

```c
float noise(float p){
	float fl = floor(p);
    float fc = fract(p);
	return mix(rand(fl), rand(fl + 1.0), smoothstep(0.,1.,fc));
}

float noise(vec2 n) {
	const vec2 d = vec2(0.0, 1.0);
    vec2 b = floor(n);
    vec2 f = smoothstep(vec2(0.0), vec2(1.0), fract(n));
	return mix(mix(rand(b), rand(b + d.yx), f.x), 
               mix(rand(b + d.xy), rand(b + d.yy), f.x)
               , f.y);
}

float noise(vec2 p){
	vec2 ip = floor(p);
	vec2 u = fract(p);
	u = u*u*(3.0-2.0*u);
	
	float res = mix(
		mix(rand(ip),rand(ip+vec2(1.0,0.0)),u.x),
		mix(rand(ip+vec2(0.0,1.0)),rand(ip+vec2(1.0,1.0)),u.x),u.y);
	return res*res;
}
```

一般这种simple value noise的patten是，floor你的变量，拿到离散的整数位，fract你的变量，拿到一个单位跨度的连续增长，在离散的整数位上做随机，然后用这个整数位间的连续量做插值，插值的效果，比如过渡的方法，通过影射插值连续量实现，比如smooth，比如上面第二个noise的多项式。

> value noise一般非常块状，为了消除这种块状的效果，在 1985 年 [Ken Perlin](https://mrl.nyu.edu/~perlin/) 开发了另一种 noise 算法 Gradient Noise。Ken 解决了如何插入随机的 gradients（梯度、渐变）而不是一个固定值。这些梯度值来自于一个二维的随机函数，返回一个方向（vec2 格式的向量），而不仅是一个值（float格式）。

各种高级的noise可以参考 [https://gist.github.com/patriciogonzalezvivo/670c22f3966e662d2f83](https://gist.github.com/patriciogonzalezvivo/670c22f3966e662d2f83)

### 平移旋转缩放

这个没有什么好说的，把matrix乘到归一化的coord上就可以, 比如旋转。

```c
mat2 rotate2d(float _angle){
    return mat2(cos(_angle),-sin(_angle),
                sin(_angle),cos(_angle));
}

vec2 st = gl_FragCoord.xy/u_resolution.xy;
st = rotate2d( PI ) * st;
```



### 硬边缘和重复性

典型的重复例如平铺，可以通过放大，然后取小数，重新映射回1-0的技巧实现。

```c
	vec2 st = gl_FragCoord.xy/u_resolution;
    vec3 color = vec3(0.0);

    st *= 3.0;      // Scale up the space by 3
    st = fract(st); // Wrap arround 1.0

    // Now we have 3 spaces that goes from 0-1
```

提取奇偶项或者整数倍项的做法：利用取余

```c
fract(x) === mod(x,1.0) // 事实上可以替代的
mod(x,2.0) < 1.0 ? 0. : 1. ; //偶数项
```

一般如果需要生成几何类型的patten，往往是利用step或者mod判断来做出分段，patten上的任何分段，分块之类的硬边缘，几乎都是shader里的条件判断产生的。如何设计出通用的shader函数，和组合的模式，来总结出一套patten的生成方法，似乎是有趣而充满挑战的事情。

