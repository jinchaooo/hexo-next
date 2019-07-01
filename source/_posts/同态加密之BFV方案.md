---
title: 同态加密之BFV方案
date: 2019-02-08 09:47:56
tags: FHE
categories: 算法和研究
---
学习同态加密也有一段时间了。对于工程背景非数学专业的人来说，这个领域还是有点入门门槛的。前几个月一直徘徊在概念理解阶段，后来对这一课题有了一个整体的认识，也在将同态加密的packing和神经网络结合应用方面有了些自己的想法和实现。可以说基本上入门了，但前面还有非常多的东西要学。

今天来记录一下最近学习微软的同态加密库SEAL中BFV方案的一些笔记。BFV与BGV，CKKS等是同态加密基于RLWE困难问题的一类方案中最出名的几种方案。这里先从如何使用BFV库的角度来理解BFV方案相关的概念。后面会再从算法和原理的角度深入分析。

## BFV方案的参数

要使用BFV，首先要设置相关的参数。目前BFV的EncryptionParameters包含以下参数。

<style>
table th:first-of-type {
    width: 30%;
}
table tbody tr:nth-child(odd) {
    background: #F5F5F5;
}
table tbody tr:nth-child(even) {
    background: none;
}
table tbody tr:hover:nth-child(odd) {
    background: #F5F5F5;
}
table tbody tr:hover:nth-child(even) {
    background: none;
}
</style>

|参数|描述|
|------|------|
|poly_modulus_degree|环的分母项（分圆多项式）x^n + 1中n的值。明文多项式或密文多项式中最高次数为n - 1。BFV方案中n必须为2的幂，通常取值为2^10到2^15。n取值越大则RLWE加密方案越安全，但对应的密文也越大，各种计算等操作也更慢。|
|coeff_modulus|**1)**. 密文多项式系数的模。通常记为q或logq。<br>**2)**. q是决定密文噪音容忍度noise budget的重要参数。q值越大则noise budget越大，密文可支持更大的运算深度。但q值越大则RLWE加密方案越不安全（需要增大n值来“中和”）。<br>**3)**. BFV的q是多个较小的素数的积，素数可以为30bit到60bit。SEAL实现的库中预定义了一些素数表，可以用small_mods_xxbits(int index)来获取。<br>**4)**. 相应的，SEAL也提供了coeff_modulus_128bit(int n)等函数来获取在128bit等安全级别下某n值所对应的一组素数模数。<br>**5)**. SEAL中使用NTT来实现多项式在素数模下的乘积，因此需要每个素数都满足在模2n下与1同余。<br>**6)**. 另外，素数模的个数也是影响加密计算等操作的重要因素，因此，在允许的范围内尽量使用较大的素数模以减少总的素数模的个数。|
|plain_modulus|**1)**. 明文多项式系数的模。明文多项式系数的模可以取值任意的正整数。在新加密生成的密文多项式中的noise budget的值为log2（coeff_modulus/plain_modulus）bits，而每次同态乘法操作所消耗的noise budget为log2（plain_modulus）加上一些其它的项。因此在允许的范围内，明文多项式系数的模取值越小越好。但是也有以下几点需要注意：<br>**2)**. 如果要使用packing效果的BatchEncoder，需要plain_modulus的取值为在模2n下与1同余的素数。这时，明文多项式可以看作内含一个2-by-(n/2)的矩阵，每个矩阵元素是一个在模plain_modulus下的整数。<br>**3)**.随着计算深度的增加，明文多项式内pack的值可能会越来越大，最终超出plain_modulus时会发生wrap around，导致解密后得不到实际应用想要的结果。因此需要设置plain_modulus的值大于预期的最终结果以防止wrap around的发生。<br>**4)**. 如果实际应用所需要的plain_modulus过大（如最终结果的值过大），可以采用中国剩余定理CRT的方法：选取一组小的素数（满足模2n下与1同余以利用packing）作为plain_modulus，将实际应用的输入对这组素数CRT分解后分别在每个素数plain_modulus下运算，得到的一组结果再通过逆CRT合并后得到需要的原始结果。实际上，通过逆CRT合并得出的结果是在以所有素数plain_modulus乘积为模下的结果，因为这个乘积大于预计的原始结果，所以不会有wrap around发生。|
|noise_standard_deviation|默认值是3.20，一般不需要修改。|
|random_generator|自定义随机数生成器。|

<!--more-->

## BFV方案将数值编码进明文多项式的方法（encode）

BFV的加密过程是将明文多项式转化为密文多项式，而不是直接对整数或浮点数进行加密操作。因此，编码操作就是将整数或浮点数转化为明文多项式，以便进行随后的加密操作。BFV目前主要有三种编码方式：IntegerEncoder, FractionalEncoder, 和BatchEncoder。

|编码器|描述|
|-|-|
|IntegerEncoder|将一个整数编码成一个明文多项式。先将整数表示成b进制的形式，注意此时b进制的每一位系数可能为正或负，其取值范围为-b/2 ... b/2 - 1（b为偶数）或-(b-1)/2 ... (b-1)/2 （b为奇数）。当b为2时稍为特殊，若整数为正则系数取值为0或1，若整数为负则系数取值为0或-1。接下来将b换为x得到明文多项式。<br>**注意1）**明文多项式实际上是在一个环中，环的模多项式为x^n + 1。因此要确保计算过程中明文多项式的最高次数小于n，否则会发生wrap around而得不到期望结果。另外，明文多项式的系数是在模plain_modulus下的，因此也要确保运算过程中系数比plain_modulus小而不发生wrap around。<br>**注意2）**负系数在转换为多项式时以模plain_modulus后的正数表示和运算。<br>**注意3）**解码只需将b带入x计算即可。|
|FractionalEncoder|将一个浮点数编码成一个明文多项式。浮点数的整数部分跟整数编码方法一样。小数部分对应项的次数为负数，此时给每一项乘以-x^n（类似于1与-x^n在模x^n + 1下同余）转换为正的高次项。浮点数的加乘运算对应明文多项式的加乘运算，除了要注意系数不发生wrap around外，还要注意低次项和高次项在运算过程中向中间靠拢时不发生交叉覆盖。解码时，低次的整数部分直接将b代入x，高次的小数部分转回负次数后代入。|
|BatchEncoder|将一组模plain_modulus下的整数编码成一个明文多项式。这是BFV的数据packing方式：明文多项式可以看作2-by-(n/2)的整数矩阵。（此时需要plain_modulus是在模2n下与1同余的素数。）BatchEncoder背后的数学原理是**基于多项式环的**中国剩余定理（R-CRT）。<br>事实上，**在模plain_modulus的意义下**，环的模多项式(x^n + 1)可以分解为(x - a1)(x - a2)...(x - an)的形式，其中a1 ... an是不同的2n次本原单位根。根据R-CRT，模(x^n + 1)的多项式与n个分别模(x - a1)到模(x - an)的一组多项式（此处实际是n个整数，因为模是1次多项式）同构。<br>使用BatchEncoder时还有一个很重要的功能是可以旋转移动内含矩阵每行中的元素（横向旋转），也可以将矩阵两行互换（纵向旋转）。旋转基于的是多项式环的自同构原理，具体实现上是将x^k代替x代入原多项式中。对于不同的旋转步数，需要在(Z mod 2n)*中找出相应的k值。|

## BFV方案的再线性化操作（relinearization）

BFV密文的size是其所含的多项式的个数。初始时密文由2个多项式组成，其size为2。但是，每次乘法操作后，结果密文的size变为M + N - 1，其中M和N分别为参与乘法操作的密文的size。虽然说size变大后密文还能正常解密，但是会带来两个负面影响。**第一，乘法和加法的计算开销变大。**每次密文乘法需要进行O(M \* N)次多项式乘法，而密文加法则需要O(M + N)次多项式加法。**第二，每次乘法操作所消耗的noise budget也变大。**再线性化Relinearization可以让密文的size重新回到最小值2，因此对性能有较大的好处。Relinearization操作需要先创建相应的Relinearization Keys，每一次从size为M降为2的操作需要M - 2个Keys。如果需要relinearize的密文的size与relinearization操作的keys的数目不匹配，则会发生错误。<br>然而，Relinearization操作本身也有计算开销和noise budget消耗，并且与decomposition bit count参数（记为w）相关。w取值为1到60之间的整数。总体来说，若w取值小，计算速度较慢，但每次操作noise budget消耗很少；反之，若w取值大，计算速度较快，但每次操作noise budget消耗很大。但是，虽然某个w可能取值较大，但该w在每一组加密参数取值下都对应一个**noise budget阈值**，当ciphertext的noise budget在该阈值之上时，每次relinearization操作会消耗较多的noise budget；一旦ciphertext的noise budget减到低过该阈值，再进行relinearizaiton的noise budget消耗会下降到很小。

## BFV方案BatchEncoding下的旋转操作（Rotation）

如前所述，在BatchEncoding下，每个密文可看作内含一个2-by-(n/2)的矩阵。而旋转操作Rotation就是将矩阵内的元素按行或列循环移位。与relinearization操作一样，Rotation也是基于Decomposition Bit Count参数w，w在1到60之间取值越大则计算越快，但消耗的noise budget也越多。同样的，在某个w和一组加密参数下存在对应的noise budget阈值，当密文的noise budget大于阈值时，大的w会导致noise budget下降快，但一旦下降到低于该阈值，则再进行Rotation操作noise budget消耗会下降到很小。<br>进行旋转操作需要先生成所需的Galois Keys。一般来说，每一个2的幂次的旋转步长需要一个对应的Galois Key，非2的幂次的旋转步长表示成多个2的幂次相加的形式。因此要生成适合slots个数为N下的所有旋转步长的Galois Keys，只需生成logN个keys。<br>另外，参数w会影响生成的Galois Keys的大小。每一个decompostion因子对应GaloisKey中的一个数据项，w越小对应的数据项越多，Key的size越大。

## BFV方案的模转换操作（modulus switching）

BFV密文多项式系数的模coeff_modulus是一组素数的乘积。在初始化一组加密参数时，SEAL会自动生成一个参数链，链上的每个节点都是由初始参数推算出的一组新的加密参数，并且，每个节点与前一个节点的参数相比只是在coeff_modulus上减少了一个素数，其它的完全一样。链上最后一组参数的index为0，且满足参数的有效性：coeff_modulus大于plain_modulus的值。<br>初始创建的参数组中coeff_modulus是最大的，每次modulus switching都会减少coeff_modulus，从而可能导致noise budget降低。然而，modulus switching也有一些潜在的好处。

* 密文大小与coeff_modulus线性正相关，如果确定密文不需再参与计算后，可以将其coeff_modulus降到最低值，以降低存储和传输的开销。需要注意的是，只要密钥是创建给参数链上高的coeff_modulus的，那它可以用来解密链上所有coeff_modulus比它低的参数所对应的密文。
* 密文在经过计算后其内部的noise budget已经降低，此时如果**适当降低**coeff_modulus，有可能不会对noise budget产生影响。
* 有些情况下即使减小coeff_modulus会降低noise budget，但可以有效降低随后的计算开销（加快计算速度），并减小密文的大小，因此仍然是一种有益的trade off。

BFV方案中的modulus switching不是必需的，因此在创建参数时可以选择只生成单一参数组而不生成参数链。与BFV不同，CKKS方案中的modulus switching有着更重要的作用。