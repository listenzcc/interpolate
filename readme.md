# 对密集采样进行时间校准

在处理硬件采样时间抖动时，非在线情况下可通过插值方法校正数据。通过生成模拟数据并使用scipy.interpolate进行插值计算，得到较好的结果。评价插值效果时，指标表明除UnivariateSpline外，其他方法如CubicSpline表现良好。插值旨在获取理想采样时间序列并填补异常缺失值，提高数据准确性。

[https://github.com/listenzcc/interpolate](https://github.com/listenzcc/interpolate)

---

## 硬件采样的时间抖动

对于一个真实存在的模拟信号，如果我用$100Hz$的频率对它进行采样，那么理想的采样时间点为

$$
0ms, 10ms, 20ms, \dots
$$

但实际的采样过程往往需要“等待”硬件返回数据，在采集端我只能控制采样动作的发生时间是准时的，但无法保证硬件返回数据的时间也是准时的，真实的采样结果往往如下图所示，它不仅会有随机延时，甚至会出现意外“丢帧”的情况。

![Untitled](%E5%AF%B9%E5%AF%86%E9%9B%86%E9%87%87%E6%A0%B7%E8%BF%9B%E8%A1%8C%E6%97%B6%E9%97%B4%E6%A0%A1%E5%87%86%2054b489239a244d36aae53364bf848352/Untitled.png)

```python
# ----------------------------------------
# One-by-one sampling method in multiple shot
# 1. Create a new loop
loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)

# 2. Create the tasks
ads = [AttrDict(before=i) for i in range(10)]
tasks = asyncio.gather(*[method(ad) for ad in ads], return_exceptions=True)

# 3. Run one-by-one until finished
print(loop.run_until_complete(tasks))
```

```python
# -----------------------------
# More precious timing method
tic = time.time()
while self.running:
    # ! Wait until the right sample timing
    t = time.time()
    if t < (tic + self.n * self.ts):
        time.sleep(0.001)
        continue
    # ! Put sampling method here
    # Wait until value is received
```

## 校正时间抖动的插值方法比较

为了处理以上问题，在非在线情况下，可以采用插值的方式对数据进行校正，其目的有二

1. 获取理想采样时间的数据序列
2. 补足异常缺失值

### 模拟数据生成与插值计算

首先制作模拟数据，我使用opensimplex包随机生成它们。由于opensimplex生成的曲线具有高阶连续性，因此可以认为在全部定义域内，曲线的值都是解析的，并且是可知的

$$
f(t), t \in (0, 1)
$$

插值过程可以表示成由已知采样值推测未知真实值的过程

$$
\hat{f}(u) = \varphi(f(t), t), t \neq u, t\in samples
$$

其目标函数为

$$
argmin_{\varphi} = loss\begin{pmatrix}\hat{f}(u), f(u))\end{pmatrix}
$$

我使用scipy.interpolate包进行插值计算，暴力尝试了全部可用方法得到结果如下，其中黑色点代表观测到的采样点，彩色线代表插值结果。

![Untitled](%E5%AF%B9%E5%AF%86%E9%9B%86%E9%87%87%E6%A0%B7%E8%BF%9B%E8%A1%8C%E6%97%B6%E9%97%B4%E6%A0%A1%E5%87%86%2054b489239a244d36aae53364bf848352/Untitled%201.png)

[Interpolation (scipy.interpolate) — SciPy v1.12.0 Manual](https://docs.scipy.org/doc/scipy/tutorial/interpolate.html)

[opensimplex](https://pypi.org/project/opensimplex/)

### 结果评价

接下来，进行插值结果评价。我使用sklearn.metrics包进行计算，得到下表。图中展示了几种典型指标的值，这些指标可以分为error和score两类，前者的值越小代表插值效果越好；后者的值越大代表插值效果越好。它说明除了UnivariateSpline插值方法之外，其他方法均得到了较好的结果，其中最好的效果的插值方法是CubicSpline、InterpolatedUnivariateSpline、make_interp_spline等。

![Untitled](%E5%AF%B9%E5%AF%86%E9%9B%86%E9%87%87%E6%A0%B7%E8%BF%9B%E8%A1%8C%E6%97%B6%E9%97%B4%E6%A0%A1%E5%87%86%2054b489239a244d36aae53364bf848352/Untitled%202.png)

![Untitled](%E5%AF%B9%E5%AF%86%E9%9B%86%E9%87%87%E6%A0%B7%E8%BF%9B%E8%A1%8C%E6%97%B6%E9%97%B4%E6%A0%A1%E5%87%86%2054b489239a244d36aae53364bf848352/Untitled%203.png)

![Untitled](%E5%AF%B9%E5%AF%86%E9%9B%86%E9%87%87%E6%A0%B7%E8%BF%9B%E8%A1%8C%E6%97%B6%E9%97%B4%E6%A0%A1%E5%87%86%2054b489239a244d36aae53364bf848352/Untitled%204.png)

![Untitled](%E5%AF%B9%E5%AF%86%E9%9B%86%E9%87%87%E6%A0%B7%E8%BF%9B%E8%A1%8C%E6%97%B6%E9%97%B4%E6%A0%A1%E5%87%86%2054b489239a244d36aae53364bf848352/Untitled%205.png)

[3.3. Metrics and scoring: quantifying the quality of predictions](https://scikit-learn.org/stable/modules/model_evaluation.html)

以上全部代码与分析可见我的Github仓库。

[https://github.com/listenzcc/interpolate](https://github.com/listenzcc/interpolate)