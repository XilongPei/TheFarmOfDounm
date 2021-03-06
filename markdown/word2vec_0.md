﻿# Word2Vec-知其然知其所以然（一）：背景知识

---

> Word2vec的模型很简单，训练也很简单。但是针对为什么模型结构这么设计？为什么采用Hierarchical Softmax的框架？或是为什么采用Negative Sampling的框架？网上貌似没有一个比较好的介绍。

> 因此，这篇博客在介绍Word2vec原理的过程中，顺便也谈到了一些我对Word2vec模型结构的思考(在 `3【重要】Word2vec相比于神经概率语言模型的改进`)，不一定正确，欢迎互相讨论。

> 另外感谢peghoty大神的博客（参考资料1），对于我理解Word2vec算法起了很大的作用。本文的模型及梯度推导推导部分出自peghoty的博客。

[TOC]

## 1 介绍

word2vec是Google于2013年推出的开源的获取词向量word2vec的工具包。它包括了一组用于word embedding的模型，这些模型通常都是用浅层（两层）神经网络训练词向量。

Word2vec的模型以大规模语料库作为输入，然后生成一个向量空间（通常为几百维）。词典中的每个词都对应了向量空间中的一个独一的向量，而且**语料库中拥有共同上下文的词映射到向量空间中的距离会更近**。

## 2 背景知识

### 2.1 Huffman树与Huffman编码

#### 2.1.1 Huffman树的概念

一些关于树的概念：

- **路径/路径长度**：一棵树上一个点到另一个节点的通路称为路径，路径上分支（边）的个数称为路径长度。路径长度通常都是针对根节点而言，因此，设根节点层数为1，则从根节点到$L$层节点路径长度为$L-1$.
- **结点的权/带权路径长度**：为树中的某个节点赋予一个值，则该值就为该节点的权。带权路径长度指的是：从根节点到该节点之间的路径长度与该节点权的乘积。
- **树的带权路径长度**：树的所有叶子节点的带权路径长度之和。
- **Huffman树（霍夫曼树/最优二叉树）**：给定n个权值作为n个叶子节点，则带权路径最小的树称为Huffman树。

#### 2.1.2 Huffman树的构造

给定$n$个权值${w_1,w_2,\dots,w_n}$作为二叉树的$n$个叶子节点，则以此构造Huffman树的算法如下所示：

1. 将${w_1,w_2,\dots,w_3}$看成是有$n$棵树的森林（每棵树仅有一个节点）
2. 从森林中选择**两个根节点权值最小的树**合并，作为一棵新树的左右子树，且新树的根节点权值为其**左右子树根节点权值之和**
3. 从森林中删除被选中的两棵树，并且将新树加入森林
4. 重复2-3步，知道森林中只有一棵树为止，则该树即所求的Huffman树

注意：按照上面算法构造的Huffman树可能不只有一种构型（例如，叶子节点为1,2,3,3），但是所有的构型的Huffman树的带权路径长度都是相同的。

#### 2.1.3 Huffman树构造算法理解与证明

首先，给定叶子节点，让构造一颗二叉树（不一定是Huffman树）的话，那么唯一的算法就是：将每个节点看做一棵只有一个节点的树，然后每次找两棵树合并成一棵新的树，然后将该新树放入森林中，继续上面的步骤。

现在要构造的是『Huffman树，即最优二叉树』，那么问题在于，每一步中选择哪两颗树合并成新的树。

对于『树的带权路径长度』这个衡量标准来说，在上面所描述的构造二叉树的算法中，我们定义一个变量`sum`作为整个森林中所有树的『树的带权路径长度』之和。

我们可以将每一步看作是`sum`从0逐渐增大的过程。每次合并两棵树时，就相当于在`sum`加了一个值，该值就是被合并的两棵树的根的权值之和。到最后一步时，`sum`就是最终的Huffman树的带权路径长度。

因为算法的每一步中，我们都考虑的是树的根节点的权值，所以我们完全可以将每一步中的树都看成叶子节点，叶子节点的权值就是树根权值。这样并不会影响`sum`最终的值。

对于Huffman树而言，我们希望**让权值小的叶子节点层次更深，权值大的叶子节点层次更浅**。那么在将树看成叶子节点后，我们注意到：**在当前步中『被选中用来合并为新节点的两个节点』在『最终的二叉树』上层次必然比当前步中『未被选中的节点』的层次更深**。所以，我们只需要在当前这一步选择所有节点中权值最小的两个节点合并即可。

Huffman树构造算法的理解大概就是这样，具体的形式化证明请参见『算法导论16.3节』。

#### 2.1.4 Huffman编码
数据通信中，需要将传送的文字转换成二进制的字符串，用01的不同排列表示字符。二进制编码大致有两种方式：**等长编码**和**变长编码**。

等长编码即所有字符的编码长度相同，如果有6个字符，那么就需要3位二进制（$2^3=8>6$）。由于等长编码对于所有字符的编码长度相同，因此对于一些出现频率极高的字符来说，等长编码会造成数据压缩率不高。

变长编码可以达到比等长编码好的多的压缩率，其思想就是**赋予高频词短编码，低频词长编码**。变长编码中我们只考虑『前缀编码』，即一个字符的编码不能是另一个字符编码的前缀。

因此，我们可以用字符集中的每个字符作为叶子节点生成一颗编码二叉树，为了获得传送报文的最短长度，可以将每个字符的出现频率作为字符节点的权值赋予该结点上，然后构造一棵Huffman树。利用Huffman树设计的二进制前缀编码，就被称为**Huffman编码**。

Word2vec算法也用了Huffman编码，它把训练语料中的词当成叶子节点，其在语料中出现的次数当做权值，通过构造响应的Huffman树来对每一个词进行Huffman编码。

### 2.2 N-gram模型

#### 2.2.1 统计语言模型

在自然语言处理中，统计语言模型（Statistic Language Model）是很重要的一环。简单来说，统计语言模型就是计算一个句子的概率的**概率模型**。那么什么是一个句子的概率呢？就是语料库中出现这个句子的概率。

假设$W=w_1^T := (w_1,w_2,\dots, w_T)$表示由$T$个词$w_1,w_2,\dots, w_T$按照顺序构成的一个句子，则该句子的概率模型就是$w_1,w_2,\dots, w_T$的联合概率，即
$$
p(W)=p(w_1^T)=p(w1,w2,\dots,w_T) \\ (利用Bayes链式分解) \\ = p(w1)\cdot p(w_2|w_1)\cdot p(w_3|w_1^2)\cdots p(w_T|w_1^{T-1}) \label{1}
$$
其中$p(w1), p(w_2|w_1),p(w_3|w_1^2)\cdots p(w_T|w_1^{T-1})​$就是**语言模型的参数**，知道这些参数，就知道了句子的出现的概率了。

问题在于**模型参数的个数太多**。假设语料库中词典$D​$的大小为$N​$，那么一维参数有$N​$个，二维参数有$N^2​$个，三维有$N^3​$个，$T​$维有$N^T​$个参数。如果要计算任意长度为$T​$的句子的概率，理论上就需要$N+N^2+N^3+\dots+N^T​$个参数。即使能计算出来，存储这些参数也需要很大的内存开销。

因此我们需要借助N-gram模型来简化参数。

#### 2.2.2 N-gram模型

考虑$p(w_k|w_1^{k-1})(k>1)$的近似计算，利用$Bayes$公式，有
$$
p(w_k|w_1^{k-1})=\frac{p(w_1^k)}{p(w_1^{k-1})} \\ 根据大数定理，当语料库够大时 \\ 
\approx \frac{count(w_1^k)}{count(w_1^{k-1})} \label{2}
$$

其中$w_1^k​$表示句子中从第$1​$个到第$k​$个的词构成的词串，$count(w_1^k)​$表示词串$w_1^k​$在语料中出现的次数。

从公式$\ref{1}$中我们看出：一个词出现的概率和它前面所有的词都相关。如果我们假设这个词只与它前面$n-1$个词相关（做了$n-1$阶**$Markov​$假设**），这样就减少了总参数的个数。

如此一来，公式$\ref{2}​$就变成了如下：
$$
\begin{align*}
p(w_k|w_1^{k-1})\approx p(w_k|w_{k-n+1}^{k-1}) \\ \approx \frac{count(w_1^k)}{count(w_{k-n+1}^{k-1})}
\end{align*}
$$

因此，假设我们选取的n-gram中$n=3​$，那我们只需要统计1、2、3维的参数就行了，即总参数的个数变为$N+N^2+N^3​$。而且同时，对于单个参数的统计也更容易，因为统计时需要匹配的词串更短

#### 2.2.3 N-gram模型的实践

对于参数$n$的选取，我们需要考虑到$计算复杂度$和$模型效果$两个方面。

- 计算复杂度：由于参数的个数是$N+N^2+\dots+N^n$个，因此很明显，$n$越大，计算复杂度越大，而且是指数级增大。
- 模型效果：理论上是$n$越大越好，但是$n$越大的时候，模型效果提升幅度就会越小。例如，但$n$从3到4的效果提升可能就远比不上从2到3的效果提升。

因此，实际工作中，最多的情况是取$n=3$。

此外，我们还需要考虑到**『平滑化』**的问题。因为假如词串在统计时计数为0，即$count(w_{k-n+1}^{k-1})=0$，我们并不能认为$p(w_k|w_1^{k-1})=0$，否则会导致连乘的时候，整个词串的概率都为0.

### 2.3 概率模型函数化

N-gram模型是存储下来所有可能的概率参数，然后计算时对概率进行连乘。

机器学习领域有一种较为通用的做法：**对所考虑的问题建模后，先为其构造一个目标函数进行优化，求得一组最优参数，然后用最优参数对应的模型来预测**。

因此对于N-gram模型来说，**我们不需要存储所有可能的概率参数，而是求解对问题建模后得到的目标函数的最优参数即可**（通常好的建模可以使得最优参数的个数远小于所有概率参数的个数）。

对于统计语言模型来说，我们通常构造的目标函数是『最大似然函数』。
$$
\prod_{w\in C} p(w|Context(w))
$$

上式的意思是：上下文为$Context(w)$时，该词为$w$的概率。
其中

- $C$是语料库(Corpus)
- $Context(w)$是词$w$的上下文(Context)。对于N-gram模型来说，$Context(w_i)=w_{i-n+1}^{i-1}$

实际上由于连乘可能导致概率极小，所以经常采用的是**『最大对数似然』**，即目标函数为：
$$
\mathcal{L}=\sum_{w\in C}log \, p(w|Context(w)) \\ 将条件概率p(w|Context(w))视为关于w和Context(w)的函数 \\  = \sum_{w\in C}log \, F(w,Context(w),\theta)
$$

其中$\theta$是待定参数集。因此一旦对上式进行优化得到最优参数集$\theta^*$之后，$F$也就唯一确定。我们只需要存储最优参数集$\theta^*$，而不需要事先计算并保存所有的概率值了。如果选取合适的方法来构造函数$F$，可以使得$\theta$中参数的个数远小于N-gram模型中参数的个数。

### 2.4 神经概率语言模型

对应2.3节，**神经概率语言模型即用『神经网络』构建『函数$F()$』**。

#### 2.4.1 词向量

首先引入一个概念『词向量』，何为词向量？即对词典$D$中任意词$w$，指定一个固定长度的实值向量$v(w)\in \mathbb{R}^m$。则$v(w)$即称为$w$的词向量，$m$是词向量的长度。

词向量有两种表现形式：

- One-hot Representation：用维度为字典长度的向量表示一个词，仅一个分量为1，其余为0。缺点是容易收到维度灾难的困扰，而且不能很好的刻画词与词之间的关系。
- Distributed Representation：每个词映射为固定长度的短向量。通过刻画两个向量之间的距离来刻画两个向量之间的相似度。


#### 2.4.2 神经概率语言模型的网络结构

![1](https://raw.githubusercontent.com/Dounm/TheFarmOfDounm/master/resources/images/word2vec/1.png)

其中：

- 训练样本$(Content(w),w)$：$w$是语料C中的每一个词，$Content(w)$取为其前面$n-1$个词
- 投影层向量$X_w​$：将该训练样本$(Content(w),w)​$的前$n-1​$个词的『词向量首尾相接』拼接在一起构成$X_w​$。$X_w​$长度为$(n-1)\cdot m​$，$m​$即词向量长度
- 隐藏层向量$Z_w$：$$Z_w=tanh(WX_w+p)$$
- 输出层向量$y_w$：维度为$N=|D|$,即词典$D$中词的个数。$$y_w=Uz_w+q$$

注意：**在对$y_w$做Softmax归一化之后，$y_w$的分量就表示当前词是$w$的概率。**
$$
p(w|Context(w))=\frac{e^{y_{w,i_w}}}{\sum_{i=1}^Ne^{y_{w,i}}} \label{3}
$$


对于该神经网络来说，它的参数有包括：『词向量$v(w)$』以及『神经网络参数$W,p,U,q$』。一旦确定了这些参数，就相当于确定了『函数$F$』的参数，也就相当于知道了参数$p(w|Context(w))$，继而能求得整个句子的概率。

#### 2.4.3 神经概率语言模型的优缺点

相比于N-gram模型来说，神经概率语言模型有如下优点：

1. 词语与词语间的相似度可以通过词向量来体现
2. 基于词向量的模型自带『平滑化』功能，无需额外处理。因为根据公式$\ref{3}$ $p(w|Context(w))$不可能为0。

但同样，神经概率语言模型也有缺点。主要的缺点就是**计算量太大**。各参数量级分别是：

- 投影层节点数为上下文词数量*词向量维度。上下文数量通常不超过$5$，词向量维度在$10^2$量级，
- 隐层节点数在$10^2​$量级，
- 输出层节点数为词典大小，在$10^5$量级。

因此，对于神经概率语言模型来看，主要的计算集中在**『隐层和输出层之间的矩阵运算』**和**『输出层上的Softmax』**归一化运算。但是，考虑到语言模型对语料库中的每一个词$w$都要进行训练，而语料库通常都有$10^6$次方以上的词数，因此上面的计算是无法忍受的。



## 3 【重要】Word2vec相比于神经概率语言模型的改进

注意，前面**神经概率语言模型的目的是得到语言模型（这点和N-gram模型的目的一样），词向量只不过是次要目的（但是也可以求解出词向量）**。

但Word2vec模型主要的目的是计算出**『词向量word embedding』**，因此会导致网络结构进行一些较为奇怪的变化。通常根据什么计算出词向量呢？通常都是通过最大化$p(w|Context(w))$，即对语言模型的概率求最大似然来得到，但这并非是固定的，具体看下面。

### 3.1 优化网络结构

前面说对于神经概率语言模型，缺点主要是计算量太大，集中在：**『隐层和输出层之间的矩阵运算』**和**『输出层上的Softmax归一化运算』**上。因此Word2vec就是针对这两点来优化神经概率语言模型的。

Word2vec的网络机构相对于神经概率语言模型简化了很多，但是正是因为模型简单，因此训练起来更加容易，就能训练更多的数据。而神经概率语言模型结构复杂，导致训练较慢，难以训练更多的数据。

首先，Word2vec选择将输入层到投影层的运算从『拼接』变成『叠加』。也就是说，投影层的节点数不再是上下文词数量*词向量维度，而就是词向量维度。

其次，针对**『隐层和输出层之间的矩阵运算』**，word2vec选择删去隐藏层。变成类似下图所示:
![7](https://raw.githubusercontent.com/Dounm/TheFarmOfDounm/master/resources/images/word2vec/2.png)
（Output Layer和Projection Layer之间的连线没有画完）

就网络结构而言，上图删去了Hidden Layer，而且Projection Layer也由『拼接』变成了『叠加』。同时我假设词向量维度为3维，因此Projection layer只有3个节点；词典中只有4个词，因此输出层只有4个节点，$y_i$代表的就是词典中的词$w_i$。



对于上图中的输出层节点而言，任意节点$y_i$的计算公式如下：
$$
y_i=\sum_{j=1}^3x_jw_{ij} = \vec{w_i}^T\vec{x}
$$

注意，对于神经网络而言，我们的输入层的参数是$Context(w)$中各个词的词向量，那么这个词向量是怎么来的呢？

其实这个词向量就是输出层和映射层之间的参数，即$\vec{w_i}$。对于任意输出层节点$y_i$来说，它与映射层的每个节点都相连接。而映射层节点数就是词向量的维度，所以我们可以将参数$w_{i1},w_{i2},\dots,w_{in}(n是词向量维度)$看作是词$y_i$的词向量。这样一来的话，我们训练神经网络的参数，就相当于训练了每个词的词向量，也就得到了词典中每个词的词向量。
> 我觉着这也是为什么Word2vec要把输入层到映射层的操作由『拼接』变成『叠加』的原因。2.4.2节所说的神经网络的参数包括两个：『词向量$v(w)$』以及『神经网络参数$W,p,U,q$』。然而网络结构变成如上所述的话，**两种参数就合并为一种**，『神经网络参数』就是『词向量』，『词向量』也即是『神经网络参数』。

### 3.2 优化Softmax归一化
为了计算条件概率，我们引入Softmax来归一化：
$$
p(y_i|Context(w))=\frac{exp(y_i)}{\sum_{k=1}^4exp(y_k)} \\ 
= \frac{exp(\vec{w_i}^T\vec{x})}{\sum_{k=1}^4exp(\vec{w_k}^T\vec{x}))} \label{4}
$$

上述式子的计算瓶颈在于分母。分母需要枚举一遍词典中所有的词，而词典中的词的数目在$10^5$的量级。看起来是不多，但是要注意，我们需要对语料库中的每个词进行训练并按照公式$\ref{4}​$计算，而语料库中的词的数量级通常在million甚至billion量级，这样一来的话，训练复杂度就无法接受。

因此，Word2vec提出了两种优化Softmax计算过程的方法，同样也对应着Word2vec的两种框架，即：Hieraichical Softmax和Negative Sampling。

#### 3.2.1 使用Hierarchical Softmax优化
本框架之所以叫『Hierarchical Softmax』，就是它利用了树实现了分层的Softmax，即用树形结构替代了输出层的结构。

之所以使用分层Softmax，是因为它计算起来更加的高效。如论文[^1]中所说：
> A computationally efficient approximation of the full softmax is the hierarchical softmax.

Hierarchical softmax采用的树是二叉树。它将树上的叶子节点分配给词典里的词，而将从树根到叶子节点的路径上的每个非叶子结点都看作是二分类，路径上的二分类概率连乘的结果就是该叶子节点对应的词的概率。

一个full softmax需要一次计算所有的$W$个词，而hierarchical softmax却只需要计算大约$log_2(W)$（即树根到该叶子节点的路径长度）个词，大大减少了计算的复杂度。

实际应用中，Hierarchical Softmax采用是Huffman树而不是其他的二叉树，这是因为Huffman树对于高频词会赋予更短的编码，使得高频词离根节点距离更近，从而使得训练速度加快。

#### 3.2.2 使用Negative Sampling优化

另一种可行的方法是$Noise Contrastive Estimation(NCE)$。$NCE$假定好的模型应该能够通过逻辑回归的方法从数据中区别出噪声。
> NCE posits that a good model should be able to differentiate data from noise by means of logistic regression.

Word2vec采用的Negative Sampling是NCE的一种简化版本，目的是为了提高训练速度以及改善所得词的质量。相比于Hierarchical Softmax，Negative Sampling不再采用Huffman树，而是采用**随机负采样**。

Negative Sampling简单理解起来的话，是这样的。
根据公式$\ref{4}$ ：
$$
p(y_i|Context(w))= \frac{exp(\vec{w_i}^T\vec{x})}{\sum_{k=1}^4exp(\vec{w_k}^T\vec{x}))}
$$

其中$\vec{x}$是$Context(w)$联合构成的词向量，$\vec{w_i}$对应的是$y_i(w_i)$的词向量。

我们要最大化$p(y_i|Context(w))$，就相当于最大化分子，最小化分母。因为两个向量的点积越大，约等于两个向量相似度越高（cosine相似度）。所以，我们就是尽量最大化$\vec{w_i}$和$\vec{x}$的相似度，尽量最小化$\vec{w_k}$和$\vec{x}$相似度。

即**最大化$\vec{x}$与当前词$w_i$的词向量的相似度，最小化$\vec{x}$与非当前词的词向量的相似度**。

我们可以将分子的$(Context(w),w_i)$看做一个正样本，将分母的$(Context(w),w_k)$看做负样本（因为它没在训练数据中出现过）。问题在于，上面公式将词典里的所有词都看做了负样本，因此计算分母太耗时间。所以，我们使用Negative Sampling的思路，每次只从词典里随机选一些word作为当前词$w$的负样本（称为$NEG(w)$），而不是以所有的字典里的其他词作为负样本。

> 其实在做出随机选取负样本的动作之后，我们就已经抛弃了Softmax这个函数所代表的归一化的意思了。也就代表了我们已经不再关注求解**语言模型**的问题，而只关注求解**词向量**的问题。

### 3.3 总结

总结一下上面所说的Word2vec对于经典神经网络概率语言模型的优化的话，我们发现，Word2vec大大简化了网络结构。但是同时也因为网络结构简单所带来的低计算复杂度，所以它可以在更大规模的数据集上计算出非常精确的高维词向量。

这也是值得注意的思路，我们如何权衡模型复杂度与可训练数据之间的关系。模型越复杂，可训练数据的量级越小；而模型越简单，可训练数据的量级越多。