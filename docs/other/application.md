# Application应用

Kera的应用模块Application提供了带有预训练权重的Keras模型，这些模型可以用来进行预测、特征提取和finetune

模型的预训练权重将下载到```~/.keras/models/```并在载入模型时自动载入

## 可用的模型

应用于图像分类的模型,权重训练自ImageNet：
* [Xception](#xception)
* [VGG16](#vgg16)
* [VGG19](#vgg19)
* [ResNet50](#resnet50)
* [InceptionV3](#inceptionv3)

所有的这些模型(除了Xception)都兼容Theano和Tensorflow，并会自动基于```~/.keras/keras.json```的Keras的图像维度进行自动设置。例如，如果你设置```data_format="channel_last"```，则加载的模型将按照TensorFlow的维度顺序来构造，即“Width-Height-Depth”的顺序


***

## 图片分类模型的示例

### 利用ResNet50网络进行ImageNet分类
```python
from keras.applications.resnet50 import ResNet50
from keras.preprocessing import image
from keras.applications.resnet50 import preprocess_input, decode_predictions
import numpy as np

model = ResNet50(weights='imagenet')

img_path = 'elephant.jpg'
img = image.load_img(img_path, target_size=(224, 224))
x = image.img_to_array(img)
x = np.expand_dims(x, axis=0)
x = preprocess_input(x)

preds = model.predict(x)
# decode the results into a list of tuples (class, description, probability)
# (one such list for each sample in the batch)
print('Predicted:', decode_predictions(preds, top=3)[0])
# Predicted: [(u'n02504013', u'Indian_elephant', 0.82658225), (u'n01871265', u'tusker', 0.1122357), (u'n02504458', u'African_elephant', 0.061040461)]

```

### 利用VGG16提取特征
```python
from keras.applications.vgg16 import VGG16
from keras.preprocessing import image
from keras.applications.vgg16 import preprocess_input
import numpy as np

model = VGG16(weights='imagenet', include_top=False)

img_path = 'elephant.jpg'
img = image.load_img(img_path, target_size=(224, 224))
x = image.img_to_array(img)
x = np.expand_dims(x, axis=0)
x = preprocess_input(x)

features = model.predict(x)
```

### 从VGG19的任意中间层中抽取特征
```python
from keras.applications.inception_v3 import InceptionV3
from keras.preprocessing import image
from keras.models import Model
from keras.layers import Dense, GlobalAveragePooling2D
from keras import backend as K

# create the base pre-trained model
base_model = InceptionV3(weights='imagenet', include_top=False)

# add a global spatial average pooling layer
x = base_model.output
x = GlobalAveragePooling2D()(x)
# let's add a fully-connected layer
x = Dense(1024, activation='relu')(x)
# and a logistic layer -- let's say we have 200 classes
predictions = Dense(200, activation='softmax')(x)

# this is the model we will train
model = Model(input=base_model.input, output=predictions)

# first: train only the top layers (which were randomly initialized)
# i.e. freeze all convolutional InceptionV3 layers
for layer in base_model.layers:
    layer.trainable = False

# compile the model (should be done *after* setting layers to non-trainable)
model.compile(optimizer='rmsprop', loss='categorical_crossentropy')

# train the model on the new data for a few epochs
model.fit_generator(...)

# at this point, the top layers are well trained and we can start fine-tuning
# convolutional layers from inception V3. We will freeze the bottom N layers
# and train the remaining top layers.

# let's visualize layer names and layer indices to see how many layers
# we should freeze:
for i, layer in enumerate(base_model.layers):
   print(i, layer.name)

# we chose to train the top 2 inception blocks, i.e. we will freeze
# the first 172 layers and unfreeze the rest:
for layer in model.layers[:172]:
   layer.trainable = False
for layer in model.layers[172:]:
   layer.trainable = True

# we need to recompile the model for these modifications to take effect
# we use SGD with a low learning rate
from keras.optimizers import SGD
model.compile(optimizer=SGD(lr=0.0001, momentum=0.9), loss='categorical_crossentropy')

# we train our model again (this time fine-tuning the top 2 inception blocks
# alongside the top Dense layers
model.fit_generator(...)
```

### 在定制的输入tensor上构建InceptionV3
```python
from keras.applications.inception_v3 import InceptionV3
from keras.layers import Input

# this could also be the output a different Keras model or layer
input_tensor = Input(shape=(224, 224, 3))  # this assumes K.image_data_format() == 'channels_last'

model = InceptionV3(input_tensor=input_tensor, weights='imagenet', include_top=True)
```

***

## 模型文档
* [Xception](#xception)
* [VGG16](#vgg16)
* [VGG19](#vgg19)
* [ResNet50](#resnet50)
* [InceptionV3](#inceptionv3)
* [MusicTaggerCRNN](#music)


***

<a name='xception'>
<font color='#404040'>
## Xception模型
</font>
</a>
```python
keras.applications.xception.Xception(include_top=True, weights='imagenet',
             						input_tensor=None, input_shape=None,
             						pooling=None, classes=1000)
```

Xception V1 模型, 权重由ImageNet训练而言

在ImageNet上,该模型取得了验证集top1 0.790和top5 0.945的正确率

注意,该模型目前仅能以TensorFlow为后端使用,由于它依赖于"SeparableConvolution"层,目前该模型只支持tf的维度顺序(width, height, channels)

默认输入图片大小为299x299

### 参数
* include_top：是否保留顶层的3个全连接网络
* weights：None代表随机初始化，即不加载预训练权重。'imagenet'代表加载预训练权重
* input_tensor：可填入Keras tensor作为模型的图像输出tensor
* input_shape：可选，仅当`include_top=False`有效，应为长为3的tuple，指明输入图片的shape，图片的宽高必须大于71，如(150,150,3)
* pooling：当include_top=False时，该参数指定了池化方式。None代表不池化，最后一个卷积层的输出为4D张量。‘avg’代表全局平均池化，‘max’代表全局最大值池化。

* classes：可选，图片分类的类别数，仅当`include_top=True`并且不加载预训练权重时可用。


### 返回值

Keras 模型对象

### 参考文献

* [Xception: Deep Learning with Depthwise Separable Convolutions](https://arxiv.org/abs/1610.02357)

### License
预训练权重由我们自己训练而来，基于MIT license发布

***

<a name='vgg16'>
<font color='#404040'>
## VGG16模型
</font>
</a>
```python
keras.applications.vgg16.VGG16(include_top=True, weights='imagenet',
          						input_tensor=None, input_shape=None,
          						pooling=None,
          						classes=1000)
```

VGG16模型,权重由ImageNet训练而来

该模型再Theano和TensorFlow后端均可使用,并接受th和tf两种输入维度顺序

模型的默认输入尺寸时224x224

### 参数
* include_top：是否保留顶层的3个全连接网络
* weights：None代表随机初始化，即不加载预训练权重。'imagenet'代表加载预训练权重
* input_tensor：可填入Keras tensor作为模型的图像输出tensor
* input_shape：可选，仅当`include_top=False`有效，应为长为3的tuple，指明输入图片的shape，图片的宽高必须大于48，如(200,200,3)
### 返回值
* pooling：当include_top=False时，该参数指定了池化方式。None代表不池化，最后一个卷积层的输出为4D张量。‘avg’代表全局平均池化，‘max’代表全局最大值池化。
* classes：可选，图片分类的类别数，仅当`include_top=True`并且不加载预训练权重时可用。

Keras 模型对象

### 参考文献

* [Very Deep Convolutional Networks for Large-Scale Image Recognition](https://arxiv.org/abs/1409.1556)：如果在研究中使用了VGG，请引用该文

### License
预训练权重由[牛津VGG组](http://www.robots.ox.ac.uk/~vgg/research/very_deep/)发布的预训练权重移植而来，基于[Creative Commons Attribution License](https://creativecommons.org/licenses/by/4.0/)

***

<a name='vgg19'>
<font color='#404040'>
## VGG19模型
</font>
</a>
```python
keras.applications.vgg19.VGG19(include_top=True, weights='imagenet',
          						input_tensor=None, input_shape=None,
          						pooling=None,
          						classes=1000)
```
VGG19模型,权重由ImageNet训练而来

该模型再Theano和TensorFlow后端均可使用,并接受th和tf两种输入维度顺序

模型的默认输入尺寸时224x224
### 参数
* include_top：是否保留顶层的3个全连接网络
* weights：None代表随机初始化，即不加载预训练权重。'imagenet'代表加载预训练权重
* input_tensor：可填入Keras tensor作为模型的图像输出tensor
* input_shape：可选，仅当`include_top=False`有效，应为长为3的tuple，指明输入图片的shape，图片的宽高必须大于48，如(200,200,3)
* pooling：当include_top=False时，该参数指定了池化方式。None代表不池化，最后一个卷积层的输出为4D张量。‘avg’代表全局平均池化，‘max’代表全局最大值池化。
* classes：可选，图片分类的类别数，仅当`include_top=True`并且不加载预训练权重时可用。
### 返回值
### 返回值

Keras 模型对象

### 参考文献

* [Very Deep Convolutional Networks for Large-Scale Image Recognition](https://arxiv.org/abs/1409.1556)：如果在研究中使用了VGG，请引用该文

### License
预训练权重由[牛津VGG组](http://www.robots.ox.ac.uk/~vgg/research/very_deep/)发布的预训练权重移植而来，基于[Creative Commons Attribution License](https://creativecommons.org/licenses/by/4.0/)

***

<a name='resnet50'>
<font color='#404040'>
## ResNet50模型
</font>
</a>
```python
keras.applications.resnet50.ResNet50(include_top=True, weights='imagenet',
          						input_tensor=None, input_shape=None,
          						pooling=None,
          						classes=1000)
```

50层残差网络模型,权重训练自ImageNet

该模型再Theano和TensorFlow后端均可使用,并接受th和tf两种输入维度顺序

模型的默认输入尺寸时224x224

### 参数
* include_top：是否保留顶层的全连接网络
* weights：None代表随机初始化，即不加载预训练权重。'imagenet'代表加载预训练权重
* input_tensor：可填入Keras tensor作为模型的图像输出tensor
* input_shape：可选，仅当`include_top=False`有效，应为长为3的tuple，指明输入图片的shape，图片的宽高必须大于197，如(200,200,3)
* pooling：当include_top=False时，该参数指定了池化方式。None代表不池化，最后一个卷积层的输出为4D张量。‘avg’代表全局平均池化，‘max’代表全局最大值池化。
* classes：可选，图片分类的类别数，仅当`include_top=True`并且不加载预训练权重时可用。
### 返回值

Keras 模型对象

### 参考文献

* [Deep Residual Learning for Image Recognition](https://arxiv.org/abs/1512.03385)：如果在研究中使用了ResNet50，请引用该文

### License
预训练权重由[Kaiming He](https://github.com/KaimingHe/deep-residual-networks)发布的预训练权重移植而来，基于[MIT License](https://github.com/KaimingHe/deep-residual-networks/blob/master/LICENSE)

***

<a name='inceptionv3'>
<font color='#404040'>
## InceptionV3模型
</font>
</a>
```python
keras.applications.inception_v3.InceptionV3(include_top=True,
                							weights='imagenet',
                							input_tensor=None,
                							input_shape=None,
                							pooling=None,
                							classes=1000)
```
InceptionV3网络,权重训练自ImageNet

该模型再Theano和TensorFlow后端均可使用,并接受th和tf两种输入维度顺序

模型的默认输入尺寸时299x299
### 参数
* include_top：是否保留顶层的全连接网络
* weights：None代表随机初始化，即不加载预训练权重。'imagenet'代表加载预训练权重
* input_tensor：可填入Keras tensor作为模型的图像输出tensor
* pooling：当include_top=False时，该参数指定了池化方式。None代表不池化，最后一个卷积层的输出为4D张量。‘avg’代表全局平均池化，‘max’代表全局最大值池化。
* classes：可选，图片分类的类别数，仅当`include_top=True`并且不加载预训练权重时可用。
### 返回值

Keras 模型对象

### 参考文献

* [Rethinking the Inception Architecture for Computer Vision](http://arxiv.org/abs/1512.00567)：如果在研究中使用了InceptionV3，请引用该文

### License
预训练权重由我们自己训练而来，基于[MIT License](https://github.com/KaimingHe/deep-residual-networks/blob/master/LICENSE)

***




