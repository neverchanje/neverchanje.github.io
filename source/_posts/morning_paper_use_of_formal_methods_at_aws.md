---
title: Morning Paper - Use of Formal Methods at Amazon Web Services
date: 2017-1-15
type: "post"
comments: false
tags: morningpaper
---

我个人非常喜欢这篇文章。写好一篇文章其实不容易，首先要梳理好文章脉络，然后以读者的角度数易其稿，方能发表。不像我写博客，想到哪写到哪。

这是一篇 [technical report](http://glat.info/pdf/formal-methods-amazon-2014-11.pdf)。文章所讲述的是 AWS 将形式化方法广泛应用在工程上的经验和心得。AWS 从 2011 年开始使用 TLA+ 来证明分布式系统设计的正确性。TLA+ 是 Leslie Lamport 开发的（膜）一种专门用于形式化并发系统或分布式系统的语言。最初我是看到 Raft 的 Specification 是用 TLA+ 形式化描述的，才了解到了这个工具，然后读到了这篇文章。经过 TLA+ 描述之后，我们可以使用 TLC 进行 model checking，验证设计的正确性。

通常我们设计分布式系统要考虑各种 corner cases，然而时不时还是会有意想不到的 bug 出现。随着业务量的剧增，这在 AWS 是不可接受的。AWS 一开始的开发流程基于传统的方法，讲起来已经比较完善了。Design Review，Code Review，Static Code Analysis，Stress Testing，Fault-injection Testing，在国内如果流程这么完善已经非常优秀的团队了吧。然而还是不够，多种小概率事件的组合发生，还是会引发深埋逻辑里的问题。更多地，像 TLA+ 这样的方法，能够让研发人员对系统更 Confident，这非常重要。

此前我们描述一个设计的时候，会专门编写复杂的文档，然而当系统经过长时间的演化，每一个新的改动可能都需要我们考虑其对系统正确性的影响。这很麻烦。麻烦归麻烦，做还是要做的。为了减少工作量，我们可以只对核心关键的组件进行正确性证明，当然这就牺牲了一定的严谨性，牺牲一定的 confidence。还有一种方法是手写状态机，文章中提到 S3 团队画出了一个复杂的状态机，然后跑 Java 来暴力做模型检验。我挺佩服这种举措的，当然用 TLA+ 能够更好地解决这个问题。

文章里讲到他们对比过 Alloy 和 TLA+，最终他们发现 TLA+ 的表达能力更强，这是他们最早将其工程化的尝试。最早尝到甜头的这个工程师，试图将经验分享给其他人。然而大家一听到这么学术化的东西，立刻就敬而远之，无人跟进。

形式化方法这个名词听起来就难以让大众所接受，而 TLA+，一看就是实验室里跳出来的，没办法在工业界应用的样子。

后来，在 DynamoDB 开发的时候，他们 Mock 了各种网络层的故障可能，用来做 fault-injection。另一方面他们做了一些 informal proofs，保证设计的正确性，然而还是不够，还是不能保证设计没问题。设计有问题，代码就几乎不可能没问题。最终当然，DynamoDB 使用了 TLA+，效率迅速提高，甚至 Dynamo 的作者提到了，早知道有 TLA+，那他一定一开始就用了。

即使这个技术再好，如果宣传上不够接地气，还是难以推广。作者希望这个技术能够在公司范围内普及，所以他们在 slides 的一开始，没有提到 TLA+，没有提到形式化，没有提到验证（verification），很取巧地，他们把标题定为 Debugging Designs。这是个很好的经验，任何领域的技术人都需要这种技术推广的技巧。反观国内很多技术分享，要么没有考虑听众感受，没有重点；亦或者直接分享水货，大家都能听懂，但并没有什么意思。

后来 S3 团队也采纳了，并且他们发现 [PlusCal](http://research.microsoft.com/en-us/um/people/lamport/tla/pluscal.html) 更好用。PlusCal 可以用一种更像 C 的方式做设计，不像 TLA+ 那么数学化。PlusCal 可以被转换成 TLA+，然后用 TLC 做模型检验。

![TLA+ at aws](http://og0xhkmh3.bkt.clouddn.com/TLA_application_at_aws.PNG)

可以看到 TLA+ 帮助 AWS 解决了许多设计缺陷。除此之外它还有一些 side affects：

- 把系统设计形式化，可以让系统研发者更理解自己的设计。

- 梳理了整个流程之后，我们可以更好地在流程之中做 Assertion。原来我们也会经常写 assertions（感觉国内团队是不是做的少一点），但是会遇到一个问题是我们并不是总能 assert 到关键的点。有了形式化的方法，我们可以按照 spec，让代码不会违反这些保证系统正确性的 invariants。

