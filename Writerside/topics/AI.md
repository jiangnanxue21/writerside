# AI

# machine learning

装袋法、集成学习和随机森林之间的关系如下：
集成学习
集成学习是一种通过组合多个机器学习模型来提高预测性能的方法。它将多个弱学习器组合成一个强学习器，以提高模型的准确性和稳定性。集成学习主要分为装袋法（Bagging）、提升法（Boosting）和堆叠法（Stacking）。
装袋法（Bagging）
装袋法是集成学习中的一种方法，其核心思想是通过有放回抽样（Bootstrap Sampling）从原始数据集中生成多个子数据集，然后在每个子数据集上独立训练一个模型。最终，通过平均法或多数投票法将这些模型的预测结果组合起来，得到最终的预测结果。装袋法的主要目的是降低模型的方差，提高模型的稳定性和泛化能力。
随机森林
随机森林是装袋法的一种改进和具体实现，专门用于构建决策树集成。它在装袋法的基础上增加了特征选择的随机性：在每次分裂节点时，随机森林只从部分特征中选择最优特征进行分裂，而不是从所有特征中选择。这种随机性进一步降低了决策树之间的相关性，提高了模型的泛化能力。
三者的关系
集成学习是大类，包括装袋法、提升法等多种方法，目的是通过组合多个模型来提高预测性能。
装袋法是集成学习中的一种具体方法，通过有放回抽样生成多个子数据集，并独立训练模型，然后组合结果。
随机森林是装袋法的一种改进和具体实现，专门用于构建决策树集成，通过增加特征选择的随机性来进一步降低模型的方差。



袋外数据（Out-of-Bag, OOB）是指在使用自助采样（Bootstrap Sampling）构建随机森林模型时，未被采样用于训练某棵决策树的数据。由于自助采样是有放回的随机抽样，每次构建决策树时，大约有36.8%的数据不会被选中，这些未被选中的数据就构成了袋外数据。
袋外数据的主要作用是用于模型的内部验证。在随机森林中，每棵树的训练数据集是通过自助采样得到的，因此每棵树都有自己的袋外数据。这些袋外数据可以用来评估模型的泛化能力，而无需额外的测试集。具体来说，袋外误差（OOB Error）是通过使用每棵树的袋外数据进行预测，并将预测结果与真实值进行比较来计算的。这种方法提供了一种无偏的误差估计，有助于评估模型的性能。
在随机森林中，可以通过设置参数（如oob_score=True）来启用袋外误差的计算。这种方法不仅可以节省数据划分的步骤，还能提供一个快速且有效的模型性能评估指标。


![方差-偏差理论.png](方差-偏差理论.png)


Class that can be used to bootstrap and launch a Spring application from a Java main method. By default class will perform the following steps to bootstrap your application:
Create an appropriate ApplicationContext instance (depending on your classpath)
Register a CommandLinePropertySource to expose command line arguments as Spring properties
Refresh the application context, loading all singleton beans
Trigger any CommandLineRunner beans
In most circumstances the static run(Class, String[]) method can be called directly from your main method to bootstrap your application:
@Configuration
@EnableAutoConfiguration
public class MyApplication  {

    // ... Bean definitions
 
    public static void main(String[] args) {
      SpringApplication. run(MyApplication. class, args);
    }
}

For more advanced configuration a SpringApplication instance can be created and customized before being run:
public static void main(String[] args) {
SpringApplication application = new SpringApplication(MyApplication. class);
// ... customize application settings here
application. run(args)
}

## 新词发现

问：什么样的字符组合可以称为一个“词”？

目前新冠病毒的传播速度超过了疫苗的分发速度

- 内部的内聚性强：经常出现

  互信息：当A, B条件独立时，互信息最小为0；A,B越相关，即P(A,B)越大，互信息越大
    ```tex
    MI(A_k, B) = \log \frac{P(AB)}{P(A)P(B)} = \log \frac{P(A|B)}{P(A)} = \log \frac{P(B|A)}{P(B)}
    ```

- 外部的耦合性低：与其他词的搭配不固定

  新词的词语的左右邻字要足够丰富；字符组合左右邻字的丰富程度，可以用信息熵（Entropy）来表示

    ```tex
    Entropy(w) = -\sum_{w_n \in W_{\text{Neighbor}}} P(w_n \mid w) \log_2 P(w_n \mid w)
    ```

  信息熵是对信息量多少的度量，信息熵越高，表示信息量越丰富、不确定性越大。

  ![新词样例.png](新词样例.png)

  “副总裁”左右熵都高，可以成词, “人工智”右熵低，不能成词

## Transformer

### 注意力机制

跨序列进行样本相关性计算的是经典的**注意力机制（Attention）**，在一个序列内部对样本进行相关性计算的是**自注意力机制（self-attention）**。在Transformer架构中我们所使用的是自注意力机制

向量的相关性可以由两个向量的点积来衡量，等于一个发出询问矩阵(Q),一个应答矩阵(K)；QK = 相关性

在实际计算相关性的时候，一般不会直接使用原始特征矩阵并让它与转置矩阵相乘，**因为希望得到的是语义的相关性，而非单纯数字上的相关性**。因此在NLP中使用注意力机制的时候，**我们往往会先在原始特征矩阵的基础上乘以一个解读语义的$w$参数矩阵，以生成用于询问的矩阵Q、用于应答的矩阵K以及其他可能有用的矩阵**。

### 自注意力机制

transformer当中计算的相关性被称之为是**注意力分数**，该注意力分数是在原始的注意力机制上修改后而获得的全新计算方式，其具体计算公式如下

![](https://skojiangdoc.oss-cn-beijing.aliyuncs.com/2023DL/transformer/image-12.png)

```tex
Attention(Q,K,V) = softmax(\frac{QK^{T}}{\sqrt{d_k}})V
```

Transformer为相关性矩阵设置了除以$\sqrt{d_k}$的标准化流程，$d_k$就是特征的维度, 经过Softmax归一化之后的分数，就是注意力机制求解出的**权重**

### Multi-Head Attention 多头注意力机制

在self-attention的基础上，对于输入的embedding矩阵，self-attention只使用了一组$W^Q,W^K,W^V$来进行变换得到Query，Keys，Values。而Multi-Head Attention使用多组$W^Q,W^K,W^V$得到多组Query，Keys，Values，
然后每组分别计算得到一个Z矩阵，最后将得到的多个Z矩阵进行拼接。Transformer原论文里面是使用了8组不同的$W^Q,W^K,W^V$

![](https://skojiangdoc.oss-cn-beijing.aliyuncs.com/2023DL/transformer/image-12.png)


### Encoder
![encoder.png](encoder.png)

Positional Encoding: self attention的时候，加上位置的资讯

![block输出.png](block输出.png)

### Decoder

Autoregressive(AT)

![AT.png](AT.png)

self-attention -> Masked Self-attention

![masked_self_atttention.png](masked_self_atttention.png)

why masked? 因为token是一个个产生的，从左到右

![transformer.png](transformer.png)

#### cross attention

![cross_attention.png](cross_attention.png)

q来自于Decoder，K,V来自于Encoder
