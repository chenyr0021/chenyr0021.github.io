---
title: 「机器学习」笔记1：决策树原理及实现
categories: 学习笔记
date: 2019-12-20 11:11:46
tags: 机器学习
mathjax: true
---
参考：机器学习实战，[美]Peter Harrington 著，李锐 李鹏 曲亚东译

# 引言

决策树是对数据进行分类的一种算法。对数据的特征一一判断，从而得出该数据属于哪一个类别。

举个简单的例子。比如判断一种生物是否属于鱼类，我们可以先看它不浮出水面是否可以生存，如果不可以，就不是鱼类；如果可以，再看它是否有脚蹼，如果有，则是鱼类，如果没有，就不是。

文字说明可能不太清楚，画个图会看起来更清晰。

![image](https://raw.githubusercontent.com/chenyr0021/chenyr0021.github.io/hexo/source/_posts/machine-learning-decision-tree.jpg)


（当然判断鱼类不止这么简单，这里只是为了说明决策树的原理）

假设判断鱼类就只需要这两个条件，机器学习中称之为特征，那么机器如何知道先分析哪个特征可以更快更有效率地得到分类结果呢？

更进一步，当其他分类问题需要分析不止2个特征时，这个问题就变成，如何安排特征分析的顺序以使未知类别的数据得到更快的分类。

这就是决策树所解决的问题。

决策树通过对数据集样本的分析，得到一个对数据分类的最优方法（就是分析特征的最优顺序），即通过最小的步骤/代价得到分类结果的分类规则。

具体到一个特定问题上。还是上面鱼类的问题，给出下列数据集：

样本\特征|不浮出水面是否可以生存|是否有脚蹼|label：是否鱼类
-----|-----|-----|-----|
1|是|是|是|
2|是|是|是|
3|是|否|否|
4|否|是|是|
5|否|是|否|

上表的5个样本分别有2个特征，及对应的label，也就是类别。下面我们就以该数据集为例，构建一个划分鱼类的决策树。

# 构造决策树

既然我们要让机器学习到使分类最有效率的分析特征的顺序，就要为这个「效率」定一个量化标准。就像深度学习中的损失函数，为整个网络提供了优化的目标——使损失函数变得更小。在决策树的构造中，这个量化标准就是：熵。

决策树的构造就是选择当前使数据集的熵减少的最多的划分方法，把数据集划分成若干子数据集，再在子数据集中重复以上步骤，直到所有特征都用完或者样本都属于同一类别。

在每个子数据集做的工作都是相同的，所以使用递归实现。

程序逻辑：

>     函数 create_tree()
>
>     If 当前数据集的每个样本都属于同一类
>         Return 该类别
>     Else if 所有特征都已使用
>         Return 该数据集中样本数量最多的类别
>     Else
>         选择下一个划分数据集的特征（根据熵减小的量）
>         划分数据集
>         创建分支
>         for 每个划分的子集
>               调用本函数 create_tree()
>               将结果添加到分支
>     Return 当前分支

## 香农熵

熵描述的是信息的无序程度，信息越无序，熵越大。和高中化学中的熵有点像。

信息的熵由香农提出，量化表示如下：


$$ H = -\sum_{i=1}^np(x_i)\log_2{p(x_i)} $$


`p(xi)`表示选择该分类的概率。从这个公式可以看出，数据类别越一致，熵的值越小。如果所有的数据为同一类别，熵就为0。

决策树对于数据的分类就基于减少整个数据集的香农熵。

创建一个数据集：

```py
def create_dataset():
    dataset = [[1, 1, 'yes'],
               [1, 1, 'yes'],
               [1, 0, 'no'],
               [0, 1, 'no'],
               [0, 1, 'no']]
    features = ['no surfacing', 'flippers']
    return dataset, features
```

对形似以上列表的数据集计算香农熵：

```py
import math

def cal_shannon_entropy(dataset:list):
    example_num = len(dataset)
    entropy = 0.0
    label_cnt = {}
    label_list = [example[-1] for example in dataset]
    for label in label_list:
        label_cnt[label] = label_cnt.get(label, 0) + 1
    for key in label_cnt.keys():
        prob = label_cnt[key]/example_num
        entropy -= prob * math.log(prob, 2)
    return entropy
```

Terminal 运行(文件名为tree.py)：

>     import tree
>     dataset, labels = tree.create_dataset()
>     tree.cal_shannon_entropy(dataset)

输出：
>     0.97095059445466858

## 划分数据集

现在我们已经知道如何计算数据集的有序度，那么接下来就考虑，使用当前情况下哪个特征划分数据集能使有序度提高，也就是香农熵下降。

熵值在划分前后的变化量，叫做信息增益。

首先我们需要一个根据某一特征的值划分数据集的函数。对于上一节的数据集，假设按照`no surfacing=1`划分，则可以得到前3个样本组成的子数据集。用下面函数实现：

```py
def split_dataset(dataset:list, axis:int, value):
    sub_dataset = []
    for ind, example in enumerate(dataset):
        if example[axis] == value:
            # delete the feature which has been used
            reduced_vec = example[:axis] + example[axis+1:]
            sub_dataset.append(reduced_vec)
    return sub_dataset
```


`no surfacing`这个特征可以将数据集划分为2个子数据集，分别是前3个样本和后2个样本。对这两个子数据集分别求香农熵让后求和，就可以得到分类后的总熵。使用划分前的熵减去划分后的熵，就可以得到信息增益。对所有特征的信息增益进行比较，值最大的信息增益对应的特征，就是当前划分数据集最好的特征。

以上过程写成函数：

```py
def best_feat_to_split(dataset:list):
    base_entropy = cal_shannon_entropy(dataset)
    max_info_gain = -1
    best_feat_ind = -1
    feat_len = len(dataset[0]) - 1
    for feat in range(feat_len):
        # get all kinds of value of this feature
        feat_list = [example[feat] for example in dataset]
        feat_class = set(feat_list)
        # split the dataset by all the values , and calculate entropy
        entropy = 0.0
        for feat_value in feat_class:
            sub_dataset = split_dataset(dataset, feat, feat_value)
            entropy += cal_shannon_entropy(sub_dataset)
        # calculate decrement of entropy
        info_gain = base_entropy - entropy
        if info_gain >= max_info_gain:
            best_feat_ind = feat
            max_info_gain = info_gain
    return best_feat_ind
```

该函数返回当前划分数据集最好的特征的索引值。

得到该特征，据此划分数据集，再对划分后的子数据集重复以上过程，直到满足2个结束条件之一。

第1个条件比较简单。第2个条件稍微复杂，就是求取该子集中样本数量最多的类别。写个函数描述一下：

```py
import operator

def major_class(class_list:list):
    class_cnt = {}
    for class_name in class_list:
        class_cnt[class_name] = class_cnt.get(class_name, 0) + 1
    sorted_class = sorted(class_cnt.items(), key=operator.itemgetter(1), reverse=True)
    return sorted_class[0][0]
```

函数的参数是当前数据集的类别标签列表。

## 构建决策树

现在，我们需要的所有准备工作都做完了。可以根据前文的伪代码写最后的决策树了。

```py
def create_tree(dataset:list, labels:list):
    # ending conditions
    class_list = [example[-1] for example in dataset]
    if len(dataset[0]) == 1:
        return major_class(class_list)
    if class_list.count(class_list[0]) == len(class_list):
        return class_list[0]

    feat_ind = best_feat_to_split(dataset)
    best_feat = labels[feat_ind]
    tree = {best_feat:{}}
    labels = labels[:feat_ind] + labels[feat_ind+1:]
    feat_values = set([example[feat_ind] for example in dataset])
    for feat_value in feat_values:
        sub_labels = labels[:]
        sub_dataset = split_dataset(dataset, feat_ind, feat_value)
        tree[best_feat][feat_value] = create_tree(sub_dataset, sub_labels)
    return tree
```

运行：

>     import tree
>     dataset, labels = tree.create_dataset()
>     decision_tree = create_tree(dataset, labels)

输出：

>     {'no surfacing': {0: 'no', 1: {'flippers': {0: 'no', 1: 'yes'}}}}

《机器学习实战》书里还讲了如何将这种保存在字典里的树进行可视化，这里就不详细说明了。不过如果需要将大规模的决策树保存成文件，可以使用`pickel`模块。

`pickle`是Python的内建模块。在Python3中，使用二进制保存数据到`.pickle`文件中。

读写pickle文件示例：

>     import pickle as pkl
>     # write
>     with open('tree.pickle', 'wb') as f:
>         pkl.dump(decision_tree, f)
>     # read
>     with open('tree.pickle', 'rb') as f:
>         decision_tree = pkl.load(f)

# 使用决策树进行分类

我们已经得到了上述的决策树，如何运用它对一个未知的数据进行分类呢？到这里，它的工作才真正像我们在引言中描述的那样，根据一个一个特征值进行判断。

上面得到的树长这样：

>     {'no surfacing': {0: 'no', 1: {'flippers': {0: 'no', 1: 'yes'}}}}

新的数据长这样:

>     [1, 0]

（假装它是个数据集里没有的新数据emmm

根据我们对决策树的理解，不就应该先看`no surfacing`，等于1，再看`flippers`，等于0，所以是`no`嘛。

转换成程序就是：

```py
def classify(input_tree:dict, feat_labels:list, test_vec:list):
    # get the first feature and its index in feat_labels
    first_feat = list(input_tree.keys())[0]
    feat_ind = feat_labels.index(first_feat)
    # get the corresponding branch
    second_dict = input_tree[first_feat][test_vec[feat_ind]]
    if type(second_dict) == dict:
        ret = classify(second_dict, feat_labels, test_vec)
        return ret
    else:
        return second_dict
```

也是一个递归的过程。似乎冥冥之中有些联系？（哈哈）

运行：

>     tree.classify(decision_tree, labels, [1, 0])

输出：

>     no

# 总结

决策树分类器是基于数据集建立起来的，通过对数据集的分析，计算各种划分方式的信息增益，确定一个最优的划分顺序。

这也说明决策树的建立和数据集紧密相关，如果数据集不能很好地表达数据的真正分布情况，那得到的决策树就会受到影响。不过这一点我并没有验证过。

在对新数据分类的时候，根据建立好的树，一个一个判断数据的特征，这个过程就像程序中使用的`if/else`语句，一层一层往下，直到走到决策树的最底层。
