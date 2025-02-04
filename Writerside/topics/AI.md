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


## 1 注意力机制

跨序列进行样本相关性计算的是经典的注意力机制（Attention），在一个序列内部对样本进行相关性计算的是自注意力机制（self-attention）。在Transformer架构中我们所使用的是自注意力机制