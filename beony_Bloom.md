# 摘要


过滤器，例如布隆过滤器、商过滤器和布谷鸟过滤器，通过维护集合的近似表示来节省空间，并且偶尔会返回假阳性。过滤器在构建现代数据密集型应用程序中起着关键作用，并且在数据库、存储引擎、计算生物学、网络安全和网络等各个领域中都有应用。在过去的几十年里，关于过滤器的研究非常广泛，导致过滤器的性能和功能得到了极大的改进。然而，现代数据密集型应用程序仍然围绕传统过滤器的局限性进行设计，导致设计复杂且性能不佳。

本文在汇集过滤器数据结构研究前沿的研究人员，帮助数据库社区了解过滤器理论和实践的最新进展。将涵盖使用现代过滤器API重新设计应用程序的实际案例研究，以实现简化和改进的应用程序性能。还将帮助揭示理论和系统中的开放研究问题，并增加研究人员之间的互动，以解决这些问题。

# 1 引言

过滤器，例如布隆过滤器[15]、商过滤器[13, 35, 45, 46, 75, 77]和布谷鸟过滤器[18, 49]，维护集合或多重集合的近似表示。通过允许查询偶尔返回假阳性，这种近似表示节省了空间。对于给定的假阳性率𝜀：对集合𝑆的过滤器进行成员查询时，对于任何𝑥 ∈ 𝑆，返回存在；对于任何𝑥 ∉ 𝑆，以至少1−𝜀的概率返回不存在。一个大小为𝑛的集合的过滤器使用的空间取决于𝜀和𝑛，但远小于显式存储集合𝑆中所有项目所需的空间。当过滤器适合缓存但底层数据不适合时，过滤器提供了性能优势。过滤器广泛应用于数据库、网络、存储系统、机器学习、计算生物学和其他领域[6,17,19,25,26,40,41,47,50,57,78,81,88,89,94,104]。例如，在数据库中，过滤器用于总结磁盘数据的内容、查询优化和表连接[9, 10, 20, 29, 30, 32, 58, 59, 80, 94, 100, 101]。在网络中，它们用于总结缓存内容、实现网络路由和维护概率测量[19]。在计算生物学中，它们用于紧凑地表示巨大的基因组数据集[4, 5, 25, 73, 74, 76, 78, 88]。

过滤器可以根据它们支持的操作进行广泛分类。静态过滤器不支持插入或删除，并且是在构建时已知的一组项目上构建的。这些过滤器的例子包括XOR过滤器[52]和Ribbon过滤器[44]。半动态过滤器支持插入但不支持删除。这类过滤器包括布隆过滤器[15]和前缀过滤器[48]。大多数情况下，静态和半动态过滤器仅适用于表示不可变数据（例如，LSM树中的文件或连接操作中的键）。相比之下，动态过滤器，如布谷鸟过滤器和商过滤器，支持插入和删除。实际上，商过滤器支持许多功能，包括（1）有效地扩展出RAM，（2）动态调整大小，（3）计算每个输入项目出现的次数（表示多重集合），（4）处理偏斜的输入分布（在DNA测序和其他实际数据集中常见），（5）将小值与键关联，一种称为maplets的过滤器变体，以及（6）随线程数量扩展（即，实现高并发）。最近的进展如InfiniFilter[35]允许动态过滤器更有效地无限扩展。我们将讨论各种现代数据密集型应用程序如何演变以利用这些功能。

最近，研究人员引入了自适应过滤器[12, 63, 68, 96]。自适应过滤器，如自适应商过滤器[12, 96]、望远镜过滤器[63]和自适应布谷鸟过滤器[68]，在检测到假阳性时更新其结构，以避免将来重复相同的错误。此外，自适应商过滤器是单调自适应的，这意味着即使对于对抗性工作负载，其性能和假阳性概率保证仍然有效。另一方面，范围过滤器[1, 43, 51, 102, 103]是经典点过滤问题的推广，允许区间作为过滤器输入。目前，范围过滤器主要用于数据库，以快速确定给定键范围是否为空，从而避免不必要的磁盘I/O操作。

另一方面，范围过滤器[1, 43, 51, 102, 103]是经典点过滤问题的推广，允许区间作为过滤器输入。目前，范围过滤器主要用于数据库，以快速确定给定键范围是否为空，从而避免范围查询中不必要的磁盘I/O操作。


## 1.1  目标

主要目标是对于过滤器数据结构研究前沿的研究人员，使理论、系统和应用社区了解最新的发展。这将进一步帮助揭示理论和系统中的开放研究问题，并增加研究人员之间的互动，以解决这些问题。

帮助研究人员了解过滤器理论和实践的最新进展，特别是，与传统过滤器（如布隆过滤器）相比，现代过滤器支持的高级API。这将使读者了解在应用程序中使用过滤器的现代方法，以及如何充分利用过滤器的潜力来实现性能和可扩展性。例如，系统开发人员仍然以传统方式使用布隆过滤器，未能充分发挥性能。使用具有高级功能的现代过滤器（如基于指纹和自适应过滤器）可以帮助减少开发时间，并简化系统管道。

最后，本文将有助于促进从事过滤器数据结构研究的研究人员之间，以及核心数据结构研究人员与应用/系统研究人员之间的新合作。

# 2 过滤器类型

过滤器数据结构可以大致分为静态、半动态和动态。静态过滤器近似表示一组在构建过滤器之前必须已知的项目。这些过滤器的例子包括XOR过滤器[52]和Ribbon过滤器[44]。半动态过滤器，如布隆过滤器[15]和前缀过滤器[48]，支持在不预先知道键集合的情况下进行插入。然而，必须预先知道集合的大小才能保证假阳性率。因此，半动态过滤器通常仅用于表示静态键集合。

相比之下，动态过滤器近似表示一组在构建之前不需要已知的项目。由于应用程序通常不知道预先的项目集合，动态过滤器在近年来得到了更多的发展。动态过滤器的例子包括商过滤器[13, 35, 45, 46, 71, 75]和布谷鸟过滤器[18, 49]。本教程将详细介绍这些过滤器的机制及其多种变体的权衡。

已知动态过滤器需要 $$n \log \epsilon^{-1} + \Omega(n)$$ 位。例如，商过滤器使用 $$n \log \epsilon^{-1} + 2.125n$$位[75]，而布谷鸟过滤器使用 $$n \log \epsilon^{-1} + 3n$$ 位[49]。当𝜀具有典型值如 $$2^{-8}$$ 或 $$2^{-16}$$ 时，与使用 $$n \log \epsilon^{-1}$$ 的过滤器相比，2.125𝑛位的开销将空间增加了25%（分别为12.5%）。（注意，布隆过滤器使用 $$1.44n \log \epsilon^{-1}$$ ，因此仅在𝜀接近1的非实际情况下，它在空间上优于现代过滤器。）

## 2.1 基于指纹的动态过滤器

大多数动态过滤器都是基于指纹的。它们使用**商**方法紧凑且精确地存储小的、有损的指纹[71]。在商方法中，一个𝑝位的指纹ℎ(𝑥)被分为两部分：前𝑞位 $$h_0(x)$$ 称为商，剩余的𝑟 = 𝑝−𝑞位 $$h_1(x)$$ 称为余数。为了节省空间，商被隐式存储，而只有余数被显式存储。

商过滤器[13, 45, 46, 75, 77]使用Robin Hood哈希（线性探测的一种变体）将余数存储在一个线性表中。为了解决软冲突（即两个指纹共享相同的商但有不同的余数），它使用2-3个额外的元数据位来解决冲突并高效地执行查询。另一方面，布谷鸟过滤器[18, 49]使用一个4路关联表来存储余数和3个元数据位。它采用布谷鸟哈希来解决冲突。每个项目被哈希到表中的两个位置，并通过踢出操作为新进入的项目腾出空间。其他变体如Crate[14]和前缀过滤器[48]通过链式哈希桶来解决冲突。

## 2.2 可扩展过滤器

在许多使用过滤器的应用程序中，数据大小会随着时间的推移而增长。这就需要过滤器能够随着数据的增长而扩展。例如，许多现代的日志结构化键值存储使用内存中的maplet来映射存储中每个数据条目的位置[7, 21, 38, 82]。随着数据大小的增长，maplet必须扩展以映射更多的键及其存储位置。在网络社区中，可扩展过滤器最近被指出对于支持黑名单和多播路由至关重要[97]。在架构社区中，可扩展过滤器被提议用于实现页表[87]。

所有过滤器在扩展方面的共同难点在于它们存储的是键的哈希表示，而不是原始键本身。因此，扩展时无法像常规哈希表那样重新哈希原始键。显而易见的解决方法是扫描原始数据并从头开始构建一个容量更大的过滤器。然而，遍历整个数据集的成本可能是不可承受的。另一种可能性是预先分配一个非常大的过滤器，但这从一开始就浪费了大量内存，并限制了过滤器最终可以表示的集合大小。还有一种选择是创建一个过滤器链（即链表），并随着数据的增长向该链添加新的过滤器。一些工作建议向该链添加固定大小的过滤器[24, 53, 54]，而另一些则建议添加容量几何级数增加的过滤器[2, 98]。无论哪种方式，这种方法都会增加查询成本，因为在查询过程中可能需要搜索链上的所有过滤器[2, 72, 98]。

一些最近的过滤器概念上形成了一个桶的哈希环，以支持弹性扩展[65, 99]。问题在于查询、删除和插入都变成了对数时间复杂度，因为必须搜索一个搜索树以找到给定条目的桶。

商过滤器提供了有限的扩展支持：可以通过牺牲每个指纹的一个位来将其容量加倍，并将其均匀映射到扩展的哈希表中。问题在于，随着数据的增长，指纹会缩小，这会增加假阳性率（FPR）。最终，指纹位会耗尽，此时过滤器对每个查询都返回阳性，并且无法继续扩展。

受理论构造[72]启发，Taffy布谷鸟过滤器[8]允许扩展到已知的宇宙大小，同时保持快速查询和稳定的假阳性率（FPR），尽管它不支持删除。其核心思想是通过在每个槽位上增加自限定一元填充来支持可变长度指纹的哈希表。这允许在扩展期间牺牲每个现有指纹的一个位，将其均匀映射到更大的哈希表，同时仍然为扩展后插入的新条目设置长指纹。InfiniFilter[35]改进了这些想法，支持删除并扩展到无限的宇宙大小，尽管查询不是常数时间。最近的Aleph过滤器[34]在所有操作上提供了常数时间保证，改进了InfiniFilter。

## 2.3 自适应过滤器

自适应过滤器是一种过滤器，对于每个负查询，其返回True的概率最多为𝜀，无论之前查询的答案如何。对于使用自适应过滤器的字典，任何𝑛个负查询的序列将以高概率导致𝑂(𝜀𝑛)个假阳性。这对字典需要进行的（昂贵的）负访问次数提供了强有力的保证。即使查询是由自适应对手选择的，这也是成立的。

（negative query “负查询” 指的是查询一个不存在于过滤器中的元素）

自适应过滤器是传统过滤器的泛化，使过滤器能够用于一系列不同的问题。之前的工作分别考虑了这些问题，并因此基于传统过滤器开发了不同的方法来解决每个问题。例子包括级联布隆过滤器[91]、静态XOR过滤器[83]、跷跷板计数过滤器[64]、伸缩过滤器[63]和自适应布谷鸟过滤器[68]。所有这些过滤器变体在性能和准确性保证方面都表现得次优。

Bender等人[11, 12]定义了自适应过滤器的概念，它对应用程序将看到的假阳性数量提供了强有力的保证，即使在偏斜或对抗性查询分布的情况下，并提出了满足其定义的扫帚过滤器。Bender等人[11]分析了遵循Zipfian分布的查询上扫帚过滤器[12]的性能。Lee等人[63]提出了伸缩过滤器来解决偏斜查询分布问题。Mitzenmacher等人[68]提出了自适应布谷鸟过滤器来解决偏斜查询分布问题。最近，Wen等人[96]提出了自适应商过滤器的实际实现，该过滤器是单调自适应的，并提供了强有力的性能保证。

## 2.4 Maplets

在从基因组学[67]到存储引擎[7, 21, 28, 38, 82]的各种应用中，将一个小值与过滤器中的每个键关联起来是很有用的。这种键值过滤器被称为maplets[28]。对maplet中现有键的查询必须返回与该目标键关联的值，并可能返回一些额外的任意值。此外，大多数maplets在查询不存在的键时会返回任意值。应用程序应当处理查询结果中的噪声（例如，通过额外的工作来检查哪个是真实值）。现有的maplets可以根据正查询和负查询返回的平均值数量进行分类，即预期的正结果大小（PRS）和预期的负结果大小（NRS）。

Bloomier过滤器[22]可以被认为是存储固定宽度值而不是指纹的XOR过滤器[52]。虽然它支持更新与现有键关联的值，但不支持插入新数据条目，它的PRS和NRS都是1。

最近的布谷鸟过滤器和商过滤器可以通过扩展哈希表中的每个槽位以同时存储值和键的指纹，轻松转换为maplets[28, 38]。这种maplets的PRS为1 + 𝜀，NRS为𝜀。它们支持动态插入和删除，并且可以像第2.2节中所示那样进行扩展。基于商过滤器的maplets在基因组应用中广泛用于构建倒排字符串索引[3–5, 73]。

最近的存储引擎在设计具有PRS为1的动态maplets方面开创了先河，以限制尾延迟。它们通过检测和消除插入路径上的指纹冲突来实现这一点。SlimDB使用一个辅助字典来存储与现有指纹冲突的插入的完整键[82]。相比之下，Pliops Delta哈希表是一个maplet，通过简洁地编码哈希表桶内所有指纹的第一个区分位的索引来解决指纹冲突[39]。

一些maplets需要为每个键存储多个值甚至可变大小的值[38, 73]。商过滤器在这方面非常擅长，因为它们使用线性探测允许在相邻槽位中存储可变大小的条目，并在一个运行中存储所需数量的值。

## 2.5 范围过滤器

范围过滤是经典点过滤问题的推广，允许将区间作为过滤器输入。它也被称为𝜀-近似范围空性问题[51]：给定一个集合S，如何确定区间𝐼 = [𝑎, 𝑏]的“空性”（即[𝑎, 𝑏]∩𝑆 = ∅？）的假阳性概率<𝜀？Goswami等人[51]给出了能够回答范围空性查询的任何数据结构（即范围过滤器）的最坏情况空间下界：Ω(𝑛lg(𝐿/𝜀))−𝑂(𝑛)位，其中𝑛是集合S中的点数，𝐿是允许的最大区间长度。目前，范围过滤器主要用于基于LSM树的存储引擎（如RocksDB）中，以减少范围查询（如SELECT * FROM T WHERE T.year BETWEEN 2020 AND 2024）的不必要I/O操作。

Hekaton[43]中引入的自适应范围过滤器（ARF）[1]被认为是构建实用范围过滤器的首次尝试。ARF使用二叉树对整个整数键空间进行编码。ARF仅在稳定或重复的整数工作负载下表现良好。然而，高训练开销阻止了ARF高效解决一般范围过滤问题。

第一个通用范围过滤器是简洁范围过滤器（SuRF）[102, 103]。SuRF使用trie结构存储集合中键的唯一前缀。该trie的编码空间接近信息论下界。SuRF然后使用每个键（哈希）后缀的附加位在空间和假阳性率（FPR）之间进行权衡。尽管SuRF开创了实用范围过滤器的研究，并在RocksDB上实现了令人印象深刻的加速，但它有主要的局限性。首先，SuRF缺乏空间和范围查询FPR的理论保证。对抗性工作负载（例如，每对键产生一个唯一的长前缀）可以破坏SuRF的空间效率。其次，SuRF是一个静态数据结构，这限制了其在需要频繁更新过滤器的应用中的使用。

## 2.6 计数过滤器

计数过滤器将传统的点过滤器推广到多重集。它们支持插入、查询和删除操作，不同之处在于对项𝑥的查询返回𝑥被插入的次数。计数过滤器可能有一个错误率𝛿。查询以至少1−𝛿的概率返回真实计数。每当查询返回错误计数时，它必须总是大于真实计数。

计数布隆过滤器（CBF）是计数过滤器的早期例子。CBF最初被描述为使用固定大小的计数器，这意味着计数器可能会饱和。这可能导致计数布隆过滤器低估计数。一旦计数器饱和，任何未来的删除操作都无法减少计数器的值，因此在多次删除后，计数布隆过滤器可能不再满足其错误限制𝛿。这两个问题都可以通过在计数器饱和时用更大的计数器重建整个数据结构来解决。

d-left Bloom过滤器[16]提供与计数布隆过滤器相同的功能，并且使用更少的空间，通常节省两倍或更多。它使用d-left哈希并提供更好的数据局部性。然而，它不可调整大小，并且假阳性率取决于构建数据结构时使用的块大小。光谱布隆过滤器[27]是另一种计数布隆过滤器的变体，旨在高效地支持偏斜输入分布。光谱布隆过滤器通过使用可变大小的计数器来节省空间。与普通计数布隆过滤器相比，它在偏斜输入分布上提供了显著的空间节省。然而，像其他布隆过滤器变体一样，光谱布隆过滤器具有较差的缓存局部性，并且不能调整大小。

计数商过滤器（CQF）[75]是一种空间高效且可扩展的计数过滤器，在任意输入分布（包括高度偏斜的分布）上提供良好的性能。CQF基于商过滤器，并使用可变长度编码的计数器来实现编码计数器的渐近最优空间。可变长度编码还使CQF能够高效处理在现实世界数据集中经常看到的高度偏斜的分布。

## 2.7 静态过滤器（XOR/Ribbon）

在某些过滤器应用中，例如日志结构合并树（LSM）[20, 32]，过滤器用于加速成员查询（但不是后继查询），过滤器构建的集合是预先已知的。由于静态过滤器理论上可以比动态过滤器更小，并且由于RAM通常不足以容纳所有需要的过滤器以实现LSM中的性能，问题变成：是否存在几乎达到$$n \log \epsilon^{-1}$$位空间的静态过滤器，同时构建速度合理且查询速度非常快？

研究人员开发了所谓的代数过滤器，这些过滤器计算要过滤的集合的表示。XOR过滤器是第一个这样的过滤器，它实现了$$1.22n \log \epsilon^{-1}$$位。XOR+过滤器实现了$$1.08n \log \epsilon^{-1} + 0.5n$$位，这对于现实的𝜀更好。Ribbon过滤器在某些假设下将这个界限改进到$$1.005n \log \epsilon^{-1} + 0.008n$$位，并且具有更好的构建和查询时间。Ribbon过滤器在基于LSM树的RocksDB中可用。当空间绝对紧缺时，它可能有用，尽管其查询时间仍然比快速竞争过滤器慢。

## 2.8 利用查询分布

最近的一类过滤器利用查询工作负载的知识来改善假阳性率和/或空间。为了操作，这些过滤器需要一份历史查询的样本作为输入。然后，它们可以训练一个分类器来预测每个潜在键被查询的可能性以及其在数据集中存在的概率[61, 79]。这样的分类器可以用来学习预测频繁访问的正键的正面结果，从而避免将它们插入常规过滤器以节省空间。相比之下，堆叠过滤器[42]利用对频繁查询的不存在键的知识，将它们插入到额外过滤器的层次结构中，从而在查询它们时指数级地减少假阳性率。

# 3 应用

## 3.1 存储引擎和数据库
存储引擎是数据库系统的组件，它将数据布局在存储设备上，使其具有结构。大多数现代存储引擎在存储中松散地组织数据，以优化数据摄取。同时，它们使用内存中的过滤器或映射器来促进查询。特别是，LSM树[70]是一种写优化的数据结构，现在是许多存储引擎和键值存储（如RocksDB、Cassandra、HBase、SplinterDB等）的核心。它通过将传入数据作为小的排序文件刷新到存储中，并逐渐进行排序合并来工作。通常，每个文件在内存中都有一个布隆过滤器，以允许点查询跳过不包含目标条目的文件。然而，由于每个文件一旦创建就不可变，任何静态过滤器在这种情况下都是适用的。此外，LSM树支持范围查询的需求启动了关于范围过滤器的研究，这在第2.5节中讨论。

过滤器为LSM树提供了更深层次的优化机会[85]。Monkey [32, 33]通过为较小的文件分配较小的假阳性率，将查询成本从𝑂(𝜀 · lg𝑁)减少到𝑂(𝜀) I/O操作，因为假阳性率的总和随着系统中文件数量的增加而收敛。Dostoevsky [36]和LSMBush [37]系统进一步利用这一技术，更懒惰地压缩较小的文件，同时为它们设置较低的假阳性率。通过这样做，它们在不损害查询成本或内存占用的情况下，将写放大从𝑂(lg𝑁)减少到𝑂(lglg𝑁)[36, 37]。

另一种工作流用单个映射器替换LSM树的多个过滤器，该映射器将系统中的每个键映射到相应数据条目所在的文件。SlimDB [82]率先采用这种方法，通过使用辅助字典解决所有指纹冲突来消除假阳性。Chucky [38]通过使用霍夫曼编码压缩文件标识符来减少此类映射器的内存占用。SplinterDB[28]为一组文件而不是每个单独的文件使用一个映射器，以显著减少查询CPU开销。GRF [93]是利用SNARF [92]的LSM树的最新全局范围过滤器。

循环日志是另一类最近的存储引擎，它们比LSM树更优化写入摄取。循环日志将所有应用程序插入/更新/删除作为日志记录刷新到存储中的仅追加文件中，并偶尔对该日志进行垃圾回收以删除过时的条目[7, 21, 39]。为了高效地在此日志中查找条目，内存中有一个映射器将日志中的每个条目映射出来。这些映射器必须支持更新、删除和扩展，以反映对基础数据的修改和添加。它们还必须表现出高性能和低假阳性率。有趣的是，我们所知道的系统中没有使用满足这些要求的映射器，因此我们预计最近和正在进行的关于映射器的研究将在未来几年对这一领域产生重大影响。

过滤器数据结构也广泛用于处理选择性等值连接。一种常见的方法是对较小表中的合格连接键构建过滤器[62]。当扫描较大表时，我们可以将其连接键与此过滤器进行检查，以预先丢弃较小表中不匹配的行。这有助于减少连接分区的数量和大小，从而提高CPU利用率和I/O效率。

## 3.2 计算生物学
过滤器被广泛用于以𝑘-mer（长度为𝑘的子串）的形式表示大型基因组数据。将基因组数据表示为𝑘-mer使应用程序能够高效地执行序列级搜索和基因组组装。Solomon和Kingsford[88]首次引入了实验发现问题，其中对𝑘-mer集合𝑞的查询必须返回包含集合𝑞中一部分Θ的所有实验。他们引入了序列布隆树（SBT）[55, 88]，这是一种布隆过滤器的二叉树。SBT的叶子表示单个测序实验，相关的布隆过滤器包含该实验中存在的𝑘-mer。内部节点𝑛的布隆过滤器表示以𝑛为根的子树的叶子中存在的（近似）𝑘-mer集合。由于布隆过滤器中的假阳性，SBT索引在最终结果中也存在假阳性。

Mantis[3–5, 73]采用倒排索引方法来支持快速高效的序列级搜索。与作为近似索引的SBT相比，Mantis被证明更小、更快且更精确。后续版本的Mantis改进了可扩展性，并索引了来自SRA的多达40K个实验，包含超过100TB的测序数据。Mantis使用计数商过滤器[75]作为映射器，将𝑘-mer映射到包含该𝑘-mer的实验集合。每个𝑘-mer与一个长度等于实验数量的唯一位向量相关联。此外，商过滤器通过使用与原始键大小匹配的指纹来实现精确映射。

布隆过滤器和商过滤器已被用于紧凑地表示de Bruijn图。在de Bruijn图中，每个节点是来自底层生物样本的𝑘长度子序列，如果两个节点共享一个（𝑘−1）长度的子序列，则它们通过一条边连接。Pell等人[78]引入了一种使用布隆过滤器表示底层𝑘-mer集合的de Bruijn图的概率表示。尽管这种表示在边集上允许假阳性，但他们观察到，除非假阳性率非常高（即≥0.15），否则这对图的宏观结构影响很小。在这种概率表示的基础上，Chikhi和Rizk[25]引入了一种精确的de Bruijn图表示，该表示将基于布隆过滤器的近似de Bruijn图与存储关键假阳性边的精确表结合起来。Chikhi和Rizk的de Bruijn图表示利用了这样一个事实：在𝑘-mer集合的布隆过滤器表示中，连接真阳性𝑘-mer和假阳性𝑘-mer的边非常少。这些边被称为关键假阳性。他们观察到，消除这些关键假阳性足以提供精确的（导航）表示。

随后，Salikhov等人[84]通过用级联布隆过滤器替换精确表进一步改进了内存需求。级联布隆过滤器使用集合的近似（即基于布隆过滤器的）表示和一个较小的表来记录相关的假阳性，从而存储近似集合。这种结构可以递归应用，以大幅减少表示原始集合所需的内存量。deBGR[76]将基于过滤器的de Bruijn图表示推广到加权de Bruijn图。deBGR表示基于CQF[75]，后者本身提供了加权de Bruijn图的近似表示。Pandey等人观察到，在精确的加权de Bruijn图中存在某些与丰度相关的不变量，并设计了一种算法，使用这种近似数据表示来迭代地自我纠正结构中的近似误差。deBGR方法的一个主要优点是，加权de Bruijn图构建所需的内存远少于其他工具所需的内存，并且工作内存接近结构所需的最终内存。因此，整个过程可以以空间高效的方式完成。deBGR表示允许在较小且成本较低的计算机上组装更大和更复杂的转录组（通常用于大规模转录组组装的机器具有超过1TB的RAM）。此外，鉴于CQF提供的功能，de Bruijn图表示（至少部分）是动态的，允许从deBGR中删除𝑘-mer并高效地更新生成的数据结构。这种能力对于在组装前通常进行的de Bruijn图简化（例如，去除尖端和弹出气泡）非常重要。

## 3.3 网络和分布式系统

过滤器已广泛应用于网络和网络安全领域[19]。恶意网站对互联网用户构成重大威胁。例如，仅仅访问一个恶意URL就可能导致用户的网页浏览器被劫持[90]。用户也可能被主动诱骗下载有害材料或分享敏感信息，如密码或信用卡号码。由于URL很长[56]且数量众多[86]，路由器阻止恶意URL的有效方法是将它们存储为过滤器的yes列表[64]。

一种解决假阳性成本变化的方法是将重要的假阳性存储在no列表中，这样它们就永远不会被阻止，也不会支付URL验证的代价。Chazellete等人[23]引入了Bloomier过滤器，解决了yes/no列表问题。Li等人[64]提出了Seesaw计数过滤器（SSCF），专门为恶意URL阻止问题实现了yes/no列表过滤器。Reviriego等人[83]提出了集成过滤器，也实现了no列表。两者都专注于no列表是静态且预先已知的情况。SSCF有一个扩展，可以动态添加no列表项，但这样做不能保证防止假阳性，并且还可能引入假阴性。最近，Wen等人[96]表明，在静态和动态情况下，使用自适应过滤器可以有效地解决yes/no列表问题。

# 4 目标受众
本文面向核心数据结构和数据库研究人员以及在工业中构建生产应用程序的研究人员。研究核心索引数据结构和数据库内部机制的研究人员将学习现代过滤器理论和实践的最新进展。应用开发人员和行业专家将了解最新的过滤器API和在现代硬件上的性能。


# 参考文献

[1] Karolina Alexiou, Donald Kossmann, and Per-Åke Larson. 2013. Adaptive range filters for cold data: Avoiding trips to Siberia. Proceedings of the VLDB Endowment 6, 14 (2013), 1714–1725.

[2] Paulo Sérgio Almeida, Carlos Baquero, Nuno Preguiça, and David Hutchison. 2007. Scalable Bloom Filters. Inform. Process. Lett. (2007).

[3] Fatemeh Almodaresi, Jamshed Khan, Sergey Madaminov, Michael Ferdman, Rob Johnson, Prashant Pandey, and Rob Patro. 2022. An incrementally updatable and scalable system for large-scale sequence search using the Bentley–Saxe transformation. Bioinformatics 38, 12 (March 2022), 3155–3163. https://doi.org/10.1093/bioinformatics/btac142

[4] Fatemeh Almodaresi, Prashant Pandey, Michael Ferdman, Rob Johnson, and Rob Patro. 2019. An Efficient, Scalable and Exact Representation of High-Dimensional Color Information Enabled via de Bruijn Graph Search. In International Conference on Research in Computational Molecular Biology (RECOMB). Springer, 1–18.

[5] Fatemeh Almodaresi, Prashant Pandey, Michael Ferdman, Rob Johnson, and Rob Patro. 2020. An Efficient, Scalable, and Exact Representation of High-Dimensional Color Information Enabled Using de Bruijn Graph Search. Journal of Computational Biology 27, 4 (2020), 485–499.

[6] Sattam Alsubaiee, Alexander Behm, Vinayak Borkar, Zachary Heilbron, Young-Seok Kim, Michael J Carey, Markus Dreseler, and Chen Li. 2014. Storage management in AsterixDB. Proceedings of the VLDB Endowment 7, 10 (2014), 841–852.

[7] David G. Andersen, Jason Franklin, Michael Kaminsky, Amar Phanishayee, Lawrence Tan, and Vijay Vasudevan. 2009. FAWN: A Fast Array of Wimpy Nodes. SOSP (2009).

[8] Jim Apple. 2022. Stretching your data with taffy filters. Software: Practice and Experience (2022).

[9] Berk Atikoglu, Yuehai Xu, Eitan Frachtenberg, Song Jiang, and Mike Paleczny. 2012. Workload analysis of a large-scale key-value store. In Proceedings of the 12th ACM SIGMETRICS/PERFORMANCE Joint International Conference on Measurement and Modeling of Computer Systems. 53–64.

[10] Michael A. Bender, Alex Conway, Martin Farach-Colton, William Jannen, Yizheng Jiao, Rob Johnson, Eric Knorr, Sara McAllister, Nirjhar Mukherjee, Prashant Pandey, Donald E. Porter, Jun Yuan, and Yang Zhan. 2021. External-memory Dictionaries in the Affine and PDAM Models. ACM Trans. Parallel Comput. 8, 3 (2021), 15:1–15:20. https://doi.org/10.1145/3470635

[11] Michael A. Bender, Rathish Das, Martín Farach-Colton, Tianchi Mo, David Tench, and Yung Ping Wang. 2021. Mitigating False Positives in Filters: to Adapt or to Cache?. In Proc. 2nd Symposium on Algorithmic Principles of Computer System (APoCS).

[12] Michael A. Bender, Martin Farach-Colton, Mayank Goswami, Rob Johnson, Samuel McCauley, and Shikha Singh. 2018. Bloom Filters, Adaptivity, and the Dictionary Problem. In Proc. 59th Annual IEEE Symposium on Foundations of Computer Science (FOCS). Paris, France, 182–193.

[13] Michael A. Bender, Martin Farach-Colton, Rob Johnson, Russell Kaner, Bradley C. Kuszmaul, Dzejla Medjedovic, Pablo Montes, Pradeep Shetty, Richard P. Spillane, and Erez Zadok. 2012. Don’t Thrash: How to Cache Your Hash on Flash. Proceedings of the VLDB Endowment 5, 11 (2012).

[14] Ioana O Bercea and Guy Even. 2020. Fully-Dynamic Space-Efficient Dictionaries and Filters with Constant Number of Memory Accesses. SWAT.

[15] Burton H. Bloom. 1970. Space/time Trade-offs in Hash Coding With Allowable Errors. Commun. ACM 13, 7 (1970), 422–426.

[16] Flavio Bonomi, Michael Mitzenmacher, Rina Panigrahy, Sushil Singh, and George Varghese. 2006. An improved construction for counting Bloom filters. In European Symposium on Algorithms (ESA). Springer, 684–695.

[17] Phelim Bradley, Henk C Den Bakker, Eduardo PC Rocha, Gil McVean, and Zamin Iqbal. 2019. Ultrafast search of all deposited bacterial and viral genomic data. Nature biotechnology 37, 2 (2019), 152–159.

[18] Alex D Breslow and Nuwan S Jayasena. 2018. Morton filters: faster, space-efficient cuckoo filters via biasing, compression, and decoupled logical sparsity. Proceedings of the VLDB Endowment 11, 9 (2018), 1041–1055.

[19] Andrei Broder and Michael Mitzenmacher. 2004. Network applications of Bloom filters: A survey. Internet Mathematics 1, 4 (2004), 485–509.

[20] Zhichao Cao, Siying Dong, Sagar Vemuri, and David HC Du. 2020. Characterizing, modeling, and benchmarking RocksDB key-value workloads at Facebook. In 18th USENIX Conference on File and Storage Technologies (FAST). 209–223.

[21] Badrish Chandramouli, Guna Prasaad, Donald Kossmann, Justin J Levandoski, James Hunter, and Mike Barnett. 2018. FASTER: A Concurrent Key-Value Store with In-Place Updates. SIGMOD (2018).

[22] Bernard Chazelle, Joe Kilian, Ronitt Rubinfeld, and Ayellet Tal. 2004. The Bloomier filter: an efficient data structure for static support lookup tables. In Symposium on Discrete Algorithms.

[23] Bernard Chazelle, Joe Kilian, Ronitt Rubinfeld, and Ayellet Tal. 2004. The Bloomier filter: an efficient data structure for static support lookup tables. In Proceedings of the fifteenth annual ACM-SIAM symposium on Discrete algorithms. Society for Industrial and Applied Mathematics, 30–39.

[24] Hanhua Chen, Liangyi Liao, Hai Jin, and Jie Wu. 2017. The Dynamic Cuckoo Filter. In ICNP.

[25] Rayan Chikhi and Guillaume Rizk. 2013. Space-efficient and exact de Bruijn graph representation based on a Bloom filter. Algorithms for Molecular Biology 8, 1 (2013), 22.

[26] Justin Chu, Sara Sadeghi, Anthony Raymond, Shaun D Jackman, Ka Ming Nip, Richard Mar, Hamid Mohamadi, Yaron S Butterfield, A Gordon Robertson, and Inanc Birol. 2014. BioBloomtools: fast, accurate and memory-efficient host species sequence screening using bloom filters. Bioinformatics 30, 23 (2014), 3402–3404.

[27] Saar Cohen and Yossi Matias. 2003. Spectral Bloom filters. In Proceedings of the ACM International Conference on Management of Data (SIGMOD). 241–252.

[28] Alex Conway, Martín Farach-Colton, and Rob Johnson. 2023. SplinterDB and Maplets: Improving the Tradeoffs in Key-Value Store Compaction Policy. Proceedings of the ACM on Management of Data (2023).

[29] Alexander Conway, Martin Farach-Colton, and Philip Shilane. 2018. Optimal Hashing in External Memory. In ICALP (LIPIcs, Vol. 107). Schloss Dagstuhl - Leibniz-Zentrum für Informatik, 39:1–39:14.

[30] Alexander Conway, Abhishek Gupta, Vijay Chidambaram, Martin Farach-Colton, Richard Spillane, Amy Tai, and Rob Johnson. 2020. SplinterDB: Closing the Bandwidth Gap for NVMe Key-Value Stores. In 2020 USENIX Annual Technical Conference (USENIX ATC 20). 49–63.

[31] Marco Costa, Paolo Ferragina, and Giorgio Vinciguerra. 2023. Grafite: Taming Adversarial Queries with Optimal Range Filters. arXiv preprint arXiv:2311.15380 (2023).

[32] Niv Dayan, Manos Athanassoulis, and Stratos Idreos. 2017. Monkey: Optimal Navigable Key-Value Store. SIGMOD (2017).

[33] Niv Dayan, Manos Athanassoulis, and Stratos Idreos. 2018. Optimal Bloom Filters and Adaptive Merging for LSM-Trees. TODS 43, 4 (2018), 16:1–16:48.

[34] Niv Dayan, Ioana Bercea, and Rasmus Pagh. 2024. Aleph Filter: To Infinity in Constant Time. arXiv preprint arXiv:2404.04703 (2024).

[35] Niv Dayan, Ioana Bercea, Pedro Reviriego, and Rasmus Pagh. 2023. InfiniFilter: Expanding Filters to Infinity and Beyond. Proceedings of the ACM on Management of Data (2023).

[36] Niv Dayan and Stratos Idreos. 2018. Dostoevsky: Better Space-Time Trade-Offs for LSM-Tree Based Key-Value Stores via Adaptive Removal of Superfluous Merging. SIGMOD (2018).

[37] Niv Dayan and Stratos Idreos. 2019. The Log-Structured Merge-Bush & the Wacky Continuum. In SIGMOD.

[38] Niv Dayan and Moshe Twitto. 2021. Chucky: A Succinct Cuckoo Filter for LSM-Tree. In SIGMOD.

[39] Niv Dayan, Moshe Twitto, Yuval Rochman, Uri Beitler, Itai Ben Zion, Edward Bortnikov, Shmuel Dashevsky, Ofer Frishman, Evgeni Ginzburg, Igal Maly, et al. 2021. The End of Moore’s Law and the Rise of the Data Processor. VLDB (2021).

[40] Biplob Debnath, Sudipta Sengupta, Jin Li, David J Lilja, and David HC Du. 2011. BloomFlash: Bloom filter on flash-based storage. In Proceedings of the 31st International Conference on Distributed Computing Systems (ICDCS). 635–644.

[41] Biplob K Debnath, Sudipta Sengupta, and Jin Li. 2010. ChunkStash: Speeding Up Inline Storage Deduplication Using Flash Memory. In Proceedings of the USENIX Annual Technical Conference (ATC).

[42] Kyle Deeds, Brian Hentschel, and Stratos Idreos. 2020. Stacked filters: learning to filter by structure. PVLDB (2020).

[43] Cristian Diaconu, Craig Freedman, Erik Ismert, Per-Ake Larson, Pravin Mittal, Ryan Stonecipher, Nitin Verma, and Mike Zwilling. 2013. Hekaton: SQL server’s memory-optimized OLTP engine. In Proceedings of the 2013 ACM SIGMOD International Conference on Management of Data. 1243–1254.

[44] Peter C Dillinger, Lorenz Hübschle-Schneider, Peter Sanders, and Stefan Walzer. 2022. Fast Succinct Retrieval and Approximate Membership Using Ribbon. SEA (2022).

[45] Peter C. Dillinger and Panagiotis (Pete) Manolios. 2009. Fast, All-Purpose State Storage. In Proceedings of the 16th International SPIN Workshop on Model Checking Software (Grenoble, France). Springer-Verlag, Berlin, Heidelberg, 12–31. https://doi.org/10.1007/978-3-642-02652-2_6

[46] Gil Einziger and Roy Friedman. 2016. Counting with TinyTable: Every Bit Counts!. In Proceedings of the 17th International Conference on Distributed Computing and Networking (Singapore, Singapore) (ICDCN ’16). Association for Computing Machinery, New York, NY, USA, Article 27, 10 pages. https://doi.org/10.1145/2833312.2833449

[47] John Esmet, Michael A. Bender, Martin Farach-Colton, and Bradley C. Kuszmaul. 2012. The TokuFS Streaming File System. In Proc. 4th USENIX Workshop on Hot Topics in Storage (HotStorage). Boston, MA, USA.

[48] Tomer Even, Guy Even, and Adam Morrison. 2022. Prefix Filter: Practically and Theoretically Better Than Bloom. Proc. VLDB Endow. 15, 7 (2022), 1311–1323. https://doi.org/10.14778/3523210.3523211

[49] Bin Fan, Dave G Andersen, Michael Kaminsky, and Michael D Mitzenmacher. 2014. Cuckoo Filter: Practically Better Than Bloom. In Proceedings of the 10th ACM International on Conference on emerging Networking Experiments and Technologies. ACM, 75–88.

[50] Martin Farach-Colton, Rohan J. Fernandes, and Miguel A. Mosteiro. 2009. Bootstrapping a hop-optimal network in the weak sensor model. ACM Trans. Algorithms 5, 4 (2009), 37:1–37:30.

[51] Mayank Goswami, Allan Grønlund, Kasper Green Larsen, and Rasmus Pagh. 2014. Approximate range emptiness in constant time and optimal space. In Proceedings of the twenty-sixth annual ACM-SIAM symposium on Discrete algorithms. SIAM, 769–775.

[52] Thomas Mueller Graf and Daniel Lemire. 2020. Xor Filters: Faster and Smaller Than Bloom and Cuckoo Filters. JEA (2020).

[53] Deke Guo, Jie Wu, Honghui Chen, and Xueshan Luo. 2006. Theory and Network Applications of Dynamic Bloom Filters. In INFOCOM.

[54] Deke Guo, Jie Wu, Honghui Chen, Ye Yuan, and Xueshan Luo. 2009. The Dynamic Bloom Filters. IEEE Trans Knowl Data Eng (2009).

[55] Robert S. Harris and Paul Medvedev. 2019. Improved representation of sequence bloom trees. Bioinformatics 36, 3 (Aug. 2019), 721–727. https://doi.org/10.1093/bioinformatics/btz662

[56] InternetLiveStats.com. 2022. Google search statistics. https://www.internetlivestats.com/google-search-statistics/

[57] Shaun D Jackman, Benjamin P Vandervalk, Hamid Mohamadi, Justin Chu, Sarah Yeo, S Austin Hammond, Golnaz Jahesh, Hamza Khan, Lauren Coombe, Rene L Warren, et al. 2017. ABySS 2.0: resource-efficient assembly of large genomes using a Bloom filter. Genome research 27, 5 (2017), 768–777.

[58] William Jannen, Jun Yuan, Yang Zhan, Amogh Akshintala, John Esmet, Yizheng Jiao, Ankur Mittal, Prashant Pandey, Phaneendra Reddy, Leif Walsh, Michael A. Bender, Martin Farach-Colton, Rob Johnson, Bradley C. Kuszmaul, and Donald E. Porter. 2015. BetrFS: A Right-Optimized Write-Optimized File System. In Proceedings of the 13th USENIX Conference on File and Storage Technologies, FAST 2015, Santa Clara, CA, USA, February 16-19, 2015, Jiri Schindler and Erez Zadok (Eds.). USENIX Association, 301–315. https://www.usenix.org/conference/fast15/technical-sessions/presentation/jannen

[59] William Jannen, Jun Yuan, Yang Zhan, Amogh Akshintala, John Esmet, Yizheng Jiao, Ankur Mittal, Prashant Pandey, Phaneendra Reddy, Leif Walsh, Michael A. Bender, Martin Farach-Colton, Rob Johnson, Bradley C. Kuszmaul, and Donald E. Porter. 2015. BetrFS: Write-Optimization in a Kernel File System. ACM Trans. Storage 11, 4 (2015), 18:1–18:29. https://doi.org/10.1145/2798729

[60] Eric R Knorr, Baptiste Lemaire, Andrew Lim, Siqiang Luo, Huanchen Zhang, Stratos Idreos, and Michael Mitzenmacher. 2022. Proteus: A self-designing range filter. In Proceedings of the 2022 International Conference on Management of Data. 1670–1684.

[61] Tim Kraska, Alex Beutel, Ed H Chi, Jeffrey Dean, and Neoklis Polyzotis. 2018. The Case for Learned Index Structures. SIGMOD (2018).

[62] Harald Lang, Thomas Neumann, Alfons Kemper, and Peter Boncz. 2019. Performance-Optimal Filtering: Bloom Overtakes Cuckoo at High Throughput. In VLDB.

[63] David J. Lee, Samuel McCauley, Shikha Singh, and Max Stein. 2021. Telescoping Filter: A Practical Adaptive Filter. 204 (2021), 60:1–60:18. https://doi.org/10.4230/LIPIcs.ESA.2021.60

[64] Meng Li, Deyi Chen, Haipeng Dai, Rongbiao Xie, Siqiang Luo, Rong Gu, Tong Yang, and Guihai Chen. 2022. Seesaw Counting Filter: An Efficient Guardian for Vulnerable Negative Keys During Dynamic Filtering. In Proceedings of the ACM Web Conference 2022 (Virtual Event, Lyon, France) (WWW ’22). Association for Computing Machinery, New York, NY, USA, 2759–2767. https://doi.org/10.1145/3485447.3511996

[65] Lailong Luo, Deke Guo, Ori Rottenstreich, Richard TB Ma, Xueshan Luo, and Bangbang Ren. 2019. The Consistent Cuckoo Filter. In INFOCOM.

[66] Siqiang Luo, Subarna Chatterjee, Rafael Ketsetsidis, Niv Dayan, Wilson Qin, and Stratos Idreos. 2020. Rosetta: A robust space-time optimized range filter for key-value stores. In Proceedings of the 2020 ACM SIGMOD International Conference on Management of Data. 2071–2086.

[67] Hunter McCoy, Steven Hofmeyr, Katherine Yelick, and Prashant Pandey. 2023. High-performance filters for gpus. In Proceedings of the 28th ACM SIGPLAN Annual Symposium on Principles and Practice of Parallel Programming. 160–173.

[68] Michael Mitzenmacher, Salvatore Pontarelli, and Pedro Reviriego. 2020. Adaptive Cuckoo Filters. ACM J. Exp. Algorithmics 25 (2020), 1–20. https://doi.org/10.1145/3339504

[69] Bernhard Mößner, Christian Riegger, Arthur Bernhardt, and Ilia Petrov. 2023. bloomRF: On performing range-queries in Bloom-Filters with piecewise-monotone hash functions and prefix hashing. In Advances in database technology: Proceedings of the 26th International Conference on Extending database Technology (EDBT), Vol. 26. 131–143.

[70] Patrick E. O’Neil, Edward Cheng, Dieter Gawlick, and Elizabeth J. O’Neil. 1996. The Log-Structured Merge-Tree (LSM-Tree). Acta Informatica (1996).

[71] Anna Pagh, Rasmus Pagh, and S Srinivasa Rao. 2005. An optimal Bloom filter replacement. In Proceedings of the Sixteenth Annual ACM-SIAM Symposium on Discrete Algorithms (SODA). 823–829.

[72] Rasmus Pagh, Gil Segev, and Udi Wieder. 2013. How to Approximate a Set Without Knowing its Size in Advance. In FOCS.

[73] Prashant Pandey, Fatemeh Almodaresi, Michael A Bender, Michael Ferdman, Rob Johnson, and Rob Patro. 2018. Mantis: A fast, small, and exact large-scale sequence-search index. Cell systems 7, 2 (2018), 201–207.

[74] Prashant Pandey, Michael A Bender, Rob Johnson, and Rob Patro. 2017. deBGR: an efficient and near-exact representation of the weighted de Bruijn graph. Bioinformatics 33, 14 (2017), i133–i141.

[75] Prashant Pandey, Michael A Bender, Rob Johnson, and Rob Patro. 2017. A general-purpose counting filter: Making every bit count. In Proceedings of the 2017 ACM International Conference on Management of Data. 775–787.

[76] Prashant Pandey, Michael A Bender, Rob Johnson, and Rob Patro. 2017. Squeakr: an exact and approximate k-mer counting system. Bioinformatics 34, 4 (2017), 568–575.

[77] Prashant Pandey, Alex Conway, Joe Durie, Michael A. Bender, Martin Farach-Colton, and Rob Johnson. 2021. Vector Quotient Filters: Overcoming the Time/Space Trade-Off in Filter Design. In Proceedings of the 2021 International Conference on Management of Data. ACM, 1386–1399. https://doi.org/10.1145/3448016.3452841

[78] Jason Pell, Arend Hintze, Rosangela Canino-Koning, Adina Howe, James M Tiedje, and C Titus Brown. 2012. Scaling metagenome sequence assembly with probabilistic de Bruijn graphs. Proceedings of the National Academy of Sciences 109, 33 (2012), 13272–13277.

[79] Jack Rae, Sergey Bartunov, and Timothy Lillicrap. 2019. Meta-learning neural bloom filters. In International Conference on Machine Learning.

[80] Pandian Raju, Rohan Kadekodi, Vijay Chidambaram, and Ittai Abraham. 2017. PebblesDB: Building key-value stores using fragmented log-structured merge trees. In Proceedings of the 26th Symposium on Operating Systems Principles. 497–514.

[81] Brandon Reagen, Udit Gupta, Robert Adolf, Michael M Mitzenmacher, Alexander M Rush, Gu-Yeon Wei, and David Brooks. 2017. Weightless: Lossy weight encoding for deep neural network compression. arXiv preprint arXiv:1711.04686 (2017).

[82] Kai Ren, Qing Zheng, Joy Arulraj, and Garth Gibson. 2017. SlimDB: A Space-Efficient Key-Value Storage Engine For Semi-Sorted Data. PVLDB (2017).

[83] Pedro Reviriego, Alfonso Sánchez-Macián, Stefan Walzer, and Peter C. Dillinger. 2021. Approximate Membership Query Filters with a False Positive Free Set.

[84] Kamil Salikhov, Gustavo Sacomoto, and Gregory Kucherov. 2013. Using cascading Bloom filters to improve the memory usage for de Brujin graphs. In Algorithms in Bioinformatics. Springer, 364–376.

[85] Subhadeep Sarkar, Niv Dayan, and Manos Athanassoulis. 2023. The LSM Design Space and its Read Optimizations. In ICDE.

[86] Securelist.com. 2022. . https://securelist.com/kaspersky-security-bulletin-2021-statistics/105205/

[87] Dimitrios Skarlatos, Apostolos Kokolis, Tianyin Xu, and Josep Torrellas. 2020. Elastic Cuckoo Page Tables: Rethinking Virtual Memory Translation for Parallelism. In ASPLOS.

[88] Brad Solomon and Carl Kingsford. 2016. Fast search of thousands of short-read sequencing experiments. Nature biotechnology 34, 3 (2016), 300.

[89] Henrik Stranneheim, Max Käller, Tobias Allander, Björn Andersson, Lars Arvestad, and Joakim Lundeberg. 2010. Classification of DNA sequences using Bloom filters. Bioinformatics 26, 13 (2010), 1595–1600.

[90] Bo Sun, Mitsuaki Akiyama, Takeshi Yagi, Mitsuhiro Hatada, and Tatsuya Mori. 2016. Automating URL blacklist generation with similarity search approach. IEICE TRANSACTIONS on Information and Systems 99, 4 (2016), 873–882.

[91] Mahesh V. Tripunitara and Bogdan Carbunar. 2009. Efficient Access Enforcement in Distributed Role-Based Access Control (RBAC) Deployments. In Proceedings of the 14th ACM Symposium on Access Control Models and Technologies (Stresa, Italy) (SACMAT ’09). Association for Computing Machinery, New York, NY, USA, 155–164. https://doi.org/10.1145/1542207.1542232

[92] Kapil Vaidya, Subarna Chatterjee, Eric Knorr, Michael Mitzenmacher, Stratos Idreos, and Tim Kraska. 2022. SNARF: a learning-enhanced range filter. Proceedings of the VLDB Endowment 15, 8 (2022), 1632–1644.

[93] Hengrui Wang, Tw Guo, Junzhao Yang, and Zhang Huanchen. 2024. GRF: A Global Range Filter for LSM-Trees with Shape Encoding. In SIGMOD.

[94] Peng Wang, Guangyu Sun, Song Jiang, Jian Ouyang, Shiding Lin, Chen Zhang, and Jason Cong. 2014. An efficient design and implementation of LSM-tree based key-value store on open-channel SSD. In Proceedings of the 9th European Conference on Computer Systems (EuroSys). 16:1–16:14.

[95] Ziwei Wang, Zheng Zhong, Jiarui Guo, Yuhan Wu, Haoyu Li, Tong Yang, Yaofeng Tu, Huanchen Zhang, and Bin Cui. 2023. Rencoder: A space-time efficient range filter with local encoder. In 2023 IEEE 39th International Conference on Data Engineering (ICDE). IEEE, 2036–2049.

[96] Richard Wen, Hunter McCoy, David Tench, Guido Tagliavini, Michael Bender, Alex Conway, Martin Farach-Colton, Rob Johnson, and Prashant Pandey. 2025. Adaptive Quotient Filters. In Proceedings of the 2025 International Conference on Management of Data. ACM.

[97] Yuhan Wu, Jintao He, Shen Yan, Jianyu Wu, Tong Yang, Olivier Ruas, Gong Zhang, and Bin Cui. 2021. Elastic Bloom Filter: Deletable and Expandable Filter Using Elastic Fingerprints. IEEE Trans Comput (2021).

[98] Kun Xie, Yinghua Min, Dafang Zhang, Jigang Wen, and Gaogang Xie. 2007. A Scalable Bloom Filter for Membership Queries. In GLOBECOM.

[99] Minghao Xie, Quan Chen, Tao Wang, Feng Wang, Yongchao Tao, and Lianglun Cheng. 2022. Towards Capacity-Adjustable and Scalable Quotient Filter Design for Packet Classification in Software-Defined Networks. IEEE Open Journal of the Computer Society (2022).

[100] Jun Yuan, Yang Zhan, William Jannen, Prashant Pandey, Amogh Akshintala, Kanchan Chandnani, Pooja Deo, Zardosht Kasheff, Leif Walsh, Michael A. Bender, Martin Farach-Colton, Rob Johnson, Bradley C. Kuszmaul, and Donald E. Porter. 2016. Optimizing Every Operation in a Write-optimized File System. In 14th USENIX Conference on File and Storage Technologies, FAST 2016, Santa Clara, CA, USA, February 22-25, 2016, Angela Demke Brown and Florentina I. Popovici (Eds.). USENIX Association, 1–14. https://www.usenix.org/conference/fast16/technical-sessions/presentation/yuan

[101] Jun Yuan, Yang Zhan, William Jannen, Prashant Pandey, Amogh Akshintala, Kanchan Chandnani, Pooja Deo, Zardosht Kasheff, Leif Walsh, Michael A. Bender, Martin Farach-Colton, Rob Johnson, Bradley C. Kuszmaul, and Donald E. Porter. 2017. Writes Wrought Right, and Other Adventures in File System Optimization. ACM Trans. Storage 13, 1 (2017), 3:1–3:26. https://doi.org/10.1145/3032969

[102] Huanchen Zhang, Hyeontaek Lim, Viktor Leis, David G Andersen, Michael Kaminsky, Kimberly Keeton, and Andrew Pavlo. 2018. Surf: Practical range query filtering with fast succinct tries. In Proceedings of the 2018 International Conference on Management of Data. 323–336.

[103] Huanchen Zhang, Hyeontaek Lim, Viktor Leis, David G Andersen, Michael Kaminsky, Kimberly Keeton, and Andrew Pavlo. 2020. Succinct range filters. ACM Transactions on Database Systems (TODS) 45, 2 (2020), 1–31.

[104] Benjamin Zhu, Kai Li, and R Hugo Patterson. 2008. Avoiding the Disk Bottleneck in the Data Domain Deduplication File System. In Proceedings of the 6th USENIX Conference on File and Storage Technologies (FAST). 1–14.

## 原文作者相关
Prashant Pandey。Prashant Pandey是犹他大学Kahlert计算学院的助理教授。此前，他在加州大学伯克利分校和卡内基梅隆大学做博士后研究。他于2018年12月在石溪大学获得计算机科学博士学位。他在推进数据结构的理论和实践方面做了大量工作，并利用它们构建高性能和大规模的数据库和文件系统、计算生物学工具以及用于网络安全的异常检测工具。他获得了2024年的NSF职业奖、2023年的IEEE CS TCHPC早期职业研究人员奖、2018年的Catacosinos奖学金以及2016年FAST会议的最佳论文奖。Prashant Pandey是当前提案的作者之一，他在2023年SPAA（算法与架构并行性研讨会）上组织了一个关于过滤器数据结构的研讨会。所有演讲都已录制并可在此处观看。研讨会的主要目标是汇集数据结构研究前沿的研究人员，并帮助揭示开放的研究问题。

Martín Farach-Colton。Martín Farach-Colton是纽约大学坦顿工程学院计算机科学与工程系的Lenoard J. Shustek讲席教授兼系主任。他是ACM、IEEE和SIAM的会士，也是阿根廷国家科学院的成员。他的研究兴趣涵盖存储系统中数据结构的理论和实践。他的论文在FAST和ASPLOS会议上获得了最佳论文奖。他创办了Tocketek，一家性能数据库公司，该公司于2015年被收购。

Niv Dayan。Niv Dayan是多伦多大学计算机科学系的助理教授。他对存储应用的数据结构有广泛兴趣。他曾在哈佛大学和哥本哈根大学做博士后研究，并在哥本哈根IT大学获得博士学位。他是2017年“Best of SIGMOD”奖的获得者。

Huanchen Zhang。Huanchen Zhang是清华大学IIIS（姚班）的助理教授。他的研究兴趣是数据库管理系统，特别是索引、数据压缩和云数据库。他在卡内基梅隆大学计算机科学系获得博士学位。在加入清华大学之前，他在Snowflake担任博士后研究员。他是2021年SIGMOD Jim Gray论文奖和2018年SIGMOD最佳论文奖的获得者。
