# ゼロから作るDeep Learning 5

## 第4章 混合ガウスモデル

### 混合ガウス分布のサンプリング

1. どの正規分布を使うか選ぶ: カテゴリカル分布でのサンプリング
2. 選んだ正規分布でサンプリング



### 混合ガウスモデル

1. どの正規分布を使うか選ぶ

   カテゴリカル分布で実装
   $$
   p(z = k; \mathbb{\phi}) = \phi_k
   $$

2. 選んだ$z$番目の正規分布からサンプリング

$$
p(\mathbb{x}|z=k; \mathbb{\mu}, \mathbb{\Sigma}) = N(\mathbb{x}; \mathbb{\mu}_k, \mathbb{\Sigma}_k)
$$

2つの式から、
$$
p(\mathbb{x}) = \sum_{k=1}^K p(\mathbb{x}, z=k) \\
= \sum_{k=1}^K p(z=k) p(\mathbb{x}|z=k) \\
= \mathbb{\phi_k} N (\mathbb{x}; \mathbb{\mu_k}, \mathbb{\Sigma_k})
$$
つまり、パラメータ$\mathbb{\phi_k}, \mathbb{\mu_k}, \mathbb{\Sigma_k}$ があれば、推論できる！

### パラメータの推定

解析的に解けない。

上式$p(\mathbb{x})$ の最尤推定を行う。
$$
\log p(D; \mathbb{\theta}) = \sum_{n=1}^N \log (\sum_{k=1}^{K} \phi_k N (\mathbb{x}^{n};\mathbb{\mu_k}, \mathbb{\Sigma_k}))
$$
これをパラメータ$\mathbb{\phi_k}, \mathbb{\mu_k}, \mathbb{\Sigma_k}$ で微分してゼロになればいいんだけど...。解析的に無理です！

=> EMアルゴリズムを使えば行ける！



## EMアルゴリズム











