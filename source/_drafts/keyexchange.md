---
title: 密钥交换的安全原理
tags:
---

数据加密的原理几乎都可以总结为:

加密算法(密码, 明文) => 密文
解密算法(密码, 密文) => 明文
为了保证明文安全, 我们通常会将明文加密后再传输/存储, 而明文, 要么被毁灭, 要么被尽可能的保证是物理安全的

为了保证密文不被破译, 根据上述公式, 你必须保证(解密算法, 密码, 密文)至少一项不被他人所掌握

为了增强数据的安全性, 你甚至完全可以私创一套加解密算法, 然后用陆军护送密文, 空军护送密码, 海军护送解密算法, 送达同一个目的地,  结合三者即可取出明文; 只可惜这么做的代价太高了, 除了国家/富商, 没人可以时刻提供这种等级的安全保护(就算可以, 你还必须考虑时效性的问题, 以及这些军人的忠诚度/可靠度的问题)

作为一个普通人

我们没有足够的能力为每一次秘密的通信独创一套加解密算法(事实上这种做法大多数情况下是不值得的, 甚至是没有意义的), 只能使用公开的成熟的久经考验的加密算法
为了快速的通信或者使用别人的存储服务, 我们又不得不使用ISP为你架设的互联网, 不得不使用互联网公司提供的云存储服务, 不得不将密文暴露给第三方, 甚至是全世界
我们身不由己, 自己能够控制的就只剩下密码了
既然如此, 那么我们可以将安全问题简化成: 密码的安全的问题