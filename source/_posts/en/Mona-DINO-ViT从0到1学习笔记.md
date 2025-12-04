---
title: Mona-DINO-ViT从0到1学习笔记
catalog: true
date: 2025-10-19 10:46:51
subtitle: 模型微调入门篇
sticky: 23
lang: en
header-img: /img/header_img/VSCappearance.png
tags:
- [MedicalImage]
categories:
- [MedicalImage]
---

## Foreword

​	北京已经快入冬了。笔者今早差点冻死在自行车上。

​	最近事情很多很多，但是想了一想还是觉得需要写一篇博客记录这段不凡的旅程。如若没有一个稳定的记录，恐怕过几天也会被笨人遗忘，故下定决心以记之。

​	此外，笔者忙里偷闲安置了一个子域名网站做随笔日记本，但是cloudflare配置最近出了点问题，感兴趣的朋友可以先搜索`http://101.200.30.9:6277/`~~（放在这里会不会不太安全）~~，后续将以`echo.0mnilink.top`的形态和大家见面，笔者会在里面定期真实发疯，更抽象、更贴近生活😮

## Beginning of the story

​	我们的核心目标就是学习、了解adaptor微调模型的部分细节与项目框架，并且和调研需求紧密结合，最后给出一个比较reasonable的result，思路流程如图所示：
![pipeline](pipeline.png)

​	这篇博客将作为记录型博客持续更新，伴随笔者完成每一步的任务！事不宜迟，准备出发！

## Understand & Prepare

### 理解Mona的核心机制

​	我们根据经验，为自己准备了三个主线任务：

- **输入/输出：** 搞清楚Mona模块的forward函数接收哪些输入，输出什么。记下它们的维度信息（如 [Batch, Num_Heads, Seq_Len, Head_Dim]）。
- **作用位置：** 明白Mona是在Transformer的哪个具体计算步骤中起作用的。是在Q, K, V投影之后，Attention分数计算之前吗？还是作用于FFN层？。
- **核心参数：** 知道rank (秩) 是控制Mona参数量的关键超参数。理解rank越大，引入的参数越多，拟合能力可能越强，但训练成本也越高。

#### Input/Output of Mona module

```python
class MonaOp(nn.Module):
    def __init__(self, in_features):
        super().__init__()
        self.conv1 = nn.Conv2d(in_features, in_features, kernel_size=3, padding=3 // 2, groups=in_features)
        self.conv2 = nn.Conv2d(in_features, in_features, kernel_size=5, padding=5 // 2, groups=in_features)
        self.conv3 = nn.Conv2d(in_features, in_features, kernel_size=7, padding=7 // 2, groups=in_features)
		# 三个并行的 深度可分离卷积
        self.projector = nn.Conv2d(in_features, in_features, kernel_size=1, )
		# 用于特征融合
    def forward(self, x):
        identity = x # 用于第一个残差连接
        conv1_x = self.conv1(x)
        conv2_x = self.conv2(x)
        conv3_x = self.conv3(x)

        x = (conv1_x + conv2_x + conv3_x) / 3.0 + identity

        identity = x # 用于第二个残差连接

        x = self.projector(x)

        return identity + x

class Mona(BaseModule):
    def __init__(self,
                 in_dim,
                 factor=4):
        super().__init__()

        self.project1 = nn.Linear(in_dim, 64) # 降维至64
        self.nonlinear = F.gelu
        self.project2 = nn.Linear(64, in_dim) # 从64升维

        self.dropout = nn.Dropout(p=0.1)

        self.adapter_conv = MonaOp(64)

        self.norm = nn.LayerNorm(in_dim)
        self.gamma = nn.Parameter(torch.ones(in_dim) * 1e-6)
        self.gammax = nn.Parameter(torch.ones(in_dim))

    def forward(self, x, hw_shapes=None):
        identity = x

        x = self.norm(x) * self.gamma + x * self.gammax

        project1 = self.project1(x)

        b, n, c = project1.shape
        h, w = hw_shapes
        project1 = project1.reshape(b, h, w, c).permute(0, 3, 1, 2)
        # 序列[B, L, 64] ==> 图像网格 [B, 64, H, W]，要用到参数hw_shapes
        project1 = self.adapter_conv(project1)
        project1 = project1.permute(0, 2, 3, 1).reshape(b, n, c)

        nonlinear = self.nonlinear(project1)
        nonlinear = self.dropout(nonlinear)
        project2 = self.project2(nonlinear)

        return identity + project2
```

Mona 模块的 forward 函数接收 **两个** input：

1. **x**: 这是主要的特征张量，是上一层（MSA或MLP）的输出。
   - **维度**: [B, L, C]
     - B: Batch Size
     - L: Sequence Length  *= H \* W*
     - C: Channel / Embedding Dimension (特征维度)
2. **hw_shapes**: 这是一个包含特征图高度和宽度的元组 (tuple)。
   - **作用**: 这个参数至关重要，因为 Mona 模块内部需要进行卷积操作。它必须知道如何将长度为 L 的序列 x 重新塑形 (reshape) 回 [H, W] 的二维空间网格。

Mona 模块的 forward 函数输出 **一个** output：

1. **identity + project2**:
   - **维度**: [B, L, C]
     - 输出的维度与输入的特征张量 x **完全相同**。这是为了确保它可以无缝地接入Transformer的残差连接结构中。

#### Action Position

```python
class SwinTransformerBlock(nn.Module):
    """ Swin Transformer Block.

    Args:
        dim (int): Number of input channels.
        num_heads (int): Number of attention heads.
        window_size (int): Window size.
        shift_size (int): Shift size for SW-MSA.
        mlp_ratio (float): Ratio of mlp hidden dim to embedding dim.
        qkv_bias (bool, optional): If True, add a learnable bias to query, key, value. Default: True
        qk_scale (float | None, optional): Override default qk scale of head_dim ** -0.5 if set.
        drop (float, optional): Dropout rate. Default: 0.0
        attn_drop (float, optional): Attention dropout rate. Default: 0.0
        drop_path (float, optional): Stochastic depth rate. Default: 0.0
        act_layer (nn.Module, optional): Activation layer. Default: nn.GELU
        norm_layer (nn.Module, optional): Normalization layer.  Default: nn.LayerNorm
    """

    def __init__(self, dim, num_heads, window_size=7, shift_size=0,
                 mlp_ratio=4., qkv_bias=True, qk_scale=None, drop=0., attn_drop=0., drop_path=0.,
                 act_layer=nn.GELU, norm_layer=nn.LayerNorm):
        super().__init__()
        self.dim = dim
        self.num_heads = num_heads
        self.window_size = window_size
        self.shift_size = shift_size
        self.mlp_ratio = mlp_ratio
        assert 0 <= self.shift_size < self.window_size, "shift_size must in 0-window_size"

        self.norm1 = norm_layer(dim)
        self.attn = WindowAttention(
            dim, window_size=to_2tuple(self.window_size), num_heads=num_heads,
            qkv_bias=qkv_bias, qk_scale=qk_scale, attn_drop=attn_drop, proj_drop=drop)

        self.drop_path = DropPath(drop_path) if drop_path > 0. else nn.Identity()
        self.norm2 = norm_layer(dim)
        mlp_hidden_dim = int(dim * mlp_ratio)
        self.mlp = Mlp(in_features=dim, hidden_features=mlp_hidden_dim, act_layer=act_layer, drop=drop)

        self.H = None
        self.W = None

        self.my_module_1 = Mona(dim, 8)
        self.my_module_2 = Mona(dim, 8) # Adapter_FFN(dim, 8)

    def forward(self, x, mask_matrix):
        """ Forward function.

        Args:
            x: Input feature, tensor size (B, H*W, C).
            H, W: Spatial resolution of the input feature.
            mask_matrix: Attention mask for cyclic shift.
        """
        B, L, C = x.shape
        H, W = self.H, self.W
        assert L == H * W, "input feature has wrong size"

        shortcut = x
        x = self.norm1(x)
        x = x.view(B, H, W, C)

        # pad feature maps to multiples of window size
        pad_l = pad_t = 0
        pad_r = (self.window_size - W % self.window_size) % self.window_size
        pad_b = (self.window_size - H % self.window_size) % self.window_size
        x = F.pad(x, (0, 0, pad_l, pad_r, pad_t, pad_b))
        _, Hp, Wp, _ = x.shape

        # cyclic shift
        if self.shift_size > 0:
            shifted_x = torch.roll(x, shifts=(-self.shift_size, -self.shift_size), dims=(1, 2))
            attn_mask = mask_matrix
        else:
            shifted_x = x
            attn_mask = None

        # partition windows
        x_windows = window_partition(shifted_x, self.window_size)  # nW*B, window_size, window_size, C
        x_windows = x_windows.view(-1, self.window_size * self.window_size, C)  # nW*B, window_size*window_size, C

        # W-MSA/SW-MSA
        attn_windows = self.attn(x_windows, mask=attn_mask)  # nW*B, window_size*window_size, C

        # merge windows
        attn_windows = attn_windows.view(-1, self.window_size, self.window_size, C)
        shifted_x = window_reverse(attn_windows, self.window_size, Hp, Wp)  # B H' W' C

        # reverse cyclic shift
        if self.shift_size > 0:
            x = torch.roll(shifted_x, shifts=(self.shift_size, self.shift_size), dims=(1, 2))
        else:
            x = shifted_x

        if pad_r > 0 or pad_b > 0:
            x = x[:, :H, :W, :].contiguous()

        x = x.view(B, H * W, C)

        # my_adapter
        # x = self.my_module_1(x)

        x = shortcut + self.drop_path(x)

        x = self.my_module_1(x,(H,W))

        identity = x
        x = self.norm2(x)

        # my_adapter2
        # x = self.my_module_2(x)

        # FFN
        x = self.mlp(x)

        # x = self.my_module_2(x)  # todo: correct

        x = identity + self.drop_path(x)

        x = self.my_module_2(x,(H,W))  # todo: correct

        return x
```

`SwinTransformerBlock`的`forward`函数中调用两次`Mona`模块：

1. `self.my_module1`:

   位于MSA(Multi-Self-Attention)之后，接收经过MSA和其残差连接的特征，并对之进行适配。

2. `self.my_module2`:

   位于MLP之后，接收经过MLP和其残差连接的特征，也同样进行适配。

因此`Mona`是一个独立的后处理模块，被放置在 Transformer Block 的两个核心计算单元（MSA 和 MLP）的输出位置，对它们的处理结果进行增强和适配。

#### Core Parameters

```python
self.project1 = nn.Linear(in_dim, 64) # Down-projection to n=64
self.project2 = nn.Linear(64, in_dim) # Up-projection from n=64
self.adapter_conv = MonaOp(64)       # Convolutions operate on n=64 channels
```

**核心超参数**: **中间维度 *n***

​	*n* 的大小直接控制了 Mona 模块的参数量和计算复杂度。*n* 越大，降维后的信息损失越少，拟合能力可能越强，但同时引入的参数也越多。

