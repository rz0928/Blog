---
title: 什么是HTTPS?
date: 2024-04-22 21:22:13
categories: 计算机基础
tags: 计算机网络
---

# 1. HTTPS概述

HTTPS相较于HTTP多了一个安全层，也就是`HTTPS ≈ HTTP + SSL/TLS`。

SSL（Secure Sockets Layer）安全套接层，TSL（Transport Secure Layer）传输层安全协议，可以看做SSL的高版本。

## 1.1 HTTPS要干的事

HTTPS要实现的功能主要有三个（先整体看一下，下面具体说）

1. **机密性**：防止被别人截取到重要信息。使用混合加密

2. **报文完整性**：防止通信信息被恶意篡改。使用hash算法，信息是`m`传输`m+H(m)`
3. **端点鉴别**：校验客户端和服务端的身份（防止中间人攻击）。使用CA证书

# 2. 机密性

加密主要有两大类，分别是**对称加密**和**非对称加密**

**场景**：Alice和Bob要进行通信，以及旁观的Trudy

## 2.1 对称加密

**Alice**向**Bob**说`bob,i love you.alice`，由于说的是明文，在一旁的**Trudy**也可以听到。

**Alice**和**Bob**也发现了这一点，于是他们就想出了一个简单的办法，将要传达的信息通过一下**规则R**转变后再发送

**规则R**：

```
明文字母: a b c d e f g h i j k l m n o p q r s t u v w x y z
密文字母: m n b v c x z a s d f g h j k l p o i u y t r e w q
```

这样`bob,i love you.alice`就变成了`nkn, s gktc wky.mgsbc`。**Bob**在接收到信息后再根据**规则R**解密，获得真正的信息。

这样**Trudy**就会听的一脸懵，你们在说啥？不过聪明的`Trudy`每天听他们交流，统计了所有词的出现频率，根据他们平时交流用的词汇数量（比如**bob**和**nkn**出现次数差不多，就推测`b=n,o=k`）推出了他们交流使用的**规则R**，这样他们交流的内容就被**Trudy**破解了(不过这种暴力破解方式遇到比较复杂的加密方式就很难破解了)。

另外，**Trudy**还有可能偷听到**Alice**与**Bob**协商的规则R，这样Trudy甚至不需要时间推算就可以获取加密信息。

## 2.2 非对称加密

不久后，Alice和Bob也知道了这一方法的不妥，他们殚精竭虑又想出了一个新的办法，非对称加密（以RSA算法为例）。

(RSA算法是基于数学模运算实现的，本质是两种运算算力的不对等，感兴趣的自行查阅)

Bob持有RSA算法的私钥，然后Bob将公钥告诉Alice。之后Alice就可以用公钥将信息加密然后发送给Bob，Bob将得到的信息用自己的私钥解密就可以得出真正的信息。

**问题**：

1. 这种Trudy可能不知道Alice向Bob说了什么但是还是可以篡改Alice向Bob发送的信息，这样Bob也不知道Alice在说啥了。

2. Trudy也可以知道Bob告诉Alice的公钥，然后假装自己是Alice向Bob发送`bob,i hate you.alice`，Bob就有可能误会Alice。

# 3. 报文完整性

为了防止Trudy篡改消息，Alice将要发出的消息，用进行以下处理：

原消息为**message**，处理后的消息为**message + hashCode(message)**，将`hashCode(message)`作为校验码，由于两个大片内容的散列函数值几乎不可能相同，所以Bob只需要将**message的hashcode值**与**校验码**进行比较就可以确定信息是否被篡改过。

但是问题又来了？万一校验码也一起被替换了怎么办？

于是Alice和Bob又商量了一个新的加密方式。假设要发送的消息为`m`，两人协商一个**鉴别密钥s**，将校验码变为`h(m+s)`，信息还是`m`，这样Bob收到后，就可以将**信息`m`加上密钥`s`拼接**在一起计算**hashCode**与**校验码**比较。

问题又又又来了，Alice和Bob怎么保证**鉴别密钥s**只有他们两个知道呢？

可以明显发现，Alice与Bob沟通依赖出现了死循环，内部消化不了，只能通过外部助力，这样CA就应运而生

# 4. 端点鉴别

为了防止Trudy假装成Alice，Bob需要鉴别谁是真正的Alice。

HTTPS中使用将公钥和信息发送方（Alice）绑定的方式来保证数据一定是由Alice发送的，具体是通过认证中心（Certification Authority，CA）来完成的。

**CA认证过程**：Alice和Bob通信时，Alice需要获得Bob的公钥。现在不直接将Bob的公钥发给Alice，而是先通过一个第三方的机构（CA）来将Bob公钥加上一些信息生成为一个证书，















