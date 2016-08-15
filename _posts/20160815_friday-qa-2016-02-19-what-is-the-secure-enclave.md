title: "Friday Q&A 2016-02-19: 什么是安全区域？"
date: 2016-08-15
tags: [Swift]
categories: [Mike Ash]
permalink: friday-qa-2016-02-19-what-is-the-secure-enclave
keywords: secure enclave是什么,iphone安全区域,ios安全区域
custom_title: 
description: iPhone 手机使用的 iOS Secure Enclave 模块是什么呢，为什么连 FBI 甚至是苹果自己都无法破解呢，本文就来好好解析下 Secure Enclave 安全区域。

---
> 作者：Mike Ash，[原文链接](https://www.mikeash.com/pyblog/friday-qa-2016-02-19-what-is-the-secure-enclave.html)，原文日期：2016-02-19
> 译者：[littledogboy](undefined)；校对：[陈刚](undefined)；定稿：[CMB](https://github.com/chenmingbiao)
  







<!--此处开始正文-->

本周最大的科技新闻是 FBI 正试图迫使苹果公司解锁一个嫌疑人的 iPhone。有趣的是，涉案手机的型号是老款的 iPhone 5c。新款的 iPhone 中加入了苹果的 Secure Enclave（安全区域）技术，用来防止手机被暴力破解，甚至连苹果自己都无法破解。这件事过后许多朋友都在问一个问题：什么是 Secure Enclave？它扮演了什么角色？

在我开始写之前声明一下: 我平时写文章的习惯是一直深入到比特和字节，然后再讨论发生了什么。但这次必定有所不同，因为我这种凡夫俗子是触及不到安全区域的本质的。因此本文中大部分的知识来自[苹果 iOS 信息安全指南](https://www.apple.com/business/docs/iOS_Security_Guide.pdf)，又添加了一些通用的理论。参考指南中安全区域的相关信息，解释并且思考它们的含义。这篇文章建立在苹果提供了准确信息的假设上，因为从外部没有可行的方法来检测这些信息的真实性。因此最终检测结果将基于苹果文档的准确性，和我自己的理解，这点读者需要注意。

而且，此文章是为了调研本案的技术和安全区域技术。对 FBI 的要求，苹果的回应，任何其他政治问题，没有任何意见和暗示。如果你想在此讨论政治问题，出门右转。

找准了方向，让我们开始吧。

<!--more-->

## 回顾

涉案 iPhone 通过密码保护，密码没有直接存储在手机中。获取密码的唯一方法是暴力破解密码。计算验证密码非常慢，每次大约需要 80 毫秒。不过，暴力破解是可行的。一个四位数的密码，尝试 10000 个组合，每次 80 毫秒，只需不到 15 分钟。一个 6 位数字密码需要大约一天的时间。

这意味着密码并不十分安全。尝试多次输入密码失败后，苹果会增加额外的时间来延缓暴力破解的频次。几次错误的尝试之后，当你再次尝试输入密码时，iPhone 会让你等待，最开始等待 1 分钟，然后等待 5 分钟，后面的等待时间会递增。

你可能会想，如果把闪存从苹果电脑中取出来，复制它的内容，然后在一个更快的计算机上尝试破解密码，可以解决这个问题。这样就不会有苹果软件强加的额外延迟。 这么做带来的好处是，在更快的计算机上每次尝试的时间会少于 80 毫秒，并且可以并发执行多个尝试。然而，这是行不通的。因为数据加密与硬件是绑定的，所以必须在原始设备进行暴力破解。

没有安全区域的旧 iPhone 系统上有一个弱点。为了防止暴力破解而持续增加的延迟时间，只是手机操作系统的特征。密钥推导的计算时间是固定的80毫秒，而在多次错误的尝试之后额外添加的分钟或者小时数所起到的作用，仅仅是操作系统拒绝在这些延迟时间结束前接受输入而已。FBI 希望苹果构建和安装一个不包含延迟时间并且允许自动提交密码的系统版本。这将使得联邦调查局可以在几分钟或几小时内破解密码。除了苹果之外不允许任何人更新 iPhone 操作系统，所以，对于来自外部的攻击该系统是安全可靠的，但对苹果自身来说该系统是不安全的。

注意，以上的方案只针对使用数字密码的用户，比较复杂的密码在这些旧版 iPhone 仍然是安全的。例如，一个八个字符的字母数字密码将需要 550000 年的时间去尝试所有可能的组合。

## 不可读的 UIDs

每个内置的 iOS CPU 都独有一个 256 位的标识符或者 UID。UID 被直接嵌入到了硬件中，不存储在任何地方。UID 不能被外部直接访问，包括那些以最高权限在 CPU 上运行的软件。取而代之的是，CPU 中包含一个 AES 硬件加密引擎，而且唯一通过硬件获取到 UID 的方式是通过 AES 加密引擎加载秘钥，然后使用秘钥对数据进行加密或者解密。

苹果使这种硬件加密方式把用户密码和设备绑定起来。通过把设备的 UID 作为 AES 密钥，然后对密码再次加密，得到一系列随机的不可逆的数据。因为特定设备 UID 取决于用户密码和加密数据，因此它是不可读取的。加密过程执行了多次 [PBKDF2加密方法](https://en.wikipedia.org/wiki/PBKDF2)，把每轮加密后得到的结果作为下一轮加密的输入，以至于每次验证密码都需要 80 毫秒以完成复杂的运算。

## 安全区域

苹果首次引入安全区域是在搭载 A7 芯片的操作系统上。对应的手机型号从 iPhone 5S 开始，5 和 5C 以及旧版本都没有安全区域。iPad 端是从 iPad mini 2 和 iPad Air 开始。

安全区域 是 A7（以及之后的芯片）内部中一个独立的处理器，其负责低级别的加密操作。安全区域不能运行 iOS 程序或任何类似于 iOS 的程序，其上运行的是 [L4微内核](https://en.wikipedia.org/wiki/L4_microkernel_family) 程序。因为在高级权限下，运行时的代码可能会引发 bug，L4 意欲在内核中执行尽可能少的代码，从而使系统达到理论上的更加安全的状态。安全区域使用 secure boot (安全引导)系统，确保运行的代码不能被修改，它使用加密存储确保其它系统数据无法被读取或者篡改。这种方式有效地在计算机内另外构建了一台很难被攻击的电脑。

安全区域内包含自己的 UID 和 AES 引擎。该区域独立于系统的其余部分，密码验证的过程在安全区域中进行。安全区域同时也管理 Touch ID 指纹的处理和匹配，以及 Apple Pay 的授权。

安全区域管理着所有加密文件的密钥。文件加密几乎应用于所有的用户数据。大部分系统应用程序使都用了文件加密，并且系统版本不低于 iOS 7 的所有第三方应用程序默认使用的也是文件加密。每个加密文件都有一个唯一的秘钥，该秘钥又被设备的 UID 和 用户的密码进行了加密。主 CPU 自身并不能读取加密的文件，它必须从安全区域中获取文件的密钥，而该秘钥没有用户密码是获取不到的。

尝试输入密码失败而增加的延迟，是由安全区域执行的。主 CPU 只是提交密码和接收结果。安全区域负责检查密码，如果检查出多次失败，则将会延迟密码检查。主处理器并不能加快检查的速度。

## 启示

安全区域对系统整体安全来说担当一个什么角色？

在大多数系统中，如果你可以进入系统内核，那么你就可以拥有整个系统权限。内核可以完成任何事情。它可以读写系统内存中的每一个字节、控制所有的硬件、控制系统运行的所有应用程序的代码，并且可以破坏这一切。

由于安全区域是一个独立的 CPU，与系统的其余部分阻断，因此它不受内核的控制。在一个旧的 iPhone 上，掌控内核就意味着掌控了系统能做的一切功能，包括密码验证过程。有了安全区域，不管是谁在控制主 CPU，无论操作系统运行什么样的代码，基本的安全功能依旧保持不变。

这套系统本质上允许把任意代码放在加密功能之前，这样任意代码都不能绕过加密。这有点像是那个需要花费 80 毫秒的密码校验的超级版本。延迟的时间来源于固定的计算。这意味着它不能被绕过，并且固有的计算时间会限制你能做的事情。例如，你不能设置第五次尝试时的延时为一分钟，因为原密码构造不存在第五次尝试的概念。有了安全区域，一分钟的延时就变成了强制的，因为即使系统的其他区域被破换了，安全区域中延迟代码也不会改变。

## 软件更新

iPhone 5C（和其他没有搭载 A7 处理器的 iPhone）可以被暴力破解，通过创建一个新的没有延迟时间的操作系统，把该系统安装到设备上，以硬件最快的计算速度来破解密码。安全区域阻止了这样做。但是如果进行相同的更高级别的攻击，在安全区域中加载新的软件，那么可以去掉延迟时间么？

苹果的指南包含了在安全区域中更新软件的讨论：

> 安全区域使用了它自己的安全引导和专用的软件，与应用程序处理器的更新是隔离开的。

就是这样！没有任何细节。实际情况是什么？那么，我们必须进行猜测，因为只要我没有挖到任何有关于安全区域软件更新是如何执行的信息。我预测了两种可能性。

第一种可能性是安全区域使用的软件更新机制类似于其余的设备。也就是说，更新必须由苹果签署，但可以自由应用。这将使安全区域遭受到来自苹果本身的攻击时变得毫无用途，因为苹果可以创造新的安全区域软件解除限制。如果主操作系统是由第三方开发的，安全区域仍然是一个有用的功能，帮助保护用户。但是苹果是否可以进入它自己的设备，决定了该问题是否有必要。

第二种可能性是，安全区域的软件更新机制做了更先进的防攻击甚至是苹果本身的攻击。安全区域的整体思路是，强制执行附加的规则，以至于不能从外面绕过安全区域。这些附加的规则可能包括了其自身的软件更新。出于保护用户数据的目的，除非该设备已经由用户的密码解锁，否则安全区域拒绝任何软件的更新。一般情况下，用户忘了密码，想清除设备并重新启动，安全区域可能允许更新，但是会删除主密钥下保护的用户数据。

哪一种情况是正确答案？现在，我们不知道。苹果付出了很大的努力来保护用户数据，并且他们采取第二种方法，如果没有用户的密码进行更新会清除数据，将会很有意义。这将是相当容易实现的，并不应该影响设备的可用性。鉴于苹果对用户隐私的公开立场，我想如果它的安全区域的软件更新机制不是这样实现的会很奇怪。另一方面，蒂姆库克的公开信暗示，所有型号的苹果手机都存在潜在的弱点，所以也许他们没有采取这一额外的步骤。

回到执法者迫使苹果攻击 iOS 设备的问题，这是问题的关键。如果更新安全区域在苹果的攻击下都是牢固的，那么以联邦调查局的能力，会止步于苹果 5S 。如果不是，那么即使是最新的 6S 仍然可能被攻击。我对这个问题的答案很感兴趣。

## 结论

安全区域增加了一个额外的防线，通过在硬件独立的 CPU 中实现核心安全和加密功能，来防御攻击。这个独立的处理器运行特殊的软件，并隔离于系统的其余部分，隔离于主操作系统和内核的控制。安全区域实现了设备的密码验证、文件加密、Touch ID 识别和  Apple Pay，并执行安全限制，如过多次错误的密码输入后增加延迟时间。

FBI 要求苹果破解的 iPhone 5C 机型，早于安全区域技术之前，因此可以通过安装一个新的移除了延迟时间的苹果操作系统。新机型手机是否同样可以被破解取决于安全区域的软件更新机制的实现。如果更新软件时，没有密码就会删除加密主密钥，那么新的 iPhone 就不能被这样攻击，甚至抵御来自苹果自己的攻击。如果更新软件时，即便没有密码也不会删除主秘钥，那么安全区域就像旧版的 iPhone 一样可以被攻击。在我能得到确切的资料之前，真实情况是否如此依旧是一个悬而未决的问题。


> 本文由 SwiftGG 翻译组翻译，已经获得作者翻译授权，最新文章请访问 [http://swift.gg](http://swift.gg)。