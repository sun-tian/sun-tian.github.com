---
layout:     post           # 使用的布局（不需要改）
title:      Transformer源代码解释之PyTorch篇
subtitle:   Transformer源代码解释之PyTorch篇  #副标题
date:       2021-07-10             # 时间
author:     甜果果                    # 作者
header-img: https://cdn.jsdelivr.net/gh/tian-guo-guo/cdn@1.0/assets/img/post-bg-coffee.jpeg    #背景图片
catalog: true                       # 是否归档
tags:                               #标签
   - nlp


---





# [Transformer源代码解释之PyTorch篇](https://mp.weixin.qq.com/s/M5sXgQ9vjHGsS3fBNOyYSg)  

![图片](https://cdn.jsdelivr.net/gh/tian-guo-guo/cdn@master/assets/picgoimg/20210710170009.png)



**01. 词嵌入**
-------

Transformer 本质上是一种 Encoder，以翻译任务为例，原始数据集是以两种语言组成一行的，在应用时，应是 Encoder 输入源语言序列，Decoder 里面输入需要被转换的语言序列（训练时）。

一个文本常有许多序列组成，常见操作为将序列进行一些预处理（如词切分等）变成列表，一个序列的列表的元素通常为词表中不可切分的最小词，整个文本就是一个大列表，元素为一个一个由序列组成的列表。

如一个序列经过切分后变为 \["am", "##ro", "##zi", "accused", "his", "father"\]，接下来按照它们在词表中对应的索引进行转换，假设结果如 \[23, 94, 13, 41, 27, 96\]。假如整个文本一共 100 个句子，那么就有 100 个列表为它的元素，因为每个序列的长度不一，需要设定最大长度，这里不妨设为 128，那么将整个文本转换为数组之后，形状即为 100 x 128，这就对应着 batch\_size 和 seq\_length。

输入之后，紧接着进行词嵌入处理，词嵌入就是将每一个词用预先训练好的向量进行映射，参数为词表的大小和被映射的向量的维度，通俗来说就是向量里面有多少个数。注意，第一个参数是词表的大小，如果你目前有 4 个词，就填 4，你后面万一进入与这 4 个词不同的词，还需要重新映射，为了统一，一开始的也要重新映射，因此这里填词表总大小。

假如我们打算映射到 512 维（num\_features 或者 embed\_dim），那么，整个文本的形状变为 100 x 128 x 512。接下来举个小例子解释一下：假设我们词表一共有 10 个词，文本里有 2 个句子，每个句子有 4 个词，我们想要把每个词映射到 8 维的向量。于是 2，4，8 对应于 batch\_size, seq\_length, embed\_dim（如果 batch 在第一维的话）。

另外，一般深度学习任务只改变 num\_features，所以讲维度一般是针对最后特征所在的维度。

所有需要的包的导入：

    import torch
    import torch.nn as nn
    from torch.nn.parameter import Parameter
    from torch.nn.init import xavier_uniform_
    from torch.nn.init import constant_
    from torch.nn.init import xavier_normal_
    import torch.nn.functional as F
    from typing import Optional, Tuple, Any
    from typing import List, Optional, Tuple
    import math
    import warnings

  


    X = torch.zeros((2,4),dtype=torch.long)
    embed = nn.Embedding(10,8)
    print(embed(X).shape)
    # torch.Size([2, 4, 8])

  

## **02. 位置编码**

词嵌入之后紧接着就是位置编码，位置编码用以区分不同词以及同词不同特征之间的关系。代码中需要注意：X\_ 只是初始化的矩阵，并不是输入进来的；完成位置编码之后会加一个 dropout。另外，位置编码是最后加上去的，因此输入输出形状不变。

    Tensor = torch.Tensor
    def positional_encoding(X, num_features, dropout_p=0.1, max_len=512) -> Tensor:
        r'''
            给输入加入位置编码
        参数：
            - num_features: 输入进来的维度
            - dropout_p: dropout的概率，当其为非零时执行dropout
            - max_len: 句子的最大长度，默认512
    
        形状：
            - 输入： [batch_size, seq_length, num_features]
            - 输出： [batch_size, seq_length, num_features]
    
        例子：
            >>> X = torch.randn((2,4,10))
            >>> X = positional_encoding(X, 10)
            >>> print(X.shape)
            >>> torch.Size([2, 4, 10])
        '''
    
        dropout = nn.Dropout(dropout_p)
        P = torch.zeros((1,max_len,num_features))
        X_ = torch.arange(max_len,dtype=torch.float32).reshape(-1,1) / torch.pow(
            10000,
            torch.arange(0,num_features,2,dtype=torch.float32) /num_features)
        P[:,:,0::2] = torch.sin(X_)
        P[:,:,1::2] = torch.cos(X_)
        X = X + P[:,:X.shape[1],:].to(X.device)
        return dropout(X)

  

## **03. 多头注意力**

多头注意力大概分为三个部分讲，分别为参数初始化，q,k,v 从何来，遮挡机制，点积注意力。

### **3.1 初始化参数**

query，key，value 是源语言序列（本文记为 src）乘以对应的矩阵得到的，那么，那些矩阵从何而来（注意，因为大部分代码都是从源码中抽离出来的，因而常带有 self 等，最后会呈现组成好的，而行文过程中不会将整个结构呈现出来）：

    if self._qkv_same_embed_dim is False:
        # 初始化前后形状维持不变
        # (seq_length x embed_dim) x (embed_dim x embed_dim) ==> (seq_length x embed_dim)
        self.q_proj_weight = Parameter(torch.empty((embed_dim, embed_dim)))
        self.k_proj_weight = Parameter(torch.empty((embed_dim, self.kdim)))
        self.v_proj_weight = Parameter(torch.empty((embed_dim, self.vdim)))
        self.register_parameter('in_proj_weight', None)
    else:
        self.in_proj_weight = Parameter(torch.empty((3 * embed_dim, embed_dim)))
        self.register_parameter('q_proj_weight', None)
        self.register_parameter('k_proj_weight', None)
        self.register_parameter('v_proj_weight', None)
    
    if bias:
        self.in_proj_bias = Parameter(torch.empty(3 * embed_dim))
    else:
        self.register_parameter('in_proj_bias', None)
    # 后期会将所有头的注意力拼接在一起然后乘上权重矩阵输出
    # out_proj是为了后期准备的
    self.out_proj = nn.Linear(embed_dim, embed_dim, bias=bias)
    self._reset_parameters()

torch.empty 是按照所给的形状形成对应的 tensor，特点是填充的值还未初始化，类比 torch.randn（标准正态分布），这就是一种初始化的方式。在 PyTorch 中，变量类型是 tensor 的话是无法修改值的，而 Parameter() 函数可以看作为一种类型转变函数，将不可改值的 tensor 转换为可训练可修改的模型参数，即与 model.parameters 绑定在一起，register\_parameter 的意思是是否将这个参数放到 model.parameters，None 的意思是没有这个参数。

这里有个 if 判断，用以判断 q,k,v 的最后一维是否一致，若一致，则一个大的权重矩阵全部乘然后分割出来，若不是，则各初始化各的，其实初始化是不会改变原来的形状的（如 ，见注释）。

可以发现最后有一个 \_reset\_parameters() 函数，这个是用来初始化参数数值的。xavier\_uniform 意思是从连续型均匀分布里面随机取样出值来作为初始化的值，xavier\_normal\_ 取样的分布是正态分布。正因为初始化值在训练神经网络的时候很重要，所以才需要这两个函数。

constant\_ 意思是用所给值来填充输入的向量。

另外，在 PyTorch 的源码里，似乎 projection 代表是一种线性变换的意思，in\_proj\_bias 的意思就是一开始的线性变换的偏置。

    def _reset_parameters(self):
        if self._qkv_same_embed_dim:
            xavier_uniform_(self.in_proj_weight)
        else:
            xavier_uniform_(self.q_proj_weight)
            xavier_uniform_(self.k_proj_weight)
            xavier_uniform_(self.v_proj_weight)
        if self.in_proj_bias is not None:
            constant_(self.in_proj_bias, 0.)
            constant_(self.out_proj.bias, 0.)

以上便是参数初始化过程，接下来是进行 query, key 和 value 的赋值。

### **3.2 q,k,v从何来？**

对于`nn.functional.linear`函数，其实就是一个线性变换，与`nn.Linear`不同的是，前者可以提供权重矩阵和偏置，执行

![图片](https://cdn.jsdelivr.net/gh/tian-guo-guo/cdn@master/assets/picgoimg/20210710165501.png)

而后者是可以自由决定输出的维度，因为 linear 函数多层调用且无太大意义，这里省略。

    def _in_projection_packed(
        q: Tensor,
        k: Tensor,
        v: Tensor,
        w: Tensor,
        b: Optional[Tensor] = None,
    ) -> List[Tensor]:
        r"""
        用一个大的权重参数矩阵进行线性变换
    
        参数:
            q, k, v: 对自注意来说，三者都是src；对于seq2seq模型，k和v是一致的tensor。
                     但它们的最后一维(num_features或者叫做embed_dim)都必须保持一致。
            w: 用以线性变换的大矩阵，按照q,k,v的顺序压在一个tensor里面。
            b: 用以线性变换的偏置，按照q,k,v的顺序压在一个tensor里面。
    
        形状:
            输入:
            - q: shape:`(..., E)`，E是词嵌入的维度（下面出现的E均为此意）。
            - k: shape:`(..., E)`
            - v: shape:`(..., E)`
            - w: shape:`(E * 3, E)`
            - b: shape:`E * 3` 
    
            输出:
            - 输出列表 :`[q', k', v']`，q,k,v经过线性变换前后的形状都一致。
        """
        E = q.size(-1)
        # 若为自注意，则q = k = v = src，因此它们的引用变量都是src
        # 即k is v和q is k结果均为True
        # 若为seq2seq，k = v，因而k is v的结果是True
        if k is v:
            if q is k:
                return F.linear(q, w, b).chunk(3, dim=-1)
            else:
                # seq2seq模型
                w_q, w_kv = w.split([E, E * 2])
                if b is None:
                    b_q = b_kv = None
                else:
                    b_q, b_kv = b.split([E, E * 2])
                return (F.linear(q, w_q, b_q),) + F.linear(k, w_kv, b_kv).chunk(2, dim=-1)
        else:
            w_q, w_k, w_v = w.chunk(3)
            if b is None:
                b_q = b_k = b_v = None
            else:
                b_q, b_k, b_v = b.chunk(3)
            return F.linear(q, w_q, b_q), F.linear(k, w_k, b_k), F.linear(v, w_v, b_v)
    
    q, k, v = _in_projection_packed(query, key, value, in_proj_weight, in_proj_bias)

### **3.3 遮挡机制**

对于 attn\_mask 来说，若为 2D，形状如`(L, S)`，L 和 S 分别代表着目标语言和源语言序列长度，若为 3D，形状如`(N * num_heads, L, S)`，N 代表着 batch\_size，num\_heads 代表注意力头的数目。若为 ByteTensor，非 0 的位置会被忽略不做注意力；若为 BoolTensor，True 对应的位置会被忽略；若为数值，则会直接加到 attn\_weights。

因为在 decoder 解码的时候，只能看该位置和它之前的，如果看后面就犯规了，所以需要 attn\_mask 遮挡住。

下面函数直接复制 PyTorch 的，意思是确保不同维度的 mask 形状正确以及不同类型的转换。

    if attn_mask is not None:
        if attn_mask.dtype == torch.uint8:
            warnings.warn("Byte tensor for attn_mask in nn.MultiheadAttention is deprecated. Use bool tensor instead.")
            attn_mask = attn_mask.to(torch.bool)
        else:
            assert attn_mask.is_floating_point() or attn_mask.dtype == torch.bool, \
                f"Only float, byte, and bool types are supported for attn_mask, not {attn_mask.dtype}"
        # 对不同维度的形状判定
        if attn_mask.dim() == 2:
            correct_2d_size = (tgt_len, src_len)
            if attn_mask.shape != correct_2d_size:
                raise RuntimeError(f"The shape of the 2D attn_mask is {attn_mask.shape}, but should be {correct_2d_size}.")
                attn_mask = attn_mask.unsqueeze(0)
        elif attn_mask.dim() == 3:
            correct_3d_size = (bsz * num_heads, tgt_len, src_len)
            if attn_mask.shape != correct_3d_size:
                raise RuntimeError(f"The shape of the 3D attn_mask is {attn_mask.shape}, but should be {correct_3d_size}.")
        else:
            raise RuntimeError(f"attn_mask's dimension {attn_mask.dim()} is not supported")

与`attn_mask`不同的是，`key_padding_mask`是用来遮挡住 key 里面的值，详细来说应该是`<PAD>`，被忽略的情况与 attn\_mask 一致。

    # 将key_padding_mask值改为布尔值
    if key_padding_mask is not None and key_padding_mask.dtype == torch.uint8:
        warnings.warn("Byte tensor for key_padding_mask in nn.MultiheadAttention is deprecated. Use bool tensor instead.")
        key_padding_mask = key_padding_mask.to(torch.bool)

先介绍两个小函数，`logical_or`，输入两个 tensor，并对这两个 tensor 里的值做`逻辑或`运算，只有当两个值均为 0 的时候才为`False`，其他时候均为`True`，另一个是`masked_fill`，输入是一个 mask，和用以填充的值。mask 由 1，0 组成，0 的位置值维持不变，1 的位置用新值填充。

    a = torch.tensor([0,1,10,0],dtype=torch.int8)
    b = torch.tensor([4,0,1,0],dtype=torch.int8)
    print(torch.logical_or(a,b))
    # tensor([ True,  True,  True, False])

  




    r = torch.tensor([[0,0,0,0],[0,0,0,0]])
    mask = torch.tensor([[1,1,1,1],[0,0,0,0]])
    print(r.masked_fill(mask,1))
    # tensor([[1, 1, 1, 1],
    #         [0, 0, 0, 0]])

其实 attn\_mask 和 key\_padding\_mask 有些时候对象是一致的，所以有时候可以合起来看。`-inf`做 softmax 之后值为 0，即被忽略。  

    if key_padding_mask is not None:
        assert key_padding_mask.shape == (bsz, src_len), \
            f"expecting key_padding_mask shape of {(bsz, src_len)}, but got {key_padding_mask.shape}"
        key_padding_mask = key_padding_mask.view(bsz, 1, 1, src_len).   \
            expand(-1, num_heads, -1, -1).reshape(bsz * num_heads, 1, src_len)
        # 若attn_mask为空，直接用key_padding_mask
        if attn_mask is None:
            attn_mask = key_padding_mask
        elif attn_mask.dtype == torch.bool:
            attn_mask = attn_mask.logical_or(key_padding_mask)
        else:
            attn_mask = attn_mask.masked_fill(key_padding_mask, float("-inf"))
    
    # 若attn_mask值是布尔值，则将mask转换为float
    if attn_mask is not None and attn_mask.dtype == torch.bool:
        new_attn_mask = torch.zeros_like(attn_mask, dtype=torch.float)
        new_attn_mask.masked_fill_(attn_mask, float("-inf"))
        attn_mask = new_attn_mask

### **3.4 点积注意力**

    from typing import Optional, Tuple, Any
    def scaled_dot_product_attention(
        q: Tensor,
        k: Tensor,
        v: Tensor,
        attn_mask: Optional[Tensor] = None,
        dropout_p: float = 0.0,
    ) -> Tuple[Tensor, Tensor]:
        r'''
        在query, key, value上计算点积注意力，若有注意力遮盖则使用，并且应用一个概率为dropout_p的dropout
    
        参数：
            - q: shape:`(B, Nt, E)` B代表batch size， Nt是目标语言序列长度，E是嵌入后的特征维度
            - key: shape:`(B, Ns, E)` Ns是源语言序列长度
            - value: shape:`(B, Ns, E)`与key形状一样
            - attn_mask: 要么是3D的tensor，形状为:`(B, Nt, Ns)`或者2D的tensor，形状如:`(Nt, Ns)`
    
            - Output: attention values: shape:`(B, Nt, E)`，与q的形状一致;attention weights: shape:`(B, Nt, Ns)`
    
        例子：
            >>> q = torch.randn((2,3,6))
            >>> k = torch.randn((2,4,6))
            >>> v = torch.randn((2,4,6))
            >>> out = scaled_dot_product_attention(q, k, v)
            >>> out[0].shape, out[1].shape
            >>> torch.Size([2, 3, 6]) torch.Size([2, 3, 4])
        '''
        B, Nt, E = q.shape
        q = q / math.sqrt(E)
        # (B, Nt, E) x (B, E, Ns) -> (B, Nt, Ns)
        attn = torch.bmm(q, k.transpose(-2,-1))
        if attn_mask is not None:
            attn += attn_mask 
        # attn意味着目标序列的每个词对源语言序列做注意力
        attn = F.softmax(attn, dim=-1)
        if dropout_p:
            attn = F.dropout(attn, p=dropout_p)
        # (B, Nt, Ns) x (B, Ns, E) -> (B, Nt, E)
        output = torch.bmm(attn, v)
        return output, attn 

接下来将三个部分连起来：

    def multi_head_attention_forward(
        query: Tensor,
        key: Tensor,
        value: Tensor,
        num_heads: int,
        in_proj_weight: Tensor,
        in_proj_bias: Optional[Tensor],
        dropout_p: float,
        out_proj_weight: Tensor,
        out_proj_bias: Optional[Tensor],
        training: bool = True,
        key_padding_mask: Optional[Tensor] = None,
        need_weights: bool = True,
        attn_mask: Optional[Tensor] = None,
        use_seperate_proj_weight = None,
        q_proj_weight: Optional[Tensor] = None,
        k_proj_weight: Optional[Tensor] = None,
        v_proj_weight: Optional[Tensor] = None,
    ) -> Tuple[Tensor, Optional[Tensor]]:
        r'''
        形状：
            输入：
            - query：`(L, N, E)`
            - key: `(S, N, E)`
            - value: `(S, N, E)`
            - key_padding_mask: `(N, S)`
            - attn_mask: `(L, S)` or `(N * num_heads, L, S)`
            输出：
            - attn_output:`(L, N, E)`
            - attn_output_weights:`(N, L, S)`
        '''
        tgt_len, bsz, embed_dim = query.shape
        src_len, _, _ = key.shape
        head_dim = embed_dim // num_heads
        q, k, v = _in_projection_packed(query, key, value, in_proj_weight, in_proj_bias)
    
        if attn_mask is not None:
            if attn_mask.dtype == torch.uint8:
                warnings.warn("Byte tensor for attn_mask in nn.MultiheadAttention is deprecated. Use bool tensor instead.")
                attn_mask = attn_mask.to(torch.bool)
            else:
                assert attn_mask.is_floating_point() or attn_mask.dtype == torch.bool, \
                    f"Only float, byte, and bool types are supported for attn_mask, not {attn_mask.dtype}"
    
            if attn_mask.dim() == 2:
                correct_2d_size = (tgt_len, src_len)
                if attn_mask.shape != correct_2d_size:
                    raise RuntimeError(f"The shape of the 2D attn_mask is {attn_mask.shape}, but should be {correct_2d_size}.")
                attn_mask = attn_mask.unsqueeze(0)
            elif attn_mask.dim() == 3:
                correct_3d_size = (bsz * num_heads, tgt_len, src_len)
                if attn_mask.shape != correct_3d_size:
                    raise RuntimeError(f"The shape of the 3D attn_mask is {attn_mask.shape}, but should be {correct_3d_size}.")
            else:
                raise RuntimeError(f"attn_mask's dimension {attn_mask.dim()} is not supported")
    
        if key_padding_mask is not None and key_padding_mask.dtype == torch.uint8:
            warnings.warn("Byte tensor for key_padding_mask in nn.MultiheadAttention is deprecated. Use bool tensor instead.")
            key_padding_mask = key_padding_mask.to(torch.bool)
    
        # reshape q,k,v将Batch放在第一维以适合点积注意力
        # 同时为多头机制，将不同的头拼在一起组成一层
        q = q.contiguous().view(tgt_len, bsz * num_heads, head_dim).transpose(0, 1)
        k = k.contiguous().view(-1, bsz * num_heads, head_dim).transpose(0, 1)
        v = v.contiguous().view(-1, bsz * num_heads, head_dim).transpose(0, 1)
        if key_padding_mask is not None:
            assert key_padding_mask.shape == (bsz, src_len), \
                f"expecting key_padding_mask shape of {(bsz, src_len)}, but got {key_padding_mask.shape}"
            key_padding_mask = key_padding_mask.view(bsz, 1, 1, src_len).   \
                expand(-1, num_heads, -1, -1).reshape(bsz * num_heads, 1, src_len)
            if attn_mask is None:
                attn_mask = key_padding_mask
            elif attn_mask.dtype == torch.bool:
                attn_mask = attn_mask.logical_or(key_padding_mask)
            else:
                attn_mask = attn_mask.masked_fill(key_padding_mask, float("-inf"))
        # 若attn_mask值是布尔值，则将mask转换为float
        if attn_mask is not None and attn_mask.dtype == torch.bool:
            new_attn_mask = torch.zeros_like(attn_mask, dtype=torch.float)
            new_attn_mask.masked_fill_(attn_mask, float("-inf"))
            attn_mask = new_attn_mask
    
        # 若training为True时才应用dropout
        if not training:
            dropout_p = 0.0
        attn_output, attn_output_weights = _scaled_dot_product_attention(q, k, v, attn_mask, dropout_p)
        attn_output = attn_output.transpose(0, 1).contiguous().view(tgt_len, bsz, embed_dim)
        attn_output = linear(attn_output, out_proj_weight, out_proj_bias)
        if need_weights:
            # average attention weights over heads
            attn_output_weights = attn_output_weights.view(bsz, num_heads, tgt_len, src_len)
            return attn_output, attn_output_weights.sum(dim=1) / num_heads
        else:
            return attn_output, None

接下来构建 MultiheadAttention 类：

    class MultiheadAttention(nn.Module):
        r'''
        参数：
            embed_dim: 词嵌入的维度
            num_heads: 平行头的数量
            batch_first: 若`True`，则为(batch, seq, feture)，若为`False`，则为(seq, batch, feature)
    
        例子：
            >>> multihead_attn = MultiheadAttention(embed_dim, num_heads)
            >>> attn_output, attn_output_weights = multihead_attn(query, key, value)
        '''
        def __init__(self, embed_dim, num_heads, dropout=0., bias=True,
                     kdim=None, vdim=None, batch_first=False) -> None:
            factory_kwargs = {'device': device, 'dtype': dtype}
            super(MultiheadAttention, self).__init__()
            self.embed_dim = embed_dim
            self.kdim = kdim if kdim is not None else embed_dim
            self.vdim = vdim if vdim is not None else embed_dim
            self._qkv_same_embed_dim = self.kdim == embed_dim and self.vdim == embed_dim
    
            self.num_heads = num_heads
            self.dropout = dropout
            self.batch_first = batch_first
            self.head_dim = embed_dim // num_heads
            assert self.head_dim * num_heads == self.embed_dim, "embed_dim must be divisible by num_heads"
    
            if self._qkv_same_embed_dim is False:
                self.q_proj_weight = Parameter(torch.empty((embed_dim, embed_dim)))
                self.k_proj_weight = Parameter(torch.empty((embed_dim, self.kdim)))
                self.v_proj_weight = Parameter(torch.empty((embed_dim, self.vdim)))
                self.register_parameter('in_proj_weight', None)
            else:
                self.in_proj_weight = Parameter(torch.empty((3 * embed_dim, embed_dim)))
                self.register_parameter('q_proj_weight', None)
                self.register_parameter('k_proj_weight', None)
                self.register_parameter('v_proj_weight', None)
    
            if bias:
                self.in_proj_bias = Parameter(torch.empty(3 * embed_dim))
            else:
                self.register_parameter('in_proj_bias', None)
            self.out_proj = nn.Linear(embed_dim, embed_dim, bias=bias)
    
            self._reset_parameters()
    
        def _reset_parameters(self):
            if self._qkv_same_embed_dim:
                xavier_uniform_(self.in_proj_weight)
            else:
                xavier_uniform_(self.q_proj_weight)
                xavier_uniform_(self.k_proj_weight)
                xavier_uniform_(self.v_proj_weight)
    
            if self.in_proj_bias is not None:
                constant_(self.in_proj_bias, 0.)
                constant_(self.out_proj.bias, 0.)
            if self.bias_k is not None:
                xavier_normal_(self.bias_k)
            if self.bias_v is not None:
                xavier_normal_(self.bias_v)
    
    
        def forward(self, query: Tensor, key: Tensor, value: Tensor, key_padding_mask: Optional[Tensor] = None,
                    need_weights: bool = True, attn_mask: Optional[Tensor] = None) -> Tuple[Tensor, Optional[Tensor]]:
            if self.batch_first:
                query, key, value = [x.transpose(1, 0) for x in (query, key, value)]
    
            if not self._qkv_same_embed_dim:
                attn_output, attn_output_weights = F.multi_head_attention_forward(
                    query, key, value, self.num_heads,
                    self.in_proj_weight, self.in_proj_bias,
                    self.dropout, self.out_proj.weight, self.out_proj.bias,
                    training=self.training,
                    key_padding_mask=key_padding_mask, need_weights=need_weights,
                    attn_mask=attn_mask, use_separate_proj_weight=True,
                    q_proj_weight=self.q_proj_weight, k_proj_weight=self.k_proj_weight,
                    v_proj_weight=self.v_proj_weight)
            else:
                attn_output, attn_output_weights = F.multi_head_attention_forward(
                    query, key, value, self.num_heads,
                    self.in_proj_weight, self.in_proj_bias,
                    self.dropout, self.out_proj.weight, self.out_proj.bias,
                    training=self.training,
                    key_padding_mask=key_padding_mask, need_weights=need_weights,
                    attn_mask=attn_mask)
            if self.batch_first:
                return attn_output.transpose(1, 0), attn_output_weights
            else:
                return attn_output, attn_output_weights

接下来可以实践一下，并且把位置编码加起来，可以发现加入位置编码和进行多头注意力的前后形状都是不会变的。

    # 因为batch_first为False,所以src的shape：`(seq, batch, embed_dim)`
    src = torch.randn((2,4,100))
    src = positional_encoding(src,100,0.1)
    print(src.shape)
    multihead_attn = MultiheadAttention(100, 4, 0.1)
    attn_output, attn_output_weights = multihead_attn(src,src,src)
    print(attn_output.shape, attn_output_weights.shape)
    
    # torch.Size([2, 4, 100])
    # torch.Size([2, 4, 100]) torch.Size([4, 2, 2])

  

## **04. 搭建Transformer**

### **4.1 Encoder Layer**

![图片](https://cdn.jsdelivr.net/gh/tian-guo-guo/cdn@master/assets/picgoimg/20210710165657.png)

  

    图片
    
    class TransformerEncoderLayer(nn.Module):
        r'''
        参数：
            d_model: 词嵌入的维度（必备）
            nhead: 多头注意力中平行头的数目（必备）
            dim_feedforward: 全连接层的神经元的数目，又称经过此层输入的维度（Default = 2048）
            dropout: dropout的概率（Default = 0.1）
            activation: 两个线性层中间的激活函数，默认relu或gelu
            lay_norm_eps: layer normalization中的微小量，防止分母为0（Default = 1e-5）
            batch_first: 若`True`，则为(batch, seq, feture)，若为`False`，则为(seq, batch, feature)（Default：False）
    
        例子：
            >>> encoder_layer = TransformerEncoderLayer(d_model=512, nhead=8)
            >>> src = torch.randn((32, 10, 512))
            >>> out = encoder_layer(src)
        '''
    
        def __init__(self, d_model, nhead, dim_feedforward=2048, dropout=0.1, activation=F.relu,
                     layer_norm_eps=1e-5, batch_first=False) -> None:
            super(TransformerEncoderLayer, self).__init__()
            self.self_attn = MultiheadAttention(d_model, nhead, dropout=dropout, batch_first=batch_first)
            self.linear1 = nn.Linear(d_model, dim_feedforward)
            self.dropout = nn.Dropout(dropout)
            self.linear2 = nn.Linear(dim_feedforward, d_model)
    
            self.norm1 = nn.LayerNorm(d_model, eps=layer_norm_eps)
            self.norm2 = nn.LayerNorm(d_model, eps=layer_norm_eps)
            self.dropout1 = nn.Dropout(dropout)
            self.dropout2 = nn.Dropout(dropout)
            self.activation = activation        
    
    
        def forward(self, src: Tensor, src_mask: Optional[Tensor] = None, src_key_padding_mask: Optional[Tensor] = None) -> Tensor:
            src = positional_encoding(src, src.shape[-1])
            src2 = self.self_attn(src, src, src, attn_mask=src_mask, 
            key_padding_mask=src_key_padding_mask)[0]
            src = src + self.dropout1(src2)
            src = self.norm1(src)
            src2 = self.linear2(self.dropout(self.activation(self.linear1(src))))
            src = src + self.dropout(src2)
            src = self.norm2(src)
            return src

用小例子看一下

    encoder_layer = TransformerEncoderLayer(d_model=512, nhead=8)
    src = torch.randn((32, 10, 512))
    out = encoder_layer(src)
    print(out.shape)
    # torch.Size([32, 10, 512])

  

### **4.2 Encoder**

    class TransformerEncoder(nn.Module):
        r'''
        参数：
            encoder_layer（必备）
            num_layers：encoder_layer的层数（必备）
            norm: 归一化的选择（可选）
    
        例子：
            >>> encoder_layer = TransformerEncoderLayer(d_model=512, nhead=8)
            >>> transformer_encoder = TransformerEncoder(encoder_layer, num_layers=6)
            >>> src = torch.randn((10, 32, 512))
            >>> out = transformer_encoder(src)
        '''
    
        def __init__(self, encoder_layer, num_layers, norm=None):
            super(TransformerEncoder, self).__init__()
            self.layer = encoder_layer
            self.num_layers = num_layers
            self.norm = norm
    
        def forward(self, src: Tensor, mask: Optional[Tensor] = None, src_key_padding_mask: Optional[Tensor] = None) -> Tensor:
            output = positional_encoding(src, src.shape[-1])
            for _ in range(self.num_layers):
                output = self.layer(output, src_mask=mask, src_key_padding_mask=src_key_padding_mask)
    
            if self.norm is not None:
                output = self.norm(output)
    
            return output

用小例子看一下

    encoder_layer = TransformerEncoderLayer(d_model=512, nhead=8)
    transformer_encoder = TransformerEncoder(encoder_layer, num_layers=6)
    src = torch.randn((10, 32, 512))
    out = transformer_encoder(src)
    print(out.shape)
    # torch.Size([10, 32, 512])

  

### **4.3 Decoder Layer:**

    class TransformerDecoderLayer(nn.Module):
        r'''
        参数：
            d_model: 词嵌入的维度（必备）
            nhead: 多头注意力中平行头的数目（必备）
            dim_feedforward: 全连接层的神经元的数目，又称经过此层输入的维度（Default = 2048）
            dropout: dropout的概率（Default = 0.1）
            activation: 两个线性层中间的激活函数，默认relu或gelu
            lay_norm_eps: layer normalization中的微小量，防止分母为0（Default = 1e-5）
            batch_first: 若`True`，则为(batch, seq, feture)，若为`False`，则为(seq, batch, feature)（Default：False）
    
        例子：
            >>> decoder_layer = TransformerDecoderLayer(d_model=512, nhead=8)
            >>> memory = torch.randn((10, 32, 512))
            >>> tgt = torch.randn((20, 32, 512))
            >>> out = decoder_layer(tgt, memory)
        '''
        def __init__(self, d_model, nhead, dim_feedforward=2048, dropout=0.1, activation=F.relu,
                     layer_norm_eps=1e-5, batch_first=False) -> None:
            super(TransformerDecoderLayer, self).__init__()
            self.self_attn = MultiheadAttention(d_model, nhead, dropout=dropout, batch_first=batch_first)
            self.multihead_attn = MultiheadAttention(d_model, nhead, dropout=dropout, batch_first=batch_first)
    
            self.linear1 = nn.Linear(d_model, dim_feedforward)
            self.dropout = nn.Dropout(dropout)
            self.linear2 = nn.Linear(dim_feedforward, d_model)
    
            self.norm1 = nn.LayerNorm(d_model, eps=layer_norm_eps)
            self.norm2 = nn.LayerNorm(d_model, eps=layer_norm_eps)
            self.norm3 = nn.LayerNorm(d_model, eps=layer_norm_eps)
            self.dropout1 = nn.Dropout(dropout)
            self.dropout2 = nn.Dropout(dropout)
            self.dropout3 = nn.Dropout(dropout)
    
            self.activation = activation
    
        def forward(self, tgt: Tensor, memory: Tensor, tgt_mask: Optional[Tensor] = None, 
                    memory_mask: Optional[Tensor] = None,tgt_key_padding_mask: Optional[Tensor] = None, memory_key_padding_mask: Optional[Tensor] = None) -> Tensor:
            r'''
            参数：
                tgt: 目标语言序列（必备）
                memory: 从最后一个encoder_layer跑出的句子（必备）
                tgt_mask: 目标语言序列的mask（可选）
                memory_mask（可选）
                tgt_key_padding_mask（可选）
                memory_key_padding_mask（可选）
            '''
            tgt2 = self.self_attn(tgt, tgt, tgt, attn_mask=tgt_mask,
                                  key_padding_mask=tgt_key_padding_mask)[0]
            tgt = tgt + self.dropout1(tgt2)
            tgt = self.norm1(tgt)
            tgt2 = self.multihead_attn(tgt, memory, memory, attn_mask=memory_mask,
                                       key_padding_mask=memory_key_padding_mask)[0]
            tgt = tgt + self.dropout2(tgt2)
            tgt = self.norm2(tgt)
            tgt2 = self.linear2(self.dropout(self.activation(self.linear1(tgt))))
            tgt = tgt + self.dropout3(tgt2)
            tgt = self.norm3(tgt)
            return tgt

用小例子看一下：

    decoder_layer = nn.TransformerDecoderLayer(d_model=512, nhead=8)
    memory = torch.randn((10, 32, 512))
    tgt = torch.randn((20, 32, 512))
    out = decoder_layer(tgt, memory)
    print(out.shape)
    # torch.Size([20, 32, 512])

### **4.4 Decoder**

    class TransformerDecoder(nn.Module):
        r'''
        参数：
            decoder_layer（必备）
            num_layers: decoder_layer的层数（必备）
            norm: 归一化选择
    
        例子：
            >>> decoder_layer =TransformerDecoderLayer(d_model=512, nhead=8)
            >>> transformer_decoder = TransformerDecoder(decoder_layer, num_layers=6)
            >>> memory = torch.rand(10, 32, 512)
            >>> tgt = torch.rand(20, 32, 512)
            >>> out = transformer_decoder(tgt, memory)
        '''
        def __init__(self, decoder_layer, num_layers, norm=None):
            super(TransformerDecoder, self).__init__()
            self.layer = decoder_layer
            self.num_layers = num_layers
            self.norm = norm
    
        def forward(self, tgt: Tensor, memory: Tensor, tgt_mask: Optional[Tensor] = None,
                    memory_mask: Optional[Tensor] = None, tgt_key_padding_mask: Optional[Tensor] = None,
                    memory_key_padding_mask: Optional[Tensor] = None) -> Tensor:
            output = tgt
            for _ in range(self.num_layers):
                output = self.layer(output, memory, tgt_mask=tgt_mask,
                             memory_mask=memory_mask,
                             tgt_key_padding_mask=tgt_key_padding_mask,
                             memory_key_padding_mask=memory_key_padding_mask)
            if self.norm is not None:
                output = self.norm(output)
    
            return output

用小例子看一下：

    decoder_layer =TransformerDecoderLayer(d_model=512, nhead=8)
    transformer_decoder = TransformerDecoder(decoder_layer, num_layers=6)
    memory = torch.rand(10, 32, 512)
    tgt = torch.rand(20, 32, 512)
    out = transformer_decoder(tgt, memory)
    print(out.shape)
    # torch.Size([20, 32, 512])

总结一下，其实经过位置编码，多头注意力，Encoder Layer 和 Decoder Layer 形状不会变的，而 Encoder 和 Decoder 分别与 src 和 tgt 形状一致。

### **4.5 Transformer**

    class Transformer(nn.Module):
        r'''
        参数：
            d_model: 词嵌入的维度（必备）（Default=512）
            nhead: 多头注意力中平行头的数目（必备）（Default=8）
            num_encoder_layers:编码层层数（Default=8）
            num_decoder_layers:解码层层数（Default=8）
            dim_feedforward: 全连接层的神经元的数目，又称经过此层输入的维度（Default = 2048）
            dropout: dropout的概率（Default = 0.1）
            activation: 两个线性层中间的激活函数，默认relu或gelu
            custom_encoder: 自定义encoder（Default=None）
            custom_decoder: 自定义decoder（Default=None）
            lay_norm_eps: layer normalization中的微小量，防止分母为0（Default = 1e-5）
            batch_first: 若`True`，则为(batch, seq, feture)，若为`False`，则为(seq, batch, feature)（Default：False）
    
        例子：
            >>> transformer_model = Transformer(nhead=16, num_encoder_layers=12)
            >>> src = torch.rand((10, 32, 512))
            >>> tgt = torch.rand((20, 32, 512))
            >>> out = transformer_model(src, tgt)
        '''
        def __init__(self, d_model: int = 512, nhead: int = 8, num_encoder_layers: int = 6,
                     num_decoder_layers: int = 6, dim_feedforward: int = 2048, dropout: float = 0.1,
                     activation = F.relu, custom_encoder: Optional[Any] = None, custom_decoder: Optional[Any] = None,
                     layer_norm_eps: float = 1e-5, batch_first: bool = False) -> None:
            super(Transformer, self).__init__()
            if custom_encoder is not None:
                self.encoder = custom_encoder
            else:
                encoder_layer = TransformerEncoderLayer(d_model, nhead, dim_feedforward, dropout,
                                                        activation, layer_norm_eps, batch_first)
                encoder_norm = nn.LayerNorm(d_model, eps=layer_norm_eps)
                self.encoder = TransformerEncoder(encoder_layer, num_encoder_layers)
    
            if custom_decoder is not None:
                self.decoder = custom_decoder
            else:
                decoder_layer = TransformerDecoderLayer(d_model, nhead, dim_feedforward, dropout,
                                                        activation, layer_norm_eps, batch_first)
                decoder_norm = nn.LayerNorm(d_model, eps=layer_norm_eps)
                self.decoder = TransformerDecoder(decoder_layer, num_decoder_layers, decoder_norm)
    
            self._reset_parameters()
    
            self.d_model = d_model
            self.nhead = nhead
    
            self.batch_first = batch_first
    
        def forward(self, src: Tensor, tgt: Tensor, src_mask: Optional[Tensor] = None, tgt_mask: Optional[Tensor] = None,
                    memory_mask: Optional[Tensor] = None, src_key_padding_mask: Optional[Tensor] = None,
                    tgt_key_padding_mask: Optional[Tensor] = None, memory_key_padding_mask: Optional[Tensor] = None) -> Tensor:
            r'''
            参数：
                src: 源语言序列（送入Encoder）（必备）
                tgt: 目标语言序列（送入Decoder）（必备）
                src_mask: （可选)
                tgt_mask: （可选）
                memory_mask: （可选）
                src_key_padding_mask: （可选）
                tgt_key_padding_mask: （可选）
                memory_key_padding_mask: （可选）
    
            形状：
                - src: shape:`(S, N, E)`, `(N, S, E)` if batch_first.
                - tgt: shape:`(T, N, E)`, `(N, T, E)` if batch_first.
                - src_mask: shape:`(S, S)`.
                - tgt_mask: shape:`(T, T)`.
                - memory_mask: shape:`(T, S)`.
                - src_key_padding_mask: shape:`(N, S)`.
                - tgt_key_padding_mask: shape:`(N, T)`.
                - memory_key_padding_mask: shape:`(N, S)`.
    
                [src/tgt/memory]_mask确保有些位置不被看到，如做decode的时候，只能看该位置及其以前的，而不能看后面的。
                若为ByteTensor，非0的位置会被忽略不做注意力；若为BoolTensor，True对应的位置会被忽略；
                若为数值，则会直接加到attn_weights
    
                [src/tgt/memory]_key_padding_mask 使得key里面的某些元素不参与attention计算，三种情况同上
    
                - output: shape:`(T, N, E)`, `(N, T, E)` if batch_first.
    
            注意：
                src和tgt的最后一维需要等于d_model，batch的那一维需要相等
    
            例子:
                >>> output = transformer_model(src, tgt, src_mask=src_mask, tgt_mask=tgt_mask)
            '''
            memory = self.encoder(src, mask=src_mask, src_key_padding_mask=src_key_padding_mask)
            output = self.decoder(tgt, memory, tgt_mask=tgt_mask, memory_mask=memory_mask,
                                  tgt_key_padding_mask=tgt_key_padding_mask,
                                  memory_key_padding_mask=memory_key_padding_mask)
            return output
    
        def generate_square_subsequent_mask(self, sz: int) -> Tensor:
            r'''产生关于序列的mask，被遮住的区域赋值`-inf`，未被遮住的区域赋值为`0`'''
            mask = (torch.triu(torch.ones(sz, sz)) == 1).transpose(0, 1)
            mask = mask.float().masked_fill(mask == 0, float('-inf')).masked_fill(mask == 1, float(0.0))
            return mask
    
        def _reset_parameters(self):
            r'''用正态分布初始化参数'''
            for p in self.parameters():
                if p.dim() > 1:
                    xavier_uniform_(p)

用个小例子侧试一下：  

    transformer_model = Transformer(nhead=16, num_encoder_layers=12)
    src = torch.rand((10, 32, 512))
    tgt = torch.rand((20, 32, 512))
    out = transformer_model(src, tgt)
    print(out.shape)
    # torch.Size([20, 32, 512])

到此为止，PyTorch 的 Transformer 库我已经全部实现，相比于官方的版本，我自己手写的这个少了较多的判定语句，所以使用时请确保自己看了我的教程，目明白基本的输入输出，有些不必要的调用，打包均已省略，另有一些不必要的函数已经全部自己重写写过。

另外，所有代码均经过测试，若有问题，定是你的问题，完整版代码：

https://github.com/sherlcok314159/ML/blob/main/nlp/models/functions.py

  

## 完整代码

```python
import math
from typing import Optional, Tuple, Any
from typing import List, Optional, Tuple
import torch.nn as nn
import torch
import torch.nn.functional as F
from torch.nn.parameter import Parameter
from torch.nn.init import constant_, xavier_normal_
from torch.nn.init import xavier_uniform_
import warnings


Tensor = torch.Tensor
def positional_encoding(X, num_features, dropout_p=0.0, max_len=512) -> Tensor:
    r'''
        给输入加入位置编码
    参数：
        - num_features: 输入进来的维度
        - dropout_p: dropout的概率，当其为非零元素时执行dropout
        - max_len: 句子的最大长度，默认512
    
    形状：
        - 输入： [batch_size, seq_length, num_features]
        - 输出： [batch_size, seq_length, num_features]
    例子：
        >>> X = torch.randn((2,4,10))
        >>> X = positional_encoding(X, 10)
        >>> print(X.shape)
        >>> torch.Size([2, 4, 10])
    '''

    dropout = nn.Dropout(dropout_p)
    P = torch.zeros((1,max_len,num_features))
    X_ = torch.arange(max_len,dtype=torch.float32).reshape(-1,1) / torch.pow(
        10000,
        torch.arange(0,num_features,2,dtype=torch.float32) /num_features)
    P[:,:,0::2] = torch.sin(X_)
    P[:,:,1::2] = torch.cos(X_)
    X = X + P[:,:X.shape[1],:].to(X.device)
    return dropout(X)

def _in_projection_packed(
    q: Tensor,
    k: Tensor,
    v: Tensor,
    w: Tensor,
    b: Optional[Tensor] = None,
) -> List[Tensor]:
    r"""
    用一个大的权重参数矩阵进行线性变换
    参数:
        q, k, v: 对自注意来说，三者都是src；对于seq2seq模型，k和v是一致的tensor。
                 但它们的最后一维(num_features或者叫做embed_dim)都必须保持一致。
        w: 用以线性变换的大矩阵，按照q,k,v的顺序压在一个tensor里面。
        b: 用以线性变换的偏置，按照q,k,v的顺序压在一个tensor里面。
    形状:
        输入:
        - q: shape:`(..., E)`，E是词嵌入的维度（下面出现的E均为此意）。
        - k: shape:`(..., E)`
        - v: shape:`(..., E)`
        - w: shape:`(E * 3, E)`
        - b: shape:`E * 3` 
        输出:
        - 输出列表 :`[q', k', v']`，q,k,v经过线性变换前后的形状都一致。
    """
    E = q.size(-1)
    # 若为自注意，则q = k = v = src，因此它们的引用变量都是src
    # 即k is v和q is k结果均为True
    # 若为seq2seq，k = v，因而k is v的结果是True
    if k is v:
        if q is k:
            # 自注意
            return nn.functional.linear(q, w, b).chunk(3, dim=-1)
        else:
            # seq2seq模型
            w_q, w_kv = w.split([E, E * 2])
            if b is None:
                b_q = b_kv = None
            else:
                b_q, b_kv = b.split([E, E * 2])
            return (nn.functional.linear(q, w_q, b_q),) + nn.functional.linear(k, w_kv, b_kv).chunk(2, dim=-1)
    else:
        w_q, w_k, w_v = w.chunk(3)
        if b is None:
            b_q = b_k = b_v = None
        else:
            b_q, b_k, b_v = b.chunk(3)
        return nn.functional.linear(q, w_q, b_q), nn.functional.linear(k, w_k, b_k), nn.functional.linear(v, w_v, b_v)

def _scaled_dot_product_attention(
    q: Tensor,
    k: Tensor,
    v: Tensor,
    attn_mask: Optional[Tensor] = None,
    dropout_p: float = 0.0,
) -> Tuple[Tensor, Tensor]:
    r'''
    在query, key, value上计算点积注意力，若有注意力遮盖则使用，并且应用一个概率为dropout_p的dropout
    参数：
        - q: shape:`(B, Nt, E)` B代表batch size， Nt是目标语言序列长度，E是嵌入后的特征维度
        - key: shape:`(B, Ns, E)` Ns是源语言序列长度
        - value: shape:`(B, Ns, E)`与key形状一样
        - attn_mask: 要么是3D的tensor，形状为:`(B, Nt, Ns)`或者2D的tensor，形状如:`(Nt, Ns)`
        - Output: attention values: shape:`(B, Nt, E)`，与q的形状一致;attention weights: shape:`(B, Nt, Ns)`
    
    例子：
        >>> q = torch.randn((2,3,6))
        >>> k = torch.randn((2,4,6))
        >>> v = torch.randn((2,4,6))
        >>> out = scaled_dot_product_attention(q, k, v)
        >>> out[0].shape, out[1].shape
        >>> torch.Size([2, 3, 6]) torch.Size([2, 3, 4])
    '''
    B, Nt, E = q.shape
    q = q / math.sqrt(E)
    # (B, Nt, E) x (B, E, Ns) -> (B, Nt, Ns)
    attn = torch.bmm(q, k.transpose(-2, -1))
    if attn_mask is not None:
        attn += attn_mask
    # attn意味着目标序列的每个词对源语言序列做注意力
    attn = nn.functional.softmax(attn, dim=-1)
    if dropout_p > 0.0:
        attn = nn.functional.dropout(attn, p=dropout_p)
    # (B, Nt, Ns) x (B, Ns, E) -> (B, Nt, E)
    output = torch.bmm(attn, v)
    return output, attn


def multi_head_attention_forward(
    query: Tensor,
    key: Tensor,
    value: Tensor,
    num_heads: int,
    in_proj_weight: Tensor,
    in_proj_bias: Optional[Tensor],
    dropout_p: float,
    out_proj_weight: Tensor,
    out_proj_bias: Optional[Tensor],
    training: bool = True,
    key_padding_mask: Optional[Tensor] = None,
    need_weights: bool = True,
    attn_mask: Optional[Tensor] = None,
    use_separate_proj_weight: bool = False,
    q_proj_weight: Optional[Tensor] = None,
    k_proj_weight: Optional[Tensor] = None,
    v_proj_weight: Optional[Tensor] = None,
) -> Tuple[Tensor, Optional[Tensor]]:
    r'''
    形状：
        输入：
        - query：`(L, N, E)`
        - key: `(S, N, E)`
        - value: `(S, N, E)`
        - key_padding_mask: `(N, S)`
        - attn_mask: `(L, S)` or `(N * num_heads, L, S)`
        输出：
        - attn_output:`(L, N, E)`
        - attn_output_weights:`(N, L, S)`
    '''

    tgt_len, bsz, embed_dim = query.shape
    src_len, _, _ = key.shape
    head_dim = embed_dim // num_heads
    q, k, v = _in_projection_packed(query, key, value, in_proj_weight, in_proj_bias)

    if attn_mask is not None:
        if attn_mask.dtype == torch.uint8:
            warnings.warn("Byte tensor for attn_mask in nn.MultiheadAttention is deprecated. Use bool tensor instead.")
            attn_mask = attn_mask.to(torch.bool)
        else:
            assert attn_mask.is_floating_point() or attn_mask.dtype == torch.bool, \
                f"Only float, byte, and bool types are supported for attn_mask, not {attn_mask.dtype}"

        if attn_mask.dim() == 2:
            correct_2d_size = (tgt_len, src_len)
            if attn_mask.shape != correct_2d_size:
                raise RuntimeError(f"The shape of the 2D attn_mask is {attn_mask.shape}, but should be {correct_2d_size}.")
            attn_mask = attn_mask.unsqueeze(0)
        elif attn_mask.dim() == 3:
            correct_3d_size = (bsz * num_heads, tgt_len, src_len)
            if attn_mask.shape != correct_3d_size:
                raise RuntimeError(f"The shape of the 3D attn_mask is {attn_mask.shape}, but should be {correct_3d_size}.")
        else:
            raise RuntimeError(f"attn_mask's dimension {attn_mask.dim()} is not supported")

    if key_padding_mask is not None and key_padding_mask.dtype == torch.uint8:
        warnings.warn("Byte tensor for key_padding_mask in nn.MultiheadAttention is deprecated. Use bool tensor instead.")
        key_padding_mask = key_padding_mask.to(torch.bool)
    
    # reshape q,k,v将Batch放在第一维以适合点积注意力
    # 同时为多头机制，将不同的头拼在一起组成一层
    q = q.contiguous().view(tgt_len, bsz * num_heads, head_dim).transpose(0, 1)
    k = k.contiguous().view(-1, bsz * num_heads, head_dim).transpose(0, 1)
    v = v.contiguous().view(-1, bsz * num_heads, head_dim).transpose(0, 1)
    if key_padding_mask is not None:
        assert key_padding_mask.shape == (bsz, src_len), \
            f"expecting key_padding_mask shape of {(bsz, src_len)}, but got {key_padding_mask.shape}"
        key_padding_mask = key_padding_mask.view(bsz, 1, 1, src_len).   \
            expand(-1, num_heads, -1, -1).reshape(bsz * num_heads, 1, src_len)
        if attn_mask is None:
            attn_mask = key_padding_mask
        elif attn_mask.dtype == torch.bool:
            attn_mask = attn_mask.logical_or(key_padding_mask)
        else:
            attn_mask = attn_mask.masked_fill(key_padding_mask, float("-inf"))
    # 若attn_mask值是布尔值，则将mask转换为float
    if attn_mask is not None and attn_mask.dtype == torch.bool:
        new_attn_mask = torch.zeros_like(attn_mask, dtype=torch.float)
        new_attn_mask.masked_fill_(attn_mask, float("-inf"))
        attn_mask = new_attn_mask

    # 若training为True时才应用dropout
    if not training:
        dropout_p = 0.0
    attn_output, attn_output_weights = _scaled_dot_product_attention(q, k, v, attn_mask, dropout_p)
    attn_output = attn_output.transpose(0, 1).contiguous().view(tgt_len, bsz, embed_dim)
    attn_output = nn.functional.linear(attn_output, out_proj_weight, out_proj_bias)
    if need_weights:
        # average attention weights over heads
        attn_output_weights = attn_output_weights.view(bsz, num_heads, tgt_len, src_len)
        return attn_output, attn_output_weights.sum(dim=1) / num_heads
    else:
        return attn_output, None



class MultiheadAttention(nn.Module):
    r'''
    参数：
        embed_dim: 词嵌入的维度
        num_heads: 平行头的数量
        batch_first: 若`True`，则为(batch, seq, feture)，若为`False`，则为(seq, batch, feature)
    
    例子：
        >>> multihead_attn = nn.MultiheadAttention(embed_dim, num_heads)
        >>> attn_output, attn_output_weights = multihead_attn(query, key, value)
    '''
    def __init__(self, embed_dim, num_heads, dropout=0., bias=True, kdim=None, vdim=None,
                 batch_first=False) -> None:
        super(MultiheadAttention, self).__init__()
        self.embed_dim = embed_dim
        self.kdim = kdim if kdim is not None else embed_dim
        self.vdim = vdim if vdim is not None else embed_dim
        self._qkv_same_embed_dim = self.kdim == embed_dim and self.vdim == embed_dim

        self.num_heads = num_heads
        self.dropout = dropout
        self.batch_first = batch_first
        self.head_dim = embed_dim // num_heads
        assert self.head_dim * num_heads == self.embed_dim, "embed_dim must be divisible by num_heads"

        if self._qkv_same_embed_dim is False:
            self.q_proj_weight = Parameter(torch.empty((embed_dim, embed_dim)))
            self.k_proj_weight = Parameter(torch.empty((embed_dim, self.kdim)))
            self.v_proj_weight = Parameter(torch.empty((embed_dim, self.vdim)))
            self.register_parameter('in_proj_weight', None)
        else:
            self.in_proj_weight = Parameter(torch.empty((3 * embed_dim, embed_dim)))
            self.register_parameter('q_proj_weight', None)
            self.register_parameter('k_proj_weight', None)
            self.register_parameter('v_proj_weight', None)

        if bias:
            self.in_proj_bias = Parameter(torch.empty(3 * embed_dim))
        else:
            self.register_parameter('in_proj_bias', None)
        self.out_proj = nn.Linear(embed_dim, embed_dim, bias=bias)
        self._reset_parameters()

    def _reset_parameters(self):
        if self._qkv_same_embed_dim:
            xavier_uniform_(self.in_proj_weight)
        else:
            xavier_uniform_(self.q_proj_weight)
            xavier_uniform_(self.k_proj_weight)
            xavier_uniform_(self.v_proj_weight)

        if self.in_proj_bias is not None:
            constant_(self.in_proj_bias, 0.)
            constant_(self.out_proj.bias, 0.)

    def forward(self, query: Tensor, key: Tensor, value: Tensor, key_padding_mask: Optional[Tensor] = None,
                need_weights: bool = True, attn_mask: Optional[Tensor] = None) -> Tuple[Tensor, Optional[Tensor]]:
        if self.batch_first:
            query, key, value = [x.transpose(1, 0) for x in (query, key, value)]

        if not self._qkv_same_embed_dim:
            attn_output, attn_output_weights = multi_head_attention_forward(
                query, key, value, self.num_heads,
                self.in_proj_weight, self.in_proj_bias,
                self.dropout, self.out_proj.weight, self.out_proj.bias,
                key_padding_mask=key_padding_mask, need_weights=need_weights,
                attn_mask=attn_mask, use_separate_proj_weight=True,
                q_proj_weight=self.q_proj_weight, k_proj_weight=self.k_proj_weight,
                v_proj_weight=self.v_proj_weight)
        else:
            attn_output, attn_output_weights = multi_head_attention_forward(
                query, key, value,self.num_heads,
                self.in_proj_weight, self.in_proj_bias,
                self.dropout, self.out_proj.weight, self.out_proj.bias,
                key_padding_mask=key_padding_mask, need_weights=need_weights,
                attn_mask=attn_mask)
        if self.batch_first:
            return attn_output.transpose(1, 0), attn_output_weights
        else:
            return attn_output, attn_output_weights

# src = torch.randn((2,4,100))
# src = positional_encoding(src,100,0.1)
# print(src.shape)
# multihead_attn = MultiheadAttention(100, 4, 0.1)
# attn_output, attn_output_weights = multihead_attn(src,src,src)
# print(attn_output.shape, attn_output_weights.shape)



class TransformerEncoderLayer(nn.Module):
    r'''
    参数：
        d_model: 词嵌入的维度
        nhead: 多头注意力中平行头的数目
        dim_feedforward: 全连接层的神经元的数目，又称经过此层输入的维度（Default = 2048）
        dropout: dropout的概率
        activation: 两个线性层中间的激活函数，默认relu或gelu
        lay_norm_eps: layer normalization中的微小量，防止分母为0
        batch_first: 若`True`，则为(batch, seq, feture)，若为`False`，则为(seq, batch, feature)
    例子：
        >>> encoder_layer = nn.TransformerEncoderLayer(d_model=512, nhead=8)
        >>> src = torch.randn((32, 10, 512))
        >>> out = encoder_layer(src)
    '''

    def __init__(self, d_model, nhead, dim_feedforward=2048, dropout=0.1, activation=F.relu,
                 layer_norm_eps=1e-5, batch_first=False) -> None:
        super(TransformerEncoderLayer, self).__init__()
        self.self_attn = MultiheadAttention(d_model, nhead, dropout=dropout, batch_first=batch_first)
        self.linear1 = nn.Linear(d_model, dim_feedforward)
        self.dropout = nn.Dropout(dropout)
        self.linear2 = nn.Linear(dim_feedforward, d_model)

        self.activation = activation
        self.norm1 = nn.LayerNorm(d_model, eps=layer_norm_eps)
        self.norm2 = nn.LayerNorm(d_model, eps=layer_norm_eps)
        self.dropout1 = nn.Dropout(dropout)
        self.dropout2 = nn.Dropout(dropout)


    def forward(self, src: Tensor, src_mask: Optional[Tensor] = None, src_key_padding_mask: Optional[Tensor] = None) -> Tensor:
        src2 = self.self_attn(src, src, src, attn_mask=src_mask, 
        key_padding_mask=src_key_padding_mask)[0]
        src = src + self.dropout1(src2)
        src = self.norm1(src)
        src2 = self.linear2(self.dropout(self.activation(self.linear1(src))))
        src = src + self.dropout(src2)
        src = self.norm2(src)
        return src


# encoder_layer = TransformerEncoderLayer(d_model=512, nhead=8)
# src = torch.randn((32, 10, 512))
# out = encoder_layer(src)
# print(out.shape)
# torch.Size([32, 10, 512])

class TransformerDecoderLayer(nn.Module):
    r'''
    参数：
        d_model: 词嵌入的维度（必备）
        nhead: 多头注意力中平行头的数目（必备）
        dim_feedforward: 全连接层的神经元的数目，又称经过此层输入的维度（Default = 2048）
        dropout: dropout的概率（Default = 0.1）
        activation: 两个线性层中间的激活函数，默认relu或gelu
        lay_norm_eps: layer normalization中的微小量，防止分母为0（Default = 1e-5）
        batch_first: 若`True`，则为(batch, seq, feture)，若为`False`，则为(seq, batch, feature)（Default：False）
    
    例子：
        >>> decoder_layer = nn.TransformerDecoderLayer(d_model=512, nhead=8)
        >>> memory = torch.randn((10, 32, 512))
        >>> tgt = torch.randn((20, 32, 512))
        >>> out = decoder_layer(tgt, memory)
    '''
    def __init__(self, d_model, nhead, dim_feedforward=2048, dropout=0.1, activation=F.relu,
                 layer_norm_eps=1e-5, batch_first=False) -> None:
        super(TransformerDecoderLayer, self).__init__()
        self.self_attn = MultiheadAttention(d_model, nhead, dropout=dropout, batch_first=batch_first)
        self.multihead_attn = MultiheadAttention(d_model, nhead, dropout=dropout, batch_first=batch_first)

        self.linear1 = nn.Linear(d_model, dim_feedforward)
        self.dropout = nn.Dropout(dropout)
        self.linear2 = nn.Linear(dim_feedforward, d_model)

        self.norm1 = nn.LayerNorm(d_model, eps=layer_norm_eps)
        self.norm2 = nn.LayerNorm(d_model, eps=layer_norm_eps)
        self.norm3 = nn.LayerNorm(d_model, eps=layer_norm_eps)
        self.dropout1 = nn.Dropout(dropout)
        self.dropout2 = nn.Dropout(dropout)
        self.dropout3 = nn.Dropout(dropout)

        self.activation = activation

    def forward(self, tgt: Tensor, memory: Tensor, tgt_mask: Optional[Tensor] = None, 
                memory_mask: Optional[Tensor] = None,tgt_key_padding_mask: Optional[Tensor] = None, memory_key_padding_mask: Optional[Tensor] = None) -> Tensor:
        r'''
        参数：
            tgt: 目标语言序列（必备）
            memory: 从最后一个encoder_layer跑出的句子（必备）
            tgt_mask: 目标语言序列的mask（可选）
            memory_mask（可选）
            tgt_key_padding_mask（可选）
            memory_key_padding_mask（可选）
        '''
        tgt2 = self.self_attn(tgt, tgt, tgt, attn_mask=tgt_mask,
                              key_padding_mask=tgt_key_padding_mask)[0]
        tgt = tgt + self.dropout1(tgt2)
        tgt = self.norm1(tgt)
        tgt2 = self.multihead_attn(tgt, memory, memory, attn_mask=memory_mask,
                                   key_padding_mask=memory_key_padding_mask)[0]
        tgt = tgt + self.dropout2(tgt2)
        tgt = self.norm2(tgt)
        tgt2 = self.linear2(self.dropout(self.activation(self.linear1(tgt))))
        tgt = tgt + self.dropout3(tgt2)
        tgt = self.norm3(tgt)
        return tgt


# decoder_layer = nn.TransformerDecoderLayer(d_model=512, nhead=8)
# memory = torch.randn((10, 32, 512))
# tgt = torch.randn((20, 32, 512))
# out = decoder_layer(tgt, memory)
# print(out.shape)
# torch.Size([20, 32, 512])


class TransformerEncoder(nn.Module):
    r'''
    参数：
        encoder_layer
        num_layers： encoder_layer的层数
        norm: 归一化的选择
    
    例子：
        >>> encoder_layer = nn.TransformerEncoderLayer(d_model=512, nhead=8)
        >>> transformer_encoder = nn.TransformerEncoder(encoder_layer, num_layers=6)
        >>> src = torch.randn((10, 32, 512))
        >>> out = transformer_encoder(src)
    '''

    def __init__(self, encoder_layer, num_layers, norm=None):
        super(TransformerEncoder, self).__init__()
        self.layer = encoder_layer
        self.num_layers = num_layers
        self.norm = norm
    
    def forward(self, src: Tensor, mask: Optional[Tensor] = None, src_key_padding_mask: Optional[Tensor] = None) -> Tensor:
        output = positional_encoding(src, src.shape[-1])
        for _ in range(self.num_layers):
            output = self.layer(output, src_mask=mask, src_key_padding_mask=src_key_padding_mask)
        
        if self.norm is not None:
            output = self.norm(output)
        
        return output

# encoder_layer = TransformerEncoderLayer(d_model=512, nhead=8)
# transformer_encoder = nn.TransformerEncoder(encoder_layer, num_layers=6)
# src = torch.randn((10, 32, 512))
# out = transformer_encoder(src)
# print(out.shape)

class TransformerDecoder(nn.Module):
    r'''
    参数：
        decoder_layer（必备）
        num_layers: decoder_layer的层数（必备）
        norm: 归一化选择
    
    例子：
        >>> decoder_layer =TransformerDecoderLayer(d_model=512, nhead=8)
        >>> transformer_decoder = TransformerDecoder(decoder_layer, num_layers=6)
        >>> memory = torch.rand(10, 32, 512)
        >>> tgt = torch.rand(20, 32, 512)
        >>> out = transformer_decoder(tgt, memory)
    '''
    def __init__(self, decoder_layer, num_layers, norm=None):
        super(TransformerDecoder, self).__init__()
        self.layer = decoder_layer
        self.num_layers = num_layers
        self.norm = norm
    
    def forward(self, tgt: Tensor, memory: Tensor, tgt_mask: Optional[Tensor] = None,
                memory_mask: Optional[Tensor] = None, tgt_key_padding_mask: Optional[Tensor] = None,
                memory_key_padding_mask: Optional[Tensor] = None) -> Tensor:
        output = tgt
        for _ in range(self.num_layers):
            output = self.layer(output, memory, tgt_mask=tgt_mask,
                         memory_mask=memory_mask,
                         tgt_key_padding_mask=tgt_key_padding_mask,
                         memory_key_padding_mask=memory_key_padding_mask)
        if self.norm is not None:
            output = self.norm(output)

        return output

# decoder_layer =TransformerDecoderLayer(d_model=512, nhead=8)
# transformer_decoder = TransformerDecoder(decoder_layer, num_layers=6)
# memory = torch.rand(10, 32, 512)
# tgt = torch.rand(20, 32, 512)
# out = transformer_decoder(tgt, memory)
# print(out.shape)
# torch.Size([20, 32, 512])

class Transformer(nn.Module):
    r'''
    参数：
        d_model: 词嵌入的维度（必备）（Default=512）
        nhead: 多头注意力中平行头的数目（必备）（Default=8）
        num_encoder_layers:编码层层数（Default=8）
        num_decoder_layers:解码层层数（Default=8）
        dim_feedforward: 全连接层的神经元的数目，又称经过此层输入的维度（Default = 2048）
        dropout: dropout的概率（Default = 0.1）
        activation: 两个线性层中间的激活函数，默认relu或gelu
        custom_encoder: 自定义encoder（Default=None）
        custom_decoder: 自定义decoder（Default=None）
        lay_norm_eps: layer normalization中的微小量，防止分母为0（Default = 1e-5）
        batch_first: 若`True`，则为(batch, seq, feture)，若为`False`，则为(seq, batch, feature)（Default：False）
    
    例子：
        >>> transformer_model = Transformer(nhead=16, num_encoder_layers=12)
        >>> src = torch.rand((10, 32, 512))
        >>> tgt = torch.rand((20, 32, 512))
        >>> out = transformer_model(src, tgt)
    '''
    def __init__(self, d_model: int = 512, nhead: int = 8, num_encoder_layers: int = 6,
                 num_decoder_layers: int = 6, dim_feedforward: int = 2048, dropout: float = 0.1,
                 activation = F.relu, custom_encoder: Optional[Any] = None, custom_decoder: Optional[Any] = None,
                 layer_norm_eps: float = 1e-5, batch_first: bool = False) -> None:
        super(Transformer, self).__init__()
        if custom_encoder is not None:
            self.encoder = custom_encoder
        else:
            encoder_layer = TransformerEncoderLayer(d_model, nhead, dim_feedforward, dropout,
                                                    activation, layer_norm_eps, batch_first)
            encoder_norm = nn.LayerNorm(d_model, eps=layer_norm_eps)
            self.encoder = TransformerEncoder(encoder_layer, num_encoder_layers)

        if custom_decoder is not None:
            self.decoder = custom_decoder
        else:
            decoder_layer = TransformerDecoderLayer(d_model, nhead, dim_feedforward, dropout,
                                                    activation, layer_norm_eps, batch_first)
            decoder_norm = nn.LayerNorm(d_model, eps=layer_norm_eps)
            self.decoder = TransformerDecoder(decoder_layer, num_decoder_layers, decoder_norm)

        self._reset_parameters()

        self.d_model = d_model
        self.nhead = nhead

        self.batch_first = batch_first

    def forward(self, src: Tensor, tgt: Tensor, src_mask: Optional[Tensor] = None, tgt_mask: Optional[Tensor] = None,
                memory_mask: Optional[Tensor] = None, src_key_padding_mask: Optional[Tensor] = None,
                tgt_key_padding_mask: Optional[Tensor] = None, memory_key_padding_mask: Optional[Tensor] = None) -> Tensor:
        r'''
        参数：
            src: 源语言序列（送入Encoder）（必备）
            tgt: 目标语言序列（送入Decoder）（必备）
            src_mask: （可选)
            tgt_mask: （可选）
            memory_mask: （可选）
            src_key_padding_mask: （可选）
            tgt_key_padding_mask: （可选）
            memory_key_padding_mask: （可选）
        
        形状：
            - src: shape:`(S, N, E)`, `(N, S, E)` if batch_first.
            - tgt: shape:`(T, N, E)`, `(N, T, E)` if batch_first.
            - src_mask: shape:`(S, S)`.
            - tgt_mask: shape:`(T, T)`.
            - memory_mask: shape:`(T, S)`.
            - src_key_padding_mask: shape:`(N, S)`.
            - tgt_key_padding_mask: shape:`(N, T)`.
            - memory_key_padding_mask: shape:`(N, S)`.
            [src/tgt/memory]_mask确保有些位置不被看到，如做decode的时候，只能看该位置及其以前的，而不能看后面的。
            若为ByteTensor，非0的位置会被忽略不做注意力；若为BoolTensor，True对应的位置会被忽略；
            若为数值，则会直接加到attn_weights
            [src/tgt/memory]_key_padding_mask 使得key里面的某些元素不参与attention计算，三种情况同上
            - output: shape:`(T, N, E)`, `(N, T, E)` if batch_first.
        注意：
            src和tgt的最后一维需要等于d_model，batch的那一维需要相等
            
        例子:
            >>> output = transformer_model(src, tgt, src_mask=src_mask, tgt_mask=tgt_mask)
        '''
        memory = self.encoder(src, mask=src_mask, src_key_padding_mask=src_key_padding_mask)
        output = self.decoder(tgt, memory, tgt_mask=tgt_mask, memory_mask=memory_mask,
                              tgt_key_padding_mask=tgt_key_padding_mask,
                              memory_key_padding_mask=memory_key_padding_mask)
        return output
        
    def generate_square_subsequent_mask(self, sz: int) -> Tensor:
        r'''产生关于序列的mask，被遮住的区域赋值`-inf`，未被遮住的区域赋值为`0`'''
        mask = (torch.triu(torch.ones(sz, sz)) == 1).transpose(0, 1)
        mask = mask.float().masked_fill(mask == 0, float('-inf')).masked_fill(mask == 1, float(0.0))
        return mask

    def _reset_parameters(self):
        r'''用正态分布初始化参数'''
        for p in self.parameters():
            if p.dim() > 1:
                xavier_uniform_(p)


transformer_model = Transformer(nhead=16, num_encoder_layers=12)
src = torch.rand((10, 32, 512))
tgt = torch.rand((20, 32, 512))
out = transformer_model(src, tgt)
print(out.shape)
# torch.Size([20, 32, 512])
```



