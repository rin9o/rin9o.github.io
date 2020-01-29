---
title: TensorFlow.js 官方教程《从二维数据进行预测》的学习
date: 2020-01-29 10:11:31
tags:
- TensorFlow
- 机器学习
---

众多机器学习框架中，TensorFlow 具有比较完善的文档和社区支持，而 JS 版本更可以在浏览器上运行与 Web 技术结合起来使用，便开始阅读 TensorFlow.js 的官方教程，这篇博文打算记录一下自己一步步的尝试，也可以看作是官方教程 Codelab 内容的翻译。

本文对应的 Codelab 教程地址在此，如果你的英语水平较好可以直接阅读：https://codelabs.developers.google.com/codelabs/tfjs-training-regression/index.html

这次的任务是利用 Google 给出的汽车数据，通过监督学习输入汽车的马力和油耗（英里/加仑）训练出一个模型，将马力数据输入到模型中可以得到预测的油耗情况。

要完成这次任务我们需要将运行过程分为多个步骤：

1. 加载原始数据，并将数据处理为适宜输入到模型网络中训练的形态；
2. 定义模型网络结构；
3. 训练模型，并统计训练过程损失的变化；
4. 让模型作出预测，对比和原始数据的拟合程度。

## 开发前的准备

- 对神经网络的一定了解，初学者推荐看一下 3B1B 的视频：【[Part 1](https://www.bilibili.com/video/av15532370)】
- 现代的浏览器，如 Chrome、Firefox 等
- 基本的 JavaScript 开发经验

### 创建项目

创建一个文件夹放置这次教程中用到的文件代码，并在这里创建一个 `index.html` 作为训练程序运行的页面，内容如下：

```html
<!DOCTYPE html>
<html>
<head>
  <title>TensorFlow.js Tutorial</title>
  <!-- Import TensorFlow.js -->
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@1.0.0/dist/tf.min.js"></script>
  <!-- Import tfjs-vis -->
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-vis@1.0.2/dist/tfjs-vis.umd.min.js"></script>
</head>
<body>
  <!-- Import the main script file -->
  <script src="script.js"></script>
</body>
</html>
```

然后再创建一个 `script.js` 在相同目录，我们即将在此开始学习之旅。

## 第一步：准备数据

监督学习需要有映射关系的输入和输出数据作为我们的学习资料，我们可以直接下载 Google 为我们准备好的汽车数据。

### 下载数据

在 `script.js` 中声明一个 `getData()` 函数：

```javascript
async function getData() {
  const carsDataReq = await fetch('https://storage.googleapis.com/tfjs-tutorials/carsData.json');  
  const carsData = await carsDataReq.json();  
  const cleaned = carsData.map(car => ({
    mpg: car.Miles_per_Gallon,
    horsepower: car.Horsepower,
  }))
  .filter(car => (car.mpg != null && car.horsepower != null));
  
  return cleaned;
}
```

在这个函数中，我们在下载完数据后对每条汽车数据进行简化，去掉了无关变量只留下油耗和马力的数值。

在设计模型之前，我们可以先使用 tfvis 将数据以可视化的散点图展现出来，同样是在 `script.js` 加入以下代码：

```javascript
async function run() {
  // 加载要处理的数据
  const data = await getData();
  const values = data.map(d => ({
    x: d.horsepower,
    y: d.mpg,
  }));

  tfvis.render.scatterplot(
    {name: 'Horsepower v MPG'},
    {values}, 
    {
      xLabel: 'Horsepower',
      yLabel: 'MPG',
      height: 300
    }
  );

  // 后面会有更多代码添加到这下面
}

// 在网页加载完毕后开始运行
document.addEventListener('DOMContentLoaded', run);
```

这时使用浏览器打开 `index.html` 就可以执行上面添加的代码了，如果看到这张图片则说明刚才写的过程能够正常运作：

![](/images/tfjs-codelab-training-regression-1.png)

如果没有看到，检查一下拼写是否错误，以及能否正常访问 `storage.googleapis.com`

### 标准化数据

要提高 TensorFlow 框架的训练效率，我们要使用 TensorFlow 中定义的张量（[Tensors](https://developers.google.com/machine-learning/glossary/#tensor)）来装载数据，同时也要对马力、油耗数值进行[特征缩放](https://zh.wikipedia.org/zh-cn/%E7%89%B9%E5%BE%81%E7%BC%A9%E6%94%BE)到 [0, 1]。

在 `script.js` 中声明一个 `convertToTensor(data)` 函数：

```javascript
function convertToTensor(data) {
  // 通过 tidy 函数来自动释放临时创建的 Tensors
  return tf.tidy(() => {
    // 随机打乱数据
    tf.util.shuffle(data);

    // 将数据转换为 Tensors
    const inputs = data.map(d => d.horsepower)
    const labels = data.map(d => d.mpg);
    const inputTensor = tf.tensor2d(inputs, [inputs.length, 1]);
    const labelTensor = tf.tensor2d(labels, [labels.length, 1]);

    // 重新缩放数据到 [0, 1] 区间
    const inputMax = inputTensor.max();
    const inputMin = inputTensor.min();  
    const labelMax = labelTensor.max();
    const labelMin = labelTensor.min();
    const normalizedInputs = inputTensor.sub(inputMin).div(inputMax.sub(inputMin));
    const normalizedLabels = labelTensor.sub(labelMin).div(labelMax.sub(labelMin));

    return {
      inputs: normalizedInputs,
      labels: normalizedLabels,
      // 保留输入、输出的最大/最小值以便后续使用
      inputMax,
      inputMin,
      labelMax,
      labelMin,
    }
  });  
}
```

这里有几处值得注意的地方：

一、对 `inputs` 转换成 `inputTensor` 张量时，我们在 `tf.tensor2d` 函数中传入的第一个变量是原始的马力数值的一维数组，按照函数名字的意思我们应该传入一个二维数组但为什么可以这么做？

继续分析第二个变量很容易得出，这是在声明即将创建的 Tensor2D 两个维度方向的 length，`[inputs.length, 1]` 则代表“这个输入数据张量总共有 `inputs.length` 条数据，每条数据有 `1` 个特征，也就是马力”。

同理，我们的输出数据有 `labels.length` 条数据，每条只有 `1` 个特征：油耗（Miles per Gallon = `mpg`）。

二、要重新缩放一段数值到 `[0, 1]` 区间，我们只需要 `x1 = (x - min(x)) ÷ (max(x) - min(x))` 这个公式即可。求出较小的值对一些机器学习模型来说是必要的，也能够加速梯度下降法的收敛。

## 第二步：创建模型

> 使用 TensorFlow 有很多种方式来创建模型，按照 Codelab 我们将使用的是 TF 高层封装的 API，简化我们的工作。

机器学习模型是一种接收输入然后产生输出的算法，使用神经网络时这些算法就是若干个由神经元组成的网络层，有各自的权重来影响输出结果，训练过程则是让调整这些权重到理想值使得神经网络获得正确的反馈。

在 `script.js` 中声明这个函数：

```javascript
function createModel() {
  // 创建有序模型
  const model = tf.sequential();
  // 添加一个隐藏层
  model.add(tf.layers.dense({inputShape: [1], units: 1, useBias: true}));
  // 添加一个输出层
  model.add(tf.layers.dense({units: 1, useBias: true}));
  return model;
}
```

这个函数则先是使用 `tf.sequential` 函数创建了一个 `model` 模型，

再通过 `model.add(tf.layers.dense({inputShape: [1], units: 1, useBias: true}))` 添加一个输入数据的隐藏层，由于我们的输入特征只有一个，因此 `inputShape` 被设为 `[1]`；`units` 决定的是这一层的权重矩阵大小。

最后以 `tf.layers.dense({units: 1, useBias: true})` 代表输出层作为收尾。完成定义输出 `model` 。

创建模型并查看模型定义概述，需要在我们最初定义的 `run` 函数内加这一段：

```javascript
const model = createModel();
tfvis.show.modelSummary({name: 'Model Summary'});
```

## 第三步：开始训练

创建好模型之后，我们就可以开始训练了，依旧是在 `script.js` 中声明：

```javascript
async function trainModel(model, inputs, labels) {
  model.compile({
    optimizer: tf.train.adam(),
    loss: tf.losses.meanSquaredError,
    metrics: ['mse'],
  });
  
  const batchSize = 32;
  const epochs = 50;
  
  return await model.fit(inputs, labels, {
    batchSize,
    epochs,
    shuffle: true,
    callbacks: tfvis.show.fitCallbacks(
      { name: 'Training Performance' },
      ['loss', 'mse'],
      { height: 200, callbacks: ['onEpochEnd'] }
    )
  });
}
```

这里定义了一些关键参数：

- **optimizer（优化器）**：是算法优化过程寻找最优解的方法，这里使用了 adam 优化器。
- **loss（损失计算）**：告诉模型当前算法结果与期望值的损失，这里使用了 `meanSquaredError`（方差）来比较。
- **batchSize（批尺寸）**：每一批的样本数量
- **epochs**：计算了所有样本数据的累计次数
- **callbacks**：这里添加了一个每次完成一次 epoch 的回调，用于即时通过 tfvis 展示 loss 的变化。

基于之前所学习的机器学习知识理解上述参数后，我们让训练跑起来，将下面的代码加入到 `run` 函数内：

```javascript
// 转换成我们训练过程可以利用的形式
const tensorData = convertToTensor(data);
const {inputs, labels} = tensorData;

// 训练模型
await trainModel(model, inputs, labels);
console.log('完成训练');
```

刷新 `index.html` 页面，我们就可以看到一张训练过程的 loss 变化曲线图：

![](/images/tfjs-codelab-training-regression-2.png)

## 第四步：验证模型

模型训练出来后，我们需要验证一下它的有效性和准确率，在 `script.js` 中声明以下函数：

```javascript
function testModel(model, inputData, normalizationData) {
  const {inputMax, inputMin, labelMin, labelMax} = normalizationData;  
  
  const [xs, preds] = tf.tidy(() => {

    const xs = tf.linspace(0, 1, 100);
    const preds = model.predict(xs.reshape([100, 1]));

    const unNormXs = xs
      .mul(inputMax.sub(inputMin))
      .add(inputMin);
    const unNormPreds = preds
      .mul(labelMax.sub(labelMin))
      .add(labelMin);

    return [unNormXs.dataSync(), unNormPreds.dataSync()];
  });

  const predictedPoints = Array.from(xs).map((val, i) => {
    return {x: val, y: preds[i]}
  });

  const originalPoints = inputData.map(d => ({
    x: d.horsepower, y: d.mpg,
  }));

  tfvis.render.scatterplot(
    {name: 'Model Predictions vs Original Data'},
    {values: [originalPoints, predictedPoints], series: ['original', 'predicted']},
    {
      xLabel: 'Horsepower',
      yLabel: 'MPG',
      height: 300
    }
  );
}
```

最后在 `run` 函数也加上：

```javascript
testModel(model, data, tensorData);
```

我们就可以得到这个简单的模型所生成的线性回归方程的预测结果图：

![](/images/tfjs-codelab-training-regression-3.png)

原方程应该是一条曲线，对比一下相差结果很大，这是因为我们神经网络的隐藏层十分简单，后续我们可以尝试调整参数、增加隐藏层中的神经元，或者尝试使用 sigmoid 激活函数，这些需要我们继续学习 TensorFlow.js 的使用。

作者自己完善了一次的效果是这样的：

![](/images/tfjs-codelab-training-regression-4.png)

源码可以参考：https://github.com/rin9o/tfjs-codelab-training-regression/blob/25de19186fbc5ec964469e80f5e867167f228638/script.js

（最新的源码已经使用 Webpack + TypeScript 重写了，如果没有了解过这两个可以从上面的链接查看最初 commit 的版本）
