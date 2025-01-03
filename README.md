# online-softmax
simplest online-softmax notebook for explain Flash Attention

Blog link: **[手撕Online-Softmax](https://zhuanlan.zhihu.com/p/5078640012)**

## Implemention

run `online_softmax_torch.ipynb`

we show the block online softmax result

```python
X = torch.tensor([-0.3, 0.2, 0.5, 0.7, 0.1, 0.8])
X_softmax = F.softmax(X, dim = 0)
print(X_softmax)

X_block = torch.split(X, split_size_or_sections = 3 , dim = 0) 

# we parallel compute  different block max & sum
X_block_0_max = X_block[0].max()
X_block_0_sum = torch.exp(X_block[0] - X_block_0_max).sum()

X_block_1_max = X_block[1].max()
X_block_1_sum = torch.exp(X_block[1] - X_block_1_max).sum()

# online block update max & sum
X_block_1_max_update = torch.max(X_block_0_max, X_block_1_max) # X[-1] is new data
X_block_1_sum_update = X_block_0_sum * torch.exp(X_block_0_max - X_block_1_max_update) \
                     + torch.exp(X_block[1] - X_block_1_max_update).sum() # block sum

X_block_online_softmax = torch.exp(X - X_block_1_max_update) / X_block_1_sum_update
print(X_block_online_softmax)
```

output is 

```
tensor([0.0827, 0.1364, 0.1841, 0.2249, 0.1234, 0.2485])
tensor([0.0827, 0.1364, 0.1841, 0.2249, 0.1234, 0.2485])
```

## Softmax Series

### softmax 

$$
\tilde{x}_i=\frac{e^{x_i}}{\sum_j^Ne^{x_j}}
$$

### safe softmax

$$
\tilde{x}_i=\frac{e^{x_i-\max(x_{:N})}}{\sum_j^Ne^{x_j-\max(x_{:N})}}
$$

Note $M=max(x_:N)$, so
$$
\begin{align}
\tilde{x}_i &=\frac{e^{x_i-\max(x_{:N})}}{\sum_j^Ne^{x_j-\max(x_{:N})}}\\
&=\frac{e^{x_i-M}}{\sum_j^Ne^{x_j-M}}\\
&=\frac{e^{x_i}/e^{M}}{\sum_j^Ne^{x_j}/e^{M}} \\
&=\frac{e^{x_i}}{\sum_j^Ne^{x_j}} \\
\end{align}
$$


### online softmax

1. We first compute  `1:N`  element maximum value $\max(x_{:N})$ and softmax denominator $l_N$

2. We add a new element  $x_{N+1}$, we update $\max(x_{:N+1})$ and update $l_{N+1}$ as follow. 

$$
\begin{align}
l_{N} &= \sum_j^N e^{x_j-\max(x_{:N})}\\
\max(x_{:N+1})&=\max( \max(x_{:N}), x_{N+1} )\\
l_{N+1} &= \sum_j^{N+1} e^{x_j-\max(x_{:N+1})} \\
&= (\sum_j^N e^{x_j-\max(x_{:N})}) +e^{x_{N+1}-\max(x_{:N+1})} \\
&=(\sum_j^N e^{x_j-\max(x_{:N})}e^{\max(x_{:N})-\max(x_{:N+1})})+e^{x_{N+1}-\max(x_{:N+1})} \\
&=(\sum_j^N e^{x_j-\max(x_{:N})})(e^{\max(x_{:N})-\max(x_{:N+1})}) +e^{x_{N+1}-\max(x_{:N+1})} \\
&=l_N (e^{\max(x_{:N})-\max(x_{:N+1})})+e^{x_{N+1}-\max(x_{:N+1})} \\
\end{align}
$$

​	we cannot use $l_{N+1}=l_{N}+x_{N+1}$, because safe softmax need all element subtract the same maximum value.

3. We can apply the softmax function using the adjusted numerator and denominator values.

$$
\tilde{x}_{i}=\frac{e^{x_i-\max(x_{:N+1})}}{l_{N+1}}
$$

### block online softmax

online softmax make  cumulative sum $l$ dynamic update while a new element added. It's more effiecent method is to update sum $l$ with block-wise element added. This advantage is we could parallelism to compute online softmax

1. we seperate compute different block $l^{(t)}$  and $m^{(t)}$

$$
\begin{align}
l^{(1)} &= l_{N} = \sum_j^N e^{x_j-\max(x_{:N})}\\
m^{(1)} &= \max(x_{:N}) \\
l^{(2)} &= l_{N:2N} = \sum_{j=N+1}^{2N} e^{x_j-\max(x_{{N+1}:2N})}\\
m^{(2)} &= \max(x_{N+1:2N}) \\
\end{align}
$$

2. it’s easy to update global $m,l$ 
$$
\begin{align}
m=\max({x_{:2N}})&=\max(\max({x_{:N}}),\max(x_{N+1:2N}))\\
&=max(m^{(1)},m^{(2)})
\end{align}
$$
but the $l$  NOT update  as follow：
$$
l=l_{:2N}\neq l^{(1)}+l^{(2)}
$$

3. So we based block sum $l^{(t)}$ and max $m^{(t)}$  to **online** update global $l$

$$
\begin{align}
l^{(1)}&= \sum_j^N e^{x_j-\max(x_{:N})} = \sum_j^N e^{x_j-m^{(1)}}\\
l^{(2)} &= \sum_{j=N+1}^{2N} e^{x_j-\max(x_{{N+1}:2N})} = \sum_{j=N+1}^{2N} e^{x_j-m^{(2)}}\\
l &= \sum_{j}^{2N} e^{x_j-\max(x_{:2N})} \\
&= (\sum_j^N e^{x_j-\max(x_{:2N})}) +(\sum_{j=N+1}^{2N}e^{x_j-\max(x_{:2N})}) \\
&= (\sum_j^N e^{x_j-m}) +(\sum_{j=N+1}^{2N}e^{x_j-m}) \\
&= (\sum_j^N e^{x_j-m^{(1)}}) (e^{m^{(1)}-m}) +(\sum_{j=N+1}^{2N}e^{x_j-m^{(2)}})(e^{m^{(2)}-m}) \\
&= l^{(1)} (e^{m^{(1)}-m}) +l^{(2)}(e^{m^{(2)}-m})
\end{align}
$$

4. update block softmax like:

$$
\tilde{x}_{i} =\frac{e^{x_i-m}}{l}
$$

### multi block online softmax

we do multi block online softmax by for-loop :
$$
l_\text{new}= l_\text{old} (e^{m_\text{old}-m}) +l_\text{new}(e^{m_{\text{new}}-m})
$$
noted current block max/sum as $m_\text{new},l_\text{new}$ ,the m is $m=\max(m_\text{old},m_\text{new})$, and then update:
$$
l_\text{old} \leftarrow l_\text{new}
$$

### batch online softmax

In attention machine, we need softmax for attention score matrix
$$
S=QK^T,S\in\mathbb{R}^{N\times N}
$$
the query is row-wise matrix $Q\in\mathbb{R}^{N\times D}$;

and we need softmax attention score:
$$
P_{i,:}=\text{softmax}(S_{i,:})
$$
when we use online-softmax, we could parallel update k-row max $M^{(t)}$ and row-wise sum $L^{(t)}$, 
$$
L = L^{(1)}(e^{M^{(1)}-M})+L^{(2)}(e^{M^{(2)}-M})
$$
where $L,M\in\mathbb{R}^{k\times 1}$

## Reference

[手撕Flash Attention](https://zhuanlan.zhihu.com/p/663932651)

[Online normalizer calculation for softmax](https://arxiv.org/abs/1805.02867)

