# Keras 入门笔记：从 Sequential 到函数式 API

## Keras 是什么

Keras 是一个高层神经网络 API，目标是把"搭模型、训练、评估"这套流程写得像搭积木。它现在是 TensorFlow 的官方高层接口（通过 `tf.keras` 使用），同时新版 Keras 3 也支持 JAX、PyTorch 等多后端。本文按 `tf.keras` 的主流用法写，概念对 Keras 3 同样成立。

核心抽象只有三个：**层（Layer）**负责变换数据，**模型（Model）**把层组织成计算图，**编译配置**决定用什么优化器和损失函数去训练它。

## 数据先行：喂给模型的东西

神经网络吃的是数值张量，所以训练前有三件固定功课：

1. **形状对齐**：每条样本形状一致；全连接层要求输入是二维 `(样本数, 特征数)`，图片通常是 `(样本数, 高, 宽, 通道数)`。
2. **数值缩放**：像素值 0~255 要除以 255 压到 0~1，否则梯度尺度失衡、训练难收敛；表格数据常用标准化（减均值除标准差）。
3. **标签编码**：分类任务的标签要么保留整数（配 `sparse_categorical_crossentropy` 损失），要么转成 one-hot（配 `categorical_crossentropy`），两者必须对上。

训练集/测试集划分在训练前完成，测试集只在最后评估时用，中途反复看测试集调参等于变相泄漏。

## Sequential：一条直线的模型

层与层首尾相接、没有分支时，用 `Sequential` 像堆叠木一样堆起来：

```python
import numpy as np
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers

# 构造一个完整的二分类小例子：二维平面上的两类点
rng = np.random.default_rng(42)
class0 = rng.normal(loc=(-1.5, -1.5), scale=0.8, size=(200, 2))
class1 = rng.normal(loc=(1.5, 1.5), scale=0.8, size=(200, 2))
x = np.vstack([class0, class1]).astype("float32")
y = np.array([0] * 200 + [1] * 200)

model = keras.Sequential([
    layers.Input(shape=(2,)),
    layers.Dense(16, activation="relu"),
    layers.Dense(8, activation="relu"),
    layers.Dense(1, activation="sigmoid"),
])

model.compile(
    optimizer="adam",
    loss="binary_crossentropy",
    metrics=["accuracy"],
)

history = model.fit(x, y, epochs=10, batch_size=32, validation_split=0.2)
loss, acc = model.evaluate(x, y)
print(f"train loss={loss:.4f} acc={acc:.4f}")
```

几个要点：

- `Dense(16, activation="relu")` 是全连接层：16 个神经元，权重由框架自动初始化和训练，初学者不用手动管。
- 输出层按任务选激活：二分类用单个神经元 + `sigmoid`；多分类用类别数个神经元 + `softmax`；回归不用激活（或按值域选）。
- `Input` 显式声明输入形状是好习惯，`model.summary()` 能打出每层参数量，搭错形状一眼可见。

## 编译：compile 决定怎么学

`compile` 的三件套：

- **optimizer（优化器）**：怎么沿梯度更新参数。新手默认 `adam`，学习率默认 0.001，多半够用。
- **loss（损失函数）**：学得好不好的标尺。二分类 `binary_crossentropy`、多分类 `sparse_categorical_crossentropy`（整数标签）或 `categorical_crossentropy`（one-hot 标签）、回归 `mse`/`mae`。选错损失是最常见的低级错误：模型能跑，但学的是另一件事。
- **metrics（指标）**：训练时给人看的数字（accuracy、auc 等），不参与梯度计算，所以它和 loss 可以不同。

## 训练：fit 的几个旋钮

- `epochs`：全数据过一遍算一个 epoch。不是越大越好，过头会过拟合。
- `batch_size`：一次喂多少条样本算一次更新。32 是常见默认值；显存不够调小，想快调大。
- `validation_split` / `validation_data`：留出验证集监控泛化。训练中盯 `val_loss`：训练损失还在降、验证损失开始升，就是过拟合信号。
- `history` 对象保存每轮的损失和指标曲线（`history.history`），画出来看是基本功。
- **Callbacks**：训练过程的钩子。最常用的两个：`EarlyStopping`（验证指标不再变好就自动停，还能恢复最佳权重）、`ModelCheckpoint`（每轮把当前最好的模型存盘，防断电白跑）。

## 函数式 API：模型有了分支就用它

`Sequential` 只能是一条直线。一旦出现多输入、多输出、层复用、残差连接，就要用**函数式 API**：把层当函数调用，显式写出张量怎么流动，最后用 `Model(inputs=..., outputs=...)` 收拢。

```python
inputs = keras.Input(shape=(2,))
h = layers.Dense(16, activation="relu")(inputs)
h = layers.Dense(8, activation="relu")(h)
outputs = layers.Dense(1, activation="sigmoid")(h)
model_fn = keras.Model(inputs=inputs, outputs=outputs)
model_fn.summary()
```

函数式模型和 Sequential 模型在编译、训练、评估上完全一致，只是定义方式不同。判断标准很简单：画得出一条直线就用 Sequential，画不直就用函数式；再复杂的自定义训练循环才需要继承 `Model` 类重写 `train_step`，入门阶段用不上。

## 评估、预测、保存

- `evaluate(x, y)` 返回损失和指标，用在没参与训练的测试集上才是可信数字。
- `predict(x)` 返回原始输出（概率或数值），分类取 `argmax` 得到类别。
- 保存整个模型（含结构、权重、优化器状态）：`model.save("model.keras")`，读回来是 `keras.models.load_model("model.keras")`。`.keras` 是新版推荐格式；只存权重用 `model.save_weights("weights.weights.h5")`，前提是先有一份同结构的模型来 `load_weights`。
- 只上线推理、不再训练时，保存时带上优化器状态没必要，但带着也无妨，别用老式 HDF5 全模型格式和新版混着来。

## 过拟合：入门必踩，提前认识

症状：训练准确率一路涨到接近 100%，验证准确率中途掉头向下。对付顺序从简到繁：

1. 先看曲线确认是过拟合而不是欠拟合（两边都低是欠拟合，加容量/多练几轮）；
2. 加数据或做数据增强（对建模帮助最大）；
3. 加正则：`Dense(..., kernel_regularizer=keras.regularizers.l2(1e-4))` 或在层后加 `layers.Dropout(0.3)`；
4. `EarlyStopping` 兜底，把最佳轮次的权重留住；
5. 减小模型容量。

## 小结

- 流程固定：备数据 → 选 Sequential 或函数式搭模型 → compile 定优化器/损失/指标 → fit 训练并盯验证曲线 → evaluate 验收 → save 保存。
- 选型口诀：二分类 sigmoid、多分类 softmax、回归无激活；损失与标签编码必须配套。
- 真正的分水岭不在会不会调 API，而在会不会看验证曲线判断过拟合与欠拟合。

本文为原创学习笔记：按公开标准知识编写，未参考同名书籍内容。示例代码已通过 Python 语法检查（py_compile）；因本地无 TensorFlow 环境，未实际执行训练，读者本地运行时请留意版本差异。
