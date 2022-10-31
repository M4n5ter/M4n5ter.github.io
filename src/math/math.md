$\because\ e^y = x \rightarrow y = \ln x \\ \therefore\ e^{\ln x} = x \\ \therefore\ e^{\ln x^x} = x^x $

$\huge (uv)^{(n)} = \Sigma^n_{k=0} C^k_n u^{(n-k)}v^{(k)}$

$\forall 表示任取$

# 等价无穷小替换

$\lim\limits_{x\to0}\frac{\tan x - x}{x^2\sin x} = \lim\limits_{x\to0}\frac{\tan x -x}{x^3}$

此处 $\sin x$ 可以替换为 $x$，$\tan x$ 不可替换为 $x$。

因为等价无穷小替换的前提是分子或分母是一些项的乘积，而这里分子不是一些项的乘积，所以分子处不能使用等价无穷小替换；而分母满足条件，可以使用等价无穷小替换

# 微分公式

$$
\huge dy = f'(x)dx
$$

# 复合函数的微分

$$
\huge y = f(u), \, u = g(x) \,\Rightarrow du=g'(x)dx
\\
dy = y'_xdx = f'(u)g'(x)dx = f'(u)du \\
\Downarrow \\ dy = f'(u)du
$$

> 从上面俩可以得出 **“微分形式不变性”**

# 微分在近似计算中的应用

$\Delta y = f(x_0+\Delta x) - f(x_0)$   **精确值**

$dy = f'(x_0)\Delta x$                          **近似值**

$\Delta y \approx dy$ 

$\therefore f(x_0 + \Delta x) \approx f'(x_0)\Delta x + f(x_0)$  

> 例：半径 1 cm 球，镀铜 0.01 cm，求镀的铜的体积

$$
V = \frac{4}{3}\pi r^3, \,\,\,\, r_0 = 1, \Delta r = 0.01 \\
V' = 4\pi r^2 
$$

目的是求 $\Delta V$,  $\Delta V$ 是精确值。

$\Delta V \approx 4\pi r^2|_{r=1}\Delta r = 4\pi \times 0.01 = 0.04\pi \approx 0.13(cm^3)$

要求质量的话再乘以密度即可。

## 近似计算中的常用公式

$\LARGE x \to 0 $
$\normalsize (1)\,\,\,\, \Large (1+x)^\alpha \approx 1 + \alpha x$
$\normalsize (2)\,\,\,\, \Large sinx \approx x$

$\normalsize (3)\,\,\,\, \Large tanx \approx x$

$\normalsize (4)\,\,\,\, \Large e^x \approx 1 + x $

$\normalsize (5)\,\,\,\, \Large \ln(1+x) \approx x$

**意义：**

**1. 左边的失去计算器没一个好算的，但是右边的都好算。**

**2. 精确度高**

# 微分中值定理

## 拉格朗日中值定理

> 1. $[a,b]\,\,连续$
> 
> 2. $(a,b)\,\,可导$
> 
> $(a,b)\,\,$至少有一点 $\xi$ 满足:
> 
> $f(b) - f(a) = f'(\xi)(b-a) \Rightarrow f'(\xi) = \frac{f(b)-f(a)}{b-a}$

## 罗尔中值定理

> 1. $[a,b]\,\,连续$
> 
> 2. $(a,b)\,\,可导$
> 
> 3. $f(a) = f(b)$
> 
> 则至少有一个$\xi \in (a,b). f'(\xi) = 0$

罗尔中值定理是拉格朗日中值定理的一种特殊情况。

从以上可以再推出一个定理：

$$
\large f(x) 在区间\, I \,连续且可导,同时导数恒为\,0\,,f(x) = C\,(C\, 为常数)
$$

## 柯西中值定理

> $若有\,f(x)\,和\,F(x)$
> 
> 1. $[a,b]\,\,连续$
> 
> 2. $(a,b)\,\,可导$
> 
> 3. $\forall x \in (a,b) 且 F'(x) \neq 0$
> 
> $至少有一点 \xi 满足\,\frac{f(b)-f(a)}{F(b)-F(a)} = \frac{f'(\xi)}{F'(\xi)}$

**柯西中值定理是这几个中值定理里面最一般化的。**

**柯西中值定理中将 $F(x)=x$ 时即可推出拉格朗日中值定理**

**拉格朗日中值定理中将 $f(a)=f(b)$ 即可推出罗尔中值定理**

# 洛必达法则

> 1. $x\to a时, f(x)\to 0\,\,F(x)\to 0$
> 
> 2. $在\,a\,的去心邻域内\,f'(x)\,\,F'(x)\,存在且\,F'(x)\neq 0$
> 
> 3. $\lim\limits_{x\to a}\frac{f'(x)}{F'(x)}存在(或\infty)$
> 
> $则\,\,\lim\limits_{x\to a}\frac{f(x)}{F(x)} = \lim\limits_{x\to a}\frac{f'(x)}{F'(x)}$

> 1. $x\to \infty时, f(x)\to 0,,F(x)\to 0$
> 
> 2. $在\,a\,的去心邻域内\,f'(x)\,\,F'(x)\,存在且\,F'(x)\neq 0$
> 
> 3. $\lim\limits_{x\to a}\frac{f'(x)}{F'(x)}存在(或\infty)$
> 
> $则\,\,\lim\limits_{x\to a}\frac{f(x)}{F(x)} = \lim\limits_{x\to a}\frac{f'(x)}{F'(x)}$

$\frac{0}{0},\,\frac{\infty}{\infty},\,0\cdot\infty,\infty-\infty,\, 0^0,\,1^\infty,\,\infty^0$

**上面这几种情况都可以通过转换成 $\frac{0}{0}或\frac{\infty}{\infty}$ 的形式来使用洛必达法则**

例如:

$0\cdot\infty的情况:$

$\large \lim\limits_{x\to0^+}x^n \ln x\,(n>0) = \lim\limits_{x\to0^+}\frac{lnx}{\frac{1}{x^n}} = \lim\limits_{x\to0^+}\frac{\frac{1}{x}}{-n\cdot x^{-n-1}} = \lim\limits_{x\to0^+}-\frac{1}{nx^{-n}} = \lim\limits_{x\to0^+}-\frac{x^n}{n} = 0$

**2 种技巧：**

* 等价无穷小替换

* 适当的将项(**常数或者趋于常数**)朝外挪

比如$\large\lim\limits_{x\to0}\frac{x^2-\tan x}{\cos x\sin x}中\,\cos x 趋于\,0，所以可以转换成\,\,1\cdot \lim\limits_{x\to0}\frac{x^2-\tan x}{\sin x}$

# 泰勒公式

$$
f(x)\,在\,x_0\,处有\,n\,阶导，\exist x_0的一个邻域,则有:\\
f(x)=f(x_0) + \frac{f'(x_0)}{1!}(x-x_0) + \frac{f''(x_0)}{2!}(x-x_0)^2 +\ldots+\frac{f^{(n)}(x_0)}{n!}(x-x_0)^n +R_n(x)\\
Rn(x) = \circ(\small{(x-x_0)^n})，即是高阶无穷小 \\
Rn(x) = \frac{f^{(n+1)}(\xi)}{(n+1)!}(x-x_0)^{n+1}, \xi\,介于\,x_0\,与\,x\,之间 
$$

# 麦克劳林公式

一般用的比较多的是泰勒公式在 $x_0 = 0$ 处展开,那么就有了麦克劳林公式：

$$
x_0 = 0,\,f(x) = f(0) + \frac{f'(0)}{1!}x + \frac{f''(0)}{2!}x^2+\ldots+
\frac{f^{(n)}(0)}{n!}x^n + \frac{f^{(n+1)}(\theta x)}{(n+1)!}x^{n+1}\,,\\
0<\theta<1
$$

## 用麦克劳林公式解释等价无穷小

> 解释 $e^x-1 \sim x$

$$
f(x)=e^x,f'(x)=f''(x)=\ldots=f^{(n)}(x)=e^x,f^{(n+1)}(\theta x) = e^{\theta x}\\

\because e^x=1+x+\frac{1}{2!}x^2+\frac{1}{3!}x^3 +\ldots+\frac{1}{n!}x^n+\frac{e^{\theta x}}{(n+1)!}x^{n+1},
0<\theta<1\\
e^x \approx 1+x+\frac{1}{2!}x^2+\frac{1}{3!}x^3 +\ldots+\frac{1}{n!}x^n\\

\therefore 当\,x\to0\,时，x^2\,以及更高次可以忽略不计，就有了\\
e^x\sim1+x \Rightarrow e^x-1 \sim x
$$

# 不定积分

$$
\Large \int f(x)dx = F(x) + C \\
f(x)\,为被积函数，dx\,为积分变量。
$$

$$
F'(x) = f(x)\\
F(x)为原函数,f(x)为导函数
$$

求谁的原函数，谁就是被积函数。

## 积分表/求导表

| 积分表                                                             | 求导表                                   |
| --------------------------------------------------------------- | ------------------------------------- |
| $\int kdx=kx+C$                                                 | $(kx)'=k$                             |
| $\int x^udx=\frac{x^{u+1}}{u+1}+C$                              | $(x^u)'=ux^{u-1}$                     |
| $\int\frac{1}{x}dx=\ln\|x\|+C$                                  | $(\ln x)'=\frac{1}{x}$                |
| $\int\frac{dx}{1+x^2}=\arctan x+C$                              | $x>0,(\arctan x)'=\frac{1}{1+x^2}$    |
| $\int\frac{dx}{\sqrt{1-x^2}}=\arcsin x+C$                       | $(\arcsin x)'=\frac{1}{\sqrt{1-x^2}}$ |
| $\int\cos xdx=\sin x+C$                                         | $(\sin x)'=\cos x$                    |
| $\int\sin xdx=-\cos x+C$                                        | $(\cos x)'=-\sin x$                   |
| $\int \tan xdx=-\ln\|\cos x\|+C$                                |                                       |
| $\int \cot xdx=\ln\|\sin x\|+C$                                 |                                       |
| $\int \csc xdx=\ln\|\tan\frac{x}{2}\|+C=\ln\|\csc x-\cot x\|+C$ |                                       |
| $\int\sec xdx=\ln\|\sec x+\tan x\|+V$                           |                                       |
| $\int\frac{dx}{a^2+x^2}=\frac{1}{a}\arctan\frac{x}{a}+C$        |                                       |
| $\int\frac{dx}{x^2-a^2}=\frac{1}{2a}\ln\|\frac{x-a}{x+a}\|+C$   |                                       |
| $\int\frac{dx}{\cos^2x}=\int\sec^2xdx=\tan x+C$                 | $(\tan x)'=\frac1{\cos^2x}$           |
| $\int\frac{dx}{\sin^2x}=\int\csc^2dx=-\cot x+C$                 | $(\cot x)'=-(\csc^2x)$                |
| $\int\sec x\tan xdx=\sec x+C$                                   | $(\sec x)'=\sec x\tan x$              |
| $\int\csc^2\cot x dx=-\csc x+C$                                 | $(\csc x)' = -\csc x\cot x$           |
| $\int e^xdx=e^x+C$                                              | $(e^x)'=e^x$                          |
| $\int a^xdx=\frac{a^x}{\ln a}+C$                                | $(a^x)'=a^x\ln a$                     |

## 第二类换元积分法一般的替换模式

1. $\sqrt{a^2-x^2}\,\,\,\,x=a\sin t\,(此处都需要加上t取值范围)\,\,\,\,\sqrt{a^2-a^2\sin^2t}=\sqrt{a^2\cos^2t}$

2. $\sqrt{x^2-a^2}\,\,\,\,x=a\sec t\,\,\,\,\sqrt{a^2\sec^2t-a^2}=\sqrt{a^2\tan^2t}$

3. $\sqrt{x^2+a^2}\,\,\,\,x=a\tan t\,\,\,\,\sqrt{a^2\tan^2t+a^2}=\sqrt{a^2\sec^2t}$

# 分部积分法

$$
\int udv=uv-\int vdu
$$

向 $d$ 后面拿的优先级：

1. $e^x$

2. $\sin \cos$

3. $x^n$

用分部积分做题很少有一次分部积分公式就搞定的，基本上要2次及以上。

**用分部积分法解题的常见现象，要求的东西分部积分求着求着又出现了：**

$$
看系数\begin{cases}
+1 说明做错了 \Rightarrow检查\\
其它 \Rightarrow基本没问题，把左右两边一样的合并
\end{cases}
$$

> 使用分部积分法求 $\ln x原函数$

$\int\ln xdx=x\ln x-\int xd\ln x=x\ln x-\int1dx=x\ln x-x+C$

# 微积分基本公式

$$
[\int^{\phi(x)}_{\psi(x)}f(t)dt]'=f({\phi(x)})\phi'(x)-f({\psi(x)})\psi'(x)
$$

# 定积分基本公式

## 牛顿-莱布尼茨公式

$$
F(x)是f(x)的原函数。\\
\int^b_af(x)dx=F(x)|^b_a=F(b)-F(a)
$$

## 定积分换元法

**定理：**

$$
\begin{align*}
&x=\phi(t) \\
&1)\,\,\,\,\phi(\alpha)=a,\phi(\beta)=b \\
&\int_a^bf(x)dx=\int_\alpha^\beta f({\phi(t)})\phi(t)dt
\end{align*}
$$

1. 引入换元函数

2. 上下限也跟变

**例题：**

$$
\begin{align*}
&1)\int_0^a\sqrt{a^2-x^2}\,dx\,\,(a>0)\\
&解:\\
&x=a\sin t,\,dx=a\cos t\,dt,\,x=0\to t=0,\,x=a\to t=\frac{\pi}{2}\\
&\therefore \int_0^a\sqrt{a^2-x^2}\,dx\\
&=\int_0^{\frac{\pi}{2}}\sqrt{a^2-a^2\sin^2t}
\,a\cos t\,dt\\&=\int_0^{\frac{\pi}{2}}a^2\cos^2t\,dt\\
&=\frac{1}{2}\int_0^{\frac{\pi}{2}}a^2+a^2\cos 2t\,dt\\
&=\frac{\pi}{4}a^2
\end{align*}
$$

> 特殊结论

$$
\begin{align*}
&[-a,a]\,\,\,\,f(x)为偶函数时，\int_{-a}^af(x)dx=2\int_0^af(x)dx\\

&[-a,a]\,\,\,\,f(x)为奇函数时，\int_{-a}^af(x)dx=0
\end{align*}
$$

# 一阶线性微分方程

$$
\frac{dy}{dx} + P(x)y = Q(x) \\
(Q(x)=0表示齐次)\\
齐次、非齐次通用公式：\\
\huge\Downarrow\\
\huge y=e^{-\int P(x)dx}(\int Q(x)e^{\int P(x)dx}dx+C)
$$

# 常系数线性齐次微分方程

## 二阶常系数线性齐次微分方程

$$
\begin{align*}
&y'' + py' + qy = 0 \\
&y称为其通解 \\
& 做法是将其转化为特征方程 \Longrightarrow r^2 + pr + q =0 \\
&\begin{cases}
① \Delta=p^2-4q > 0,r_1=\frac{-p+\sqrt{\Delta}}{2}, r_2=\frac{-p-\sqrt{\Delta}}{2}, 
&y=C_1e^{r_1x} + C_2e^{r_2x}\\
② \Delta=p^2-4q = 0,r_1=r_2=\frac{-p}{2}, &y=(C_1+C_2x)e^{r_1x} \\
③ \Delta=p^2-4q < 0,r_1=\alpha+\beta i,r_2=\alpha-\beta i, &y=e^{\alpha x}
(C_1\cos \beta x + C_2\sin \beta x)
\end{cases}

\end{align*}
$$

# 无穷级数

## 常数项级数

> 等比（几何）级数

$$
\large a+aq+aq^2+aq^3+\cdot\cdot\cdot+aq^{n-1}+\cdot\cdot\cdot
$$

$$
\begin{align*}
&S_n为前n项和 \\

&①|q|\neq 1 \,\,\,\, S_n=\frac{a(1-q^n)}{1-q}
\begin{cases}
|q|<1 \lim\limits_{n\to\infty}S_n=\frac{a}{1-q} \\
|q|>1 \lim\limits_{n\to\infty}S_n发散\\
\end{cases} \\
&②|q|=1
\begin{cases}
q=1 发散\\
q=-1 发散\\
\end{cases}\\

&结论: 
\begin{cases}
|q|<1 收敛\\
|q|\geq1发散
\end{cases}
\end{align*}
$$

> 性质1)
> 
> $$
> \large \sum^\infty_{n=1}U_n收敛于S, \sum^\infty_{n=1}kU_n收敛于kS
> $$

> 性质2)
> 
> $$
> \large \sum^\infty_{n=1}U_n和\sum^\infty_{n=1}V_n分别收敛于 S 和\sigma.\\
\sum^\infty_{n=1}(U_n\pm V_n)也收敛.和S\pm\sigma
> $$



正推可以，反推不可以。

> 性质3)
> 
> $$
> 去掉、加上或改变有限项，敛散性不变（注意不是和不变）
> $$

> 性质4)
> 
> $$
> \large \sum U_n收敛，任意加括号得级数也收敛，且和不变。
> $$

正推可以，反推不可以

**但是，加括号后发散的，原级数一定发散**

> 性质5)
> 
> $$
> \begin{align*}
\large \sum &U_n收敛，U_n \to 0.\\
\because &U_n=S_n-S_{n-1} \\
&S_n \to S \\
&S_{n-1} \to S \\
\therefore &U_n \to 0
\end{align*}
> $$

正推可以，反推不可以

性质5的逆否命题（为真）做题时很有用

> $$
> U_n \nrightarrow 0,\sum U_n 发散
> $$

## 调和级数

调和级数是个发散的级数，因为项跟项之间的和增大的速度大于后面项趋于0的速度

例子:

$$
1+\frac1{2}+\frac1{3}+\cdot\cdot\cdot+\frac1{n}+\cdot\cdot\cdot是发散的
$$

## 正项级数

$U_n \geq 0$

$S_1\leq S_2\leq S_3\,\cdot\cdot\cdot$

$\{S_n\} \geq 0$

> **定理1**
> 
> $$
> \begin{align*}
&前提都是正项级数\\
&\sum U_n 收敛 \Leftrightarrow \{S_n\}有界\\
\end{align*}
> $$

> **定理2**
> 
> $$
> \begin{align*}
\sum U_n,\sum V_n是正项级数，且U_n \leq V_n
\begin{cases}
\sum V_n 收敛 \to \sum U_n 收敛\\
\sum U_n 发散 \to \sum V_n 发散\\
\end{cases}
\end{align*}
> $$

### P一级数

$$
1+\frac{1}{2^p}+\frac{1}{3^p}+\cdot\cdot\cdot\frac{1}{n^p}+\cdot\cdot\cdot
$$

结论：

$$
① p \leq1,发散\\
② p >1,收敛
$$

$$
\frac{1}{k^p}=\int^k_{k-1}\frac1{k^p}dx\leq\int^k_{k-1}\frac1{x^p}dx
$$

$$
S_n=1+\sum^n_{k=2}\frac1{k^p}\leq 1+\sum^n_{k-1}\frac1{x^p}dx=1+\int^n_1\frac1{x^p}dx
=1+\frac1{p-1}(1-\frac1{n^{p-1}})<1+\frac1{p-1}\\

\therefore S_n 有界
$$

### 正项级数的比较审敛法

#### 极限改进的比较审敛法：

$\sum U_n和\sum V_n都是正项级数$

> 定理1：
> 
> $$
> \lim\limits_{n\to\infty}\frac{U_n}{V_n}=l,(0\leq l < +\infty) \Rightarrow
\sum V_n 收敛，\sum U_n 收敛
> $$

> 定理2：
> 
> $$
> \lim\limits_{n\to \infty}\frac{U_n}{V_n}=l > 0 或 +\infty \Rightarrow
\sum V_n 发散，\sum U_n 发散
> $$

### 正项级数的比值审敛法

> 定理
> 
> $$
> \lim\limits_{n\to\infty}\frac{U_{n+1}}{U_n}=\rho\\
\rho < 1 收敛，\rho > 1 (包含+\infty)发散
> $$
> 
> 缺点是 $\rho = 1$时无法判断，优点是不需要跟比较审敛法一样找一个比较对象

### 正向级数的根值审敛法（柯西判别法）

> 定理
> 
> $$
> \lim\limits_{n\to\infty}\sqrt[n]{U_n}=\rho \\
\begin{align*}
1)& \rho < 1 收敛 \\
2)& \rho > 1 发散 \\
3)& \rho = 1 无法确定（同比值审敛法，本方法失效）
\end{align*}
> $$

## 交错级数

$$
\begin{align*}
&U_n \geq 0\\
&U_1-U_2+U_3-U_4+\cdot\cdot\cdot \\
&-U_1+U_2-U_3+U_4+\cdot\cdot\cdot \\
\sum^\infty_{n=1}(-1)^{n-1}U_n
\end{align*}
$$

> 莱布尼茨定理
> 
> $$
> \begin{align*}
&1) U_n \geq U_{n+1}\\
&2) \lim\limits_{n\to\infty} U_n = 0\\
则级数收敛，S \leq U_1,并且余项|r_n|\leq U_{n+1}
\end{align*}
> $$

## 任意项级数

$$
\begin{align*}
&任意项级数：U_1+U_2+U_3+\cdot\cdot\cdot& (U_n正负不知道)\\
&正项级数：|U_1|+|U_2|+|U_3|+\cdot\cdot\cdot& (绝对值级数)\\
\end{align*}
$$

> 定理：
> 
> $$
> \sum^{+\infty}_{n=1}|U_n|收敛，则\sum^{+\infty}_{n=1}U_n也收敛
> $$

$$
\begin{align*}
&绝对收敛： \sum |U_n|收敛，称\sum U_n绝对收敛 \\
&条件收敛： \sum U_n收敛，\sum |U_n|发散，\sum U_n是条件收敛
\end{align*}
$$

> 定理：
> 
> $$
>  \sum^{+\infty}_{n=1}U_n=U_1+U_2+U_3+\cdot\cdot\cdot是任意项级数\\
> $$
> 
>    $$

$$
\lim\limits_{n\to +\infty}|\frac{U_n+1}{U_n}|=l \longleftarrow
正项级数的比值审敛法\\
①: l < 1 时，\sum U_n(绝对)收敛\\
②：l > 1 (+\infty)时，\sum U_n 发散
③：l = 1 时，本方法无法判断
$$



## 幂级数

$$
1+x+x^2+x^3+\cdot\cdot\cdot+x^n+\cdot\cdot\cdot
\begin{cases}
|x|<1时，收敛域(-1,1),和是\frac{a}{1-q}=\frac1{1-x}\\
|x| \geq 1 时，发散域(-\infty,-1]\cup[1,+\infty)
\end{cases}
$$

> 定理（阿贝尔定理）
> 
> $$
> \sum^{+\infty}_{n=0}a_nx^n,
\begin{cases}
如果x=x_0时收敛，则|x|<|x_0|时幂级数绝对收敛\\
如果x=x_0时收敛，则|x|>|x_0|时幂级数发散
\end{cases}
> $$

> 推论：
> 
> $$
> 收敛情况
\begin{cases}
①:x=0时收敛,例如\sum n!x^n\\
②:x\in(-\infty,+\infty)时收敛,例如\sum \frac{x^n}{n!}\\
③:|x|<R绝对收敛(但是在-R跟R这两个端点的情况就需要另外讨论)\\
\end{cases}
> $$

$$
\begin{align*}
&(R>0)R叫作收敛半径，(-R,R)叫收敛区间\\
&[-R,R),[-R,R],(-R,R),(-R,R]这四种叫收敛域\\
&收敛区间不包括端点，收敛域需要在讨论完端点的情况后决定是否将端点加进来
\end{align*}
$$

### 如何求 R?

> 定理
> 
> $$
> \begin{align*}
&\lim\limits_{n\to\infty}|\frac{a_{n+1}}{a_n}|=\rho\\
&R=
\begin{cases}
\frac1{\rho},\,\,\,&\rho\neq0\\
+\infty,\,\,\,&\rho=0\\
0,\,\,\,&\rho=+\infty
\end{cases}
&其实可以只记R=\frac1{\rho}
\end{align*}
> $$

### 幂级数的运算

$$
\begin{align*}
&性质1)\sum^{+\infty}_{n=0}a_nx^n的和函数S(x)在收敛域I上是连续的 \\
&性质2)\sum^{+\infty}_{n=0}a_nx^n的和函数S(x)在收敛域I上是可积的,例如\int^x_0S(t)dt&=
\int^x_0 \sum^{+\infty}_{n=0}(a_nt^n)dt \\
&&=\sum^{+\infty}_{n=0}\int^x_0 a_nt^ndt \,\,\,\\
&逐项求积分后与原幂级数的收敛半径相同，但收敛域就需要重新考察端点情况 \\
&性质3)\,\,S(x)在(-R,R)内可导,S'(x)=(\sum^{+\infty}_{n=0}a_nx^n)'=
\sum^{+\infty}_{n=0}(a_nx^n)'=\sum^{+\infty}_{n=0}na_nx^{n-1}\\
&逐项求导后与原幂级数的收敛半径相同，但收敛域就需要重新考察端点情况 \\
\end{align*}
$$

### 函数展成幂级数

> 直接展开法：
> 
> 就是直接用麦克劳林公式展开

> 间接展开法:
> 
> 就是利用已有的进行展开，做题一般都用这个
> 
> 记忆：
> 
> $$
> \begin{align*}
①&e^x=1+x+\frac{x^2}{2!}+\frac{x^3}{3!}+\cdot\cdot\cdot+\frac{x^n}{n!}+
\cdot\cdot\cdot (-\infty<x<+\infty) \\
②&\sin x = x-\frac{x^3}{3!}+\frac{x^5}{5!}-\frac{x^7}{7!}+\cdot\cdot\cdot+
(-1)^n\frac{x^{2n+1}}{(2n+1)!}+\cdot\cdot\cdot(-\infty<x<+\infty)\\
③&\frac1{1-x}=1+x+x^2+x^3+\cdot\cdot\cdot+x^n+\cdot\cdot\cdot(-1<x<1)\\
④&\frac1{1+x}=1-x+x^2-x^3+\cdot\cdot\cdot+(-1)^nx^n+\cdot\cdot\cdot(-1<x<1)\\
&0到x求积分\huge\Downarrow \\
⑤&\ln(1+x)=x-\frac{x^2}{2}+\frac{x^3}{3}-\frac{x^4}{4}+\cdot\cdot\cdot+
\frac{(-1)^{n-1}}{n}x^n+\cdot\cdot\cdot(-1<x\leq1)\\
⑥&\cos x=1-\frac{x^2}{2!}+\frac{x^4}{4!}-\frac{x^6}{6!}+\cdot\cdot\cdot+
(-1)^n\frac{x^{2n}}{(2n)!}+\cdot\cdot\cdot(-\infty<x<+\infty)\\
⑦&a^x=1+x\ln a + \frac{x^2(\ln a)^2}{2!}+ \frac{x^3(\ln a)^3}{3!}+
\cdot\cdot\cdot+ \frac{x^n(\ln a)^n}{n!}+\cdot\cdot\cdot(-\infty<x<+\infty)\\
⑧&\frac{1}{1+x^2}=1-x^2+x^4-x^6+x^8+(-1)^nx^{2n}+\cdot\cdot\cdot(-1<x<1)\\
⑨&\arctan x=x-\frac{x^3}{3}+\frac{x^5}{5}-\frac{x^7}{7}+\cdot\cdot\cdot+
(-1)^n\frac{x^{2n+1}}{2n+1}+\cdot\cdot\cdot(-1\leq x\leq1)
\end{align*}
> $$

# 向量和解析几何

## 平面夹角

$$
n_1(A_1,B_1,C_1),n_2(A_2,B_2,C_2)为两平面各自的法向量。\\
\cos\theta=\frac{|A_1A_2+B_1B_2+C_1C_2|}{\sqrt{A_1^2+B_1^2+C_1^2}\sqrt{A_2^2+B_2^2+C_2^2}}
\begin{cases}
①：垂直, A_1A_2+B_1B_2+C_1C_2=0\\

②：平行或重合,\frac{A_1}{A_2}=\frac{B_1}{B_2}=\frac{C_1}{C_2} \\
\end{cases}
$$

## 距离公式

$$
P_0(x_0,y_0,z_0),Ax+By+Cz+D=0 \\
d=\frac{|Ax_0+By_0+Cz_0+D|}{\sqrt{A^2+B^2+C^2}}
$$
