---
title: "Oblivious Transfer and Cyclic Group"
date: 2022-07-19T19:03:35+08:00
draft: false
math: true
---
## Cyclic Group
### 定义
$$\langle\mathbb{G},q,g \rangle$$

\\(\mathbb{G}=\langle \mathbb{Z}_{n}, \cdot  \rangle\\) 

\\(\mathbb{Z}_{n} \\)是一个集合, {$0$...$n-1$}.

\\(\cdot\\)是集合中的运算符.

对\\(\mathbb{G}\\)中的元素$a$进行$k$次幂运算表示为\\(a^{k}=a \cdot a ...\cdot a\\), 即$k$个$a$进行$\cdot$运算.

若群$\mathbb{G}$的每一个元素都是$\mathbb{G}$的某一个固定元素$a$的幂，则称$\mathbb{G}$为循环群.

$q$是$\mathbb{G}$的order, $q$的值等于$\mathbb{G}$中元素的个数, 也记为$|\mathbb{G}|$.

$g$是$\mathbb{G}$的generator(生成元), $\mathbb{G}$中的所有元素都能由$g$通过幂运算生成. 

由群$\mathbb{G}$, 阶$q$和生成元$g$, 即可定义一个循环群.

### 如何寻找一个生成元
如果$\mathbb{G}$有素数阶$p$, 则$\mathbb{G}$中除了identity之外的所有元素都是$\mathbb{G}$的生成元.

如果$p$是素数, 则$\mathbb{Z}_{p}^{*}$是阶为$p-1$的循环群.

假设$\mathbb{G}$阶非素数$p$, 可以均匀的从$\mathbb{G}$中采样元素, 直到这个元素是一个生成元.
- $\mathbb{G}$的阶$q$有素数因数$\\{p_{i}\\}^k_{i=1}$, 检查元素$h$是否为生成元
  - for $i = 1$ to $k$ : if $h^{q/p_{i}}=1$ return "$h$ is not a generator"
  - return "$h$ is a generator"

- $\mathbb{G}$有阶$q$,生成元$g$, 则任意一个元素可以表示为$h=g^{x}$. 
  - 当gcd($x,q$)=1时, $h$也是$\mathbb{G}$的一个生成元.
## Oblivious Transfer from Cyclic Group
Reference:
> More efficient oblivious transfer and extensions for faster secure computation
> 
> Asharov G, Lindell Y, Schneider T, et al. 
> 
> Proceedings of the 2013 ACM SIGSAC conference on Computer & communications security.

以下方案基于DDH假设.
### Fast Notes
1. Input
   - Sender and Receiver agree on $\langle\mathbb{G},q,g \rangle$.
   - Sender and Receiver agree on Hash.
   - Sender: n个 ($x_{i}^{0}, x_{i}^{1}$) pair 
   - Receiver: n个 bit ($\sigma_{1},...,\sigma_{n}$) 
2. Receiver do
   - $\alpha_{i}\in_{R}\mathbb{Z}_{q}$
   - $h_{i}\in_{R}\mathbb{G}$
   - $h_{i}^{0}= (\sigma_{i} == 0) \ ?\ g^{\alpha_{i}} : h_{i}$
   - $h_{i}^{1}= (\sigma_{i} == 0) \ ?\ h_{i} : g^{\alpha_{i}}$
   - send $n$ pair $(h_{i}^{0}, h_{i}^{1})$ to Sender.
3. Sender do 
   - $r\in_{R}\mathbb{Z}_{q}$
   - $u=g^{r}$
   - $k_{i}^{0}, k_{i}^{1} = (h_{i}^{0})^{r}, (h_{i}^{1})^{r}$
   - $v_{i}^{0}, v_{i}^{1} = x_{i}^{0} \oplus \text{Hash}(k_{i}^{0}), x_{i}^{1} \oplus \text{Hash}(k_{i}^{1})$
   - send $u$ to Receiver.
   - send n pair $(v_{i}^{0}, v_{i}^{1})$.
4. Receiver do
   - $k_{i}^{\sigma_{i}} = u^{\alpha_{i}}$
   - $x_{i}^{\sigma_{i}} = v_{i}^{\sigma_{i}} \oplus \text{Hash}(k_{i}^{\sigma_{i}})$
5. Receiver outputs $\\{x_{i}^{\sigma_i}\\}$, Sender has no output.

### Discussion
1. $x$应当和Hash的长度保持一致, 或者小于Hash的长度, 否则$v$的头几位会泄露$x$的信息. 如果Hash的长度与$k$的长度一致, 由于$k$是$\mathbb{G}$中的元素, $x$也应是$\mathbb{G}$中的元素, 否则会泄露$x$的信息. 最好的情况是, S和R约定好Hash, 可以从较短的$k$中产生较长的Hash$(k)$, 那么此时也应约定好$x$的bit长度$l$.
2. OT传输的信息长度可能会变, 也可能太长, 如果导致计算的开销过高, 或许可以通过与对称密码结合的方法, 用OT传输秘钥, 用秘钥加密$x$, 从而让Receiver获得秘钥来解密密文. 此时对称加密的秘钥长度就可以固定下来. 不过对称加密如AES本身秘钥的长度就达到了129,192和256, 或许已经超过了$\mathbb{G}$的长度.
3. MPC中最用的两中有限域是$prime$域和$2k$域$\mathbb{G}\_{p}$和$\mathbb{G}_{2^k}$, 理论上来说素数域的效率会更高一些, 而2k域正好对应计算时32bit, 64bi的数据范围.
