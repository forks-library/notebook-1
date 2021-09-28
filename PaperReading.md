# Paper Reading

## Fuzzing error-handling code using context-sensitive software fault injection

USENIX2020 [PDF](https://www-users.cs.umn.edu/~kjlu/papers/fifuzz.pdf)

表达：
augment 说自己的fuzzer能有效补充增强现有的fuzzing技术
and we are still waiting for the response of remaning ones 提交了bug等待回复
This manual study is required for gaining the insights into building the automated static analysis 手工分析是有必要的

错误处理的代码一些只能在偶然情况下触发 如内存不够 网络连接失败
这篇文章的核心是SFI 上下文敏感的软件错误注入
发现了317个alert 根据根本原因分成50个unique bugs

error site 注入错误的位置 比如malloc
现有的技术没考虑上下文（调用栈）注入错误那就一定会错 这样会错过一些bug

方法：
	1. 静态分析找error sites 运行时有可能出错的地方 只找库函数避免重复；找满足条件的函数调用：返回的是指针或整数、返回值会和NULL或者0比较，定义一个比较次数/总次数的阈值R=0.6
	2. 运行被测程序 收集到达每个error site能有哪些calling context，以及当前的coverage
	3. 创建错误序列 错误序列的每一个不是error site 而是error point <ErrorLoc, CallCtx> 其中CallCtx 是<CallLoc, FuncLoc>的数组 具体用哈希表存储
	4. 变异错误序列 
	5. 运行
	6. 看看是否有新的coverage 循环



同时变异错误序列和程序输入

贡献：
	* 做了两个手工分析：42%的错误处理代码都是处理偶然错误，其中很少能被现有的技术触发
	* 基于上下文敏感的SFI的fuzzing基础 动态注入错误，考虑了调用关系
	* FIFUZZ 第一个系统性考虑了不同调用关系的测试错误处理代码的fuzzing框架
	* 测了9个C语言程序 发现了50个bug 和AFL AFLFast AFLSmart FairFuzz比较能发现很多别人发现不了的bug



虽然错误偶然触发，但在adversarial setting下却可以稳定触发 如攻击者耗尽内存能可靠地让malloc返回NULL ；后面也提到攻击窗口太小的话 不能成功

分析的软件：vim bision ffmpeg nasm catdoc clamav cflow gif2png+libpng openssl 每个选100个源代码看发现：
42%偶然错误，70%检查返回值是不是表示错误

看CVE发现fuzzing发现的CVE中31%和错误处理有关 但只有9%是偶然错误

一次变异一个 如果没发现新的coverage就抛弃

ASan引入了太大的runtime overhead 实验分两组 用asan和不用


选vim一个程序来显示24小时内 有用的错误序列、程序输入分别随时间增长的曲线


实验结果的表格：分asan和不用asan，产生了多少错误序列和input 多少有用，发现的bug的分类 三类：返回NULL，malloc失败，assert


附录给50个随机选的alerts 程序、触发序列Error points、崩的源代码行号、错误类别、反馈修复状态

发现的46个和错误处理相关的bug中 只有4个需要一个以上的bug注入 也就说大部分bug一个错误就能触发了；大部分bug都是因为被调用的函数正确处理了异常 但调用者没有——开发者经常由于复杂的调用在error propagation上犯错误

和其他fuzzer比较 说自己的coverage更高

自己没比人家强就说We believe that if我们也实现他们的fuzzing过程，那就会更好

discussion:
错误位置的提取 的 误报： 有些函数虽然经常跟0比较但不会返回错误 如strstr 通过分析定义和调用图检查是否真的能返回表示错误的错误值；函数的输入一定先转换成有效的数据 用符号执行来分析 对每个函数调用计算约束

错误检测 的 漏报：为了避免重复注入只考虑了库函数，但有些开发者自己写的没调用任何库函数也能返回错误；coverage没到 FIFUZZ不能提供所有可能的程序输入和配置；用ASan的局限 但可以扩展用MSan UBSan TSan

相关的paper:
轻量级运行时监测：基于硬件的tracing [3, 31]， call-path inferring[42]
用静态分析找错误处理的bug [28, 32, 33, 37, 53]
PairCheck[9] 统计分析找资源acquire和release的pair，找错误处理中没有release

最后conclusion最后一段说改进 减少误报；提高性能；其他编程语言 plan to test the program in other programming languages （such as C++ and Java）

## PANGOLIN: Incremental Hybrid Fuzzing with Polyhedral Path Abstraction

SP2020 [PDF](https://qingkaishi.github.io/public_pdfs/SP2020.pdf)

一句话概括：改进QSYM混合fuzzing 缓存求解出来的取值范围用多边形表示 来更好的变异更快的求解

单词与表达：

- sluggish 性能不行 太慢
- succinct 简洁的
- 取得平衡 a sweet spot between ...
- obstruct 阻碍
- we followed the standard instructions in the previous paper 说自己的方法是按照别人建议的
- orthogonal 说别人的研究和自己的不冲突可以互补

intro 第一段 介绍hybrid fuzzing很有用；第二段说现在的方法不行 提出问题不incremental 最后一句提However, intuitively应该可以这么做；之后举例子；核心的两点——用多边形的路径约束把种子变异转换成在一个多边形内取样、用约束降低约束求解复杂度减小可行解空间 从而加速约束求解；然后说自己测了一下 效果比state of art好10%~30%的coverage，在LAVA-M能多发现500+bugs，发现新的41个bug 拿到8个CVE

随机变异的空间总是太大了

约束的抽象 有 interval [32], octagon [33], and polyhedral [34]. 都是sound的（包含所有目标点），但polyhedral有最好的precision 包含最少的非目标点

采样用Dikin walk algorithm 能保证均匀采样 保证多样性：每两个采样之间的距离都要大于一个动态边界

具体做法并不盲目计算x+y和x-y，而是加入约束中的线性表达式 如5x+y

应该采多少样本？越难cover的需要越多，越多依赖它的路径需要越多


evaluation 三个RQ研究问题：1. 比现在最好的fuzzer能不能发现更多的bug；2.能不能更高的coverage；3. 变异方法的有效性

实验用了两个afl，angora有-j 2，但T-Fuzz不支持 也就直接跑了

选seed的方式：用AFL提供的seed，用人家代码里自带的测试样例

重复10次

比较了效果好之后讨论为啥没能发现LAVA-M的所有bug？说这是QSYM的问题 缺少对底层系统调用的建模如who用的x2nrealloc，不支持浮点数约束。但是这些限制不是这篇work要解决的问题，and we leave them as our future work.

crash去重用的`afl-cmin -C`

发现了T-Fuzz对大程序有scalability issues 所以就没有比较coverage

比QSYM好，原因是qsym对每个branch只生成一个seed，这样会漏掉一些——覆盖到一个branch并不意味着就能触发漏洞

The performance of constrained mutation：比较提出的constrained mutation和SMTSampler 看给定time buget分别3s 5s 10s的情况下能解出多少 计算平均时间的时候如果不能解决就按上限——超过95%的约束都能在3秒钟内搞定。总之，我们的方法比SMTSampler效率和有效性都强
为啥呢？减少了约束求解器的调用、保证了均匀分布

related work 种子的优先选择和调度 是基于程序结构的 忽视了输入的实际取值空间
相比于KLEE缓存了constraint，我们缓存了原始路径约束的简化形式 path abstraction

如何有效的将concolic execution和fuzzing结合is always under consideration

尽管有内存快照的技术来保存程序状态[7] "Unleashing mayhem on binary code"，但对hybrid fuzzing来说太慢了

## Full speed Fuzzing: Reducing Fuzzing Overhead

一句话概括： 每个基本块第一个字节改成CC触发中断，不需要耗时地记录coverage了，发现新的基本块就能自动知道

单词：

- tracking apparatuse插桩代码做的事情
- niche 合宜的小环境 这里似乎是受限的意思 As applications of directed fuzzing are generally niche, such as taint tracking [16] or patch testing [31], coverage-guided fuzzing’s wider scope makes it more popular among the fuzzing community [5], [6], [4], [3].
- narrow 区别不大 results in Section VI suggest that the performance gap is much narrower.
- convincingly 比其他人好 A12 excceds 0.71
- performance-taxing 耗时
- per-variant geometric mean分组计算均值
- deficit 引入的overhead: ... far outweighs the performance deficit from trimming and calibration tracing
- is not a technical challenge 没做的东西claim不难做
- graciously 感谢其他人

人家的idea：

```
把程序变成一个能自己报告有没有发现新的basic block的——在每个基本块开头变成0xCC（encode the current frontier），触发了中断说明有新的覆盖率，需要停下fork server去掉这个中断再继续跑
做coverage-guided tracing
人家的实验其实不是在做fuzzing，而是先跑afl-qemu收集24小时所有生成的seed（5个不同的test case datasets），再改了afl只跑run_target，看不同的设计下耗时的区别
去掉噪音用的trimmed-mean denoising 去掉最大最小33%
实验是5个dataset,每个重复8次 
基于afl-dyninst实现，但是这玩意性能不行 所以跑的oracle程序fork server还是用afl-as插的
在tracer记录块的覆盖率的时候，一个块可能执行多次很大的overhead，于是搞了个全局的hashmap，和fork server共享 之后的进程就知道哪些才是unique-covered basic blocks
不懂的地方：为啥会尝试对同一个地方多次unmodify?  We observe that even coverage-increasing test cases often have significant overlaps in coverage. This causes UnTracer to attempt unmodifying many already-unmodified basic blocks, resulting in high overhead.
选择的fuzz文件类型：dev开发，图片，压缩data archiving，网络network utilities，音频，文档，密码学cryptography，web开发
timeout也是一个很重要的因素 如果timeout的文件太多 作者的优势就不明显了；实验设置为500ms的超时
比较的baseline: 只fork server不进行任何插桩，这是最快的 overhead是相对于这个baseline而言的
```

后续要了解的：

```
Intel PT硬件辅助[11],[4],[12]的覆盖率 overhead更小，缺点：需要一个支持的CPU，解码CFG日志耗时，只支持x86
Xuwen的优化操作系统的系统调用[61] fuzzer-agnostic operating primitives
程序改写AFL-lafIntel [70] unrolls magic bytes into single comparisons at compile-time, but currently only supports white-box binaries.
```

三种覆盖率的计算方式：基本块，边，basic block path这一种没人做
只记录basic block来推断边的信息，是有问题的：存在critical edges就不准确，需要先去掉critical edges才行 就是空的else也要当成一个块
否则会错误地丢掉一些发现了新覆盖率的种子 erroneously discard coverage-increasing inputs.（可能afl对for的支持就是这个bug


afl的queue里的文件有+cov的才是发现了新的覆盖率的

影响afl变异优先策略的除了coverage还有文件大小，相同覆盖率优先用小的seed

p-value小于0.05要说(pair-wise)

hybrid的fuzzer花更多的时间变异 如QSYM


## Effective Program Debloating via Reinforcement Learning

CCS18 [PDF](https://www.cis.upenn.edu/~mhnaik/papers/ccs18.pdf)

概括：把机器学习用到Program Debloating，基本方法是反复切片去掉，用Reinforcement Learning来减少编译测试的次数

单词

- seldom if ever used by average users 一般用户不会用
- has led to its sparing use 导致没人用
- has been shown to suffice in the literature on ... 论文中已经提到
- mangle 搞乱程序
- sacrifice efficiency 说别人的不足的时候说牺牲了xxx
- tailored to C/C++ 只适合xxx
- myopic 短视的 Since the rules are myopic, C-Reduce generates a significant number of syntactically invalid candidates
- albeit 但是 albeit due to a different reason.
- presuming a general setting where such an analysis may not be available 举的例子可以很简单静态分析出来，说一般的情况下静态分析没用
- akin to ... 和xxx相同 插入语 because it not only avoids syntactic errors, akin to Perses, but it also learns to avoid semantic errors.
- Overall 一段话结束的时候总结
- large boilerplate code 一大段代码
- heed to 遵守 C-Reduce does not heed to common software engineering practices such as modularity and locality
- suffices in practice实际上是否足够 The reader may wonder whether a naive approach to program reduction based on runtime code coverage suffices in practice.
- empirically comfirmed 验证鲁棒性只能经验性地验证
- in this regard 在这个方面 We can mitigate the issue by combining the results of multiple static analyzers that possess different capabilities in this regard.

idea

```
程序的库和one-size-fits-all的开发方式导致了大量很少用/没用的代码
前人的做法没考虑语义依赖 导致未初始化变量等语义错误 unaware of semantic dependencies between program elements (e.g., def-use relations of variables
debloating 通过delta debugging 一步步去掉程序中能删的片段，用强化学习加速（决策树 马尔科夫决策过程），保证能过测试high-level specification，最后得到的二进制任何一个片段都不能再删（1-minimality）
这个方法能扩展到大程序，避免现有工具超时的问题
这个方法还可以降低攻击面 去掉可选功能中的漏洞，减少ROP gadget
```

想做到五点：
最小；耗时短；鲁棒不引入新的漏洞；生成的代码可维护可扩展；通用
比静态分析动态coverage的都小；其他人超时我们都行；用静态分析工具和AFL测3天可以反馈loop；保留了modularity and locality；方法通用 和编程语言、规范无关

规范就是测试代码 输入一个裁剪后的代码 输出能否满足：
能过编译；需要的功能正常；不需要的功能至少不能崩（测试样例来自回归测试）；要使用sanitizer确保没有未定义行为

马尔科夫决策过程 MDP



基于模型的强化学习 MBRL
在模型的帮助下解决MDP问题，学出转换的概率和奖赏，得到可以最大化收益和的动作

方法：执行过程会更新决策树
程序的编码就是程序n份 每一份是否存在 一开始就是n个1



具体实现的时候是先删global-level的组件 如全局变量声明、类别定义、函数定义，再删局部类别的组件 赋值、if语句、while语句，然后继续global local ...迭代到稳定
使用简单的依赖分析来拒绝不可能的程序 如没有main 缺失变量定义 变量没初始化 没有return语句
决策树用的FastDT，用的exact decision trees没有boosting bagging 
计算代码行数之前要先宏展开

Evaluation
选了10个binutils里的程序，都在busybox里 可以直接比较
避免产生死循环的程序 设置了不同的timeout 0.01~1s


画图差异太大可以截断 顶部显示数值
有些CVE没能删掉 说这些在核心功能里面 不能简单删掉 如条件竞争 
产生的代码使静态分析工具的报告减少了95.4% 报告就能看了 The decreased size and complexity of the reduced programs also enable to apply more precise program checkers such as static analyzers.
然后人工检查说都是误报
feedback process 跑AFL的时候崩了 发现是罕见的情况 把这种测试用例考虑进去后重新生成 再fuzzing就3天都不崩了This simple feedback process effectively improved the robustness of the resulting program

THREATS TO VALIDITY
测试脚本行为的不确定性：用sanitizer缓解，依赖的其他库代码需要也用同样的编译；timeout机制引入的误报，可以改变检测死循环的机制[20] 不依赖timeout
测试样例不完整may not be exhaustive enough 使用基于语法的fuzzer
静态分析工具的unsoundness 不支持复杂的特性 如复杂的指针算术计算，未知语义的API调用导致的复杂控制流——结合多种静态分析工具

更多方向

```
program reasoning 想知道是否引入了新的bug，包括静态分析 动态分析 fuzzing 运行时监控 验证
静态分析工具：Sparrow [13]—a static analyzer for finding security bugs 可以检测bug，还能移除不可达代码
程序debloating 粗粒度的Docker大容器拆解成多个小容器 需要动态分析应用行为[35]
更细粒度的有做Java [28]和Android [27]应用的 ；有避免载入不会用到的函数的[34] （函数级别的依赖分析）
另一个正交的方向：检测和减少运行时内存膨胀 [9,33,40-43]
输入样例的最小化 场景是测试编译器/解释器，产生能crash的更小的程序 不考虑安全性可读性
有工作考虑了语法的正确性[23,32,37] 但没考虑语义正确性
Program slicing 指定一个位置提取程序的一小部分 需要指定语义和依赖关系（challenging），也可能不能去掉漏洞 （这篇文章的方法更好）
静态可达性分析 不能处理复杂的控制流如间接调用、复杂条件和指针算术
动态可达性分析 这篇文章的方法更好
```

future work: 
more expressive probabilistic models with efficient incremental learning, 
designing various forms of specification other than input-output examples,
applying to debloat programs written in arbitrary languages such as binary.

## GREYONE: Data Flow Sensitive Fuzzing

USENIX20 [PDF](https://www.usenix.org/system/files/sec20spring_gan_prepub.pdf)

单词:

- is labor-intensive and requires lots of manual efforts 需要人工
- Head-to-Head Comparison 比较不同的work
- to draw conclusions as objective as possible 尽可能客观
- 选被测程序的原因 We chose target applications considering several factors, including popularity, frequency of being tested, development activeness, and functionality diversity.
- 选出来的类别包含 graphics processing libraries (e.g., libcaca and libsixel),
- shipping with 用afl-cmin的时候说the tool afl-cmin shipping with AFL
- 确定fuzzing的时间 we test target applications for more time, until the fuzzers reach a relatively stable state (i.e., the order of fuzzers’ performance does not change anymore).
- Experiments showed that the fuzzers will get stable after testing these applications for 60 hours. So, we tested each application for 60 hours in our experiment.
- 除了给平均结果 也给出maxinum number
- a steady and stronger growth trend
- 大部分方面都更快 in a faster pace than QSYM in most subjects

idea:

使用轻量级fuzzing-driven taint inference FTI
有了taint之后用输入优先模型判断先探索哪个路径 变异哪些字节 怎么变异
强调dataflow features： constraint conformance 变量的值与预期值的距离
实验在最新的软件上跑 发现了105个新的bugs，拿到了41个CVE

RQ1: 怎么做轻量准确的污点分析 FTI变异输入每个字节看变量值的变化 轻量 没有over-taint 缓解under-taint(变异不完整会有) 运行时非常快
RQ2: 有了taint怎么有效指导变异 选择啥路径 选择哪些字节 优先选择能影响更多未到达分支的字节，优先依赖更多优先字节的分支
RQ3: 如何使用数据流特征(污点属性和约束符合性)优化进化方向conformance-guided evolution 优先更高符合性的seeds 能避免Angora的陷入局部最优的问题，另外将正在执行的变异rebase到更好的seeds可以显著优化速度

符号执行不能处理大程序，不能解决复杂的约束如on-way functions
传统的动态污点分析 需要大量劳动人类工作, 不准确 undertaint的问题 不能处理隐式数据流，非常慢

不符合non-intererence rule[16]的就有依赖关系
FTI变异的时候是使用预先定义的变异规则 如单位翻转、多位翻转和算术操作，不能保证完全变异
变量值追踪是插代码 special value tracking code 只关注与路径约束有关的变量，同时也能提出值和对应的输入字节比较 一致就是direct copy

输入字节的权重是它影响的未到达的分支数量，一个未到达分支的权重是它依赖的所有字节权重的和




变异哪个位置？找当前seed未覆盖的附近的分支 按分支权重倒序；对于一个分支依赖的多个字节，按字节的权重倒序
怎么变异？direct copy如果需要跟一个运行时算出的checksum相等 先得到运行时需要的值，再使用这个值和微小变异来修复对应的输入字节 可以直接知道应该修改哪个offset 需要什么值
indirect copies就随机变异 可以一次变异多个字节 
缓解undertaint 除了依赖的字节 也以一个较低的概率变异邻居字节

计算符合性
对于未到达的分支 定义是两个比较的值之间有多少bit一致，
对于一个已经到达基本块 定义是其所有未到达邻居的符合性的最大值，
对于一个seed 定义是覆盖的所有基本块的符合性的累加

seed queue新的结构 在相同coverage 发现更高分数的seed的时候 替换；在发现相同coverage 相同分数 但基本块conformance的分布不同的时候，加入的这个Node数组

这种方式可以与梯度下降相当，还能避免Angora的陷入局部最优的问题
在替换seed后 后续对这个seed的使用也立刻换成新的seed 这样可以在LAVA数据集上快3倍

选seed的时候 给高分数的更高权重

selective testing 相比AFL的coverage tracking多了两个模式：FTI变量值监测，进化优化算conformance分数
当conformance一段时间不增加了就切回AFL的模式 提高运行速度

conformance计算用的by operations like __builtin_popcount
FTI用到的值监测 用的bitmap的结果 给每个变量一个全局唯一的ID

LAVA-M除了who也有unlisted bugs?

更深入的分析：
FTI的under taint问题
FTI有没有用？ 实现一个基于DTA的镜像比较
各个不同部分对结果的贡献 去掉优先策略，去掉conformance进化 三者比较
selective fuzzing 比较执行速度

其他Paper:

```
DigFuzz[45] 用了符号执行 概率路径优先模型
TaintInduce[46]可以自动推测出propagation rules
ProFuzzer[42]也是一次变异一个字节 但只关注coverage变化 不能推断出污点的依赖关系 和FairFuzz[24]都能推断partial type of mutated bytes 但对这个分支是否已经到过是insensitive的
MutaFlow[26] 监测sink APIs的变化 参数是否被tainted
REDQUEEN[4] 能处理direct copy但不能找到精确的位置 需要百次运行来获得a colorized version with higher entropy 再测一遍获得位置 整个过程太慢
Intel-laf[1] 将长的比较分割成短的比较 但是带来更多语义相同的路径 不能处理非常量比较
SYMFUZZ[8] 可以检测input bits之间的依赖 计算出optimal mutation ratio
```


-----

## VUzzer的变异操作

https://github.com/vusec/vuzzer/blob/f6f7d593a0e76e86afb3c7af6d5186f897bab979/operators.py#L291

这里我们分析一下vuzzer实现的变异操作，其所有operators为一个数组，每个元素都是一个函数

同一个函数可以出现多次，排序后如下所示，我按照我的理解标注了功能：

```
add_random 选择一个随机位置 插入随机字符串
add_random
add_random
change_bytes 根据TAINTMAP修改
change_bytes
change_random 选择一部分字节替换为随机字符串
change_random
change_random_full 选择一部分字节替换为随机字符串或从程序中提取出的magic bytes
change_random_full
double_fuzz 随机选择两种mutator
eliminate_double_null 删去\0\0
eliminate_null 将\0替换为A
eliminate_random 随机删去一部分
eliminate_random
int_slide 将特定位置变为 \xFF\xFF\xFF\xFF 或 \x80\x00\x00\x00 或 \x00\x00\x00\x00
lower_single_random 随机变异1~100次：每次选择一个随机字符变为其-1
raise_single_random 随机变异1~100次：每次选择一个随机字符变为其+1
single_change_random 随机变异1~100次：每次选择一个随机字符变为随机字符
totally_random 生成一个长度在100~1000的字符串
```

----

## Leaky Images: Targeted Privacy Attacks in the Web

[Usenix19](https://www.usenix.org/conference/usenixsecurity19/presentation/staicu) [PDF](https://www.usenix.org/system/files/sec19-staicu.pdf)

攻击者可以知道用户在google,onedrive,dropbox的登录身份，通过特定分享一个图片看能否成功加载

在intro里就给出一个吸引人的例子：用来去匿名化审稿人——收集所有committe的邮箱分享图片，论文中给个攻击者网站的链接

表格1比较相关的攻击：追踪像素、社交媒体指纹（是否登录）、CSRF（有副作用）、本文Leaky images

比较：谁能攻击？ 攻击者能实现啥？ 用途场景

所有网站都能发起攻击，可以准确找出受害者，定向的fine-grained去匿名化

研究了250个最流行网站的30个 找到8个网站有漏洞 手工找 共享的图片能通过一个link加载 且只有特定用户能访问（基于cookie的访问控制）

贡献：

- 新的攻击 定向隐私攻击滥用图片共享服务来确定受害者是否正在访问攻击者的网站
- 讨论了攻击的各个变种 可以攻击aim at单个用户 一群用户 不同服务之间link用户 以及不需要js的
- 展示8个流行网站存在问题 让第三方网站能定位他们的用户
- 提出多种缓解问题的方案并讨论优缺点

Table2 shows a two-dimensional matrix 根据是否鉴权+URL是公开、不同用户相同、不同用户不同来区分，只有需要鉴权+公开url 和 需要鉴权+url对攻击者可知 才能做Leaky image攻击

讨论部分：在方法一节中就讨论related work，比较追踪像素、指纹、定向攻击vs大规模追踪。定向攻击据说对高价值受害者越来越流行[37]

现实measurement的讨论 Our study of ... in real-world sites enables several observations.

- Leaky images是普遍的 prevalent
- 受害者甚至不会注意到被共享了一个图片
- 受害者不能unshare
- 图片共享服务使用了多种实现策略的混合 use a diverse mix of implementation strategies
- 不同网站攻击面不同 varies from site to site

表达：

仅仅需要 involve nothing more than ...

证明现实存在能影响今天最流行网站 Section 4 shows that these cases occur in practice, and that they affect some of today's most popular websites.

为了理解最流行网站受影响的程度 To understand to what extent popular websites are affected by the privacy problem discussed in this paper

第一且最重要的The first and perhaps most important observation is that ...

希望能促进以后更自动化的研究 We hope that our results will spur future work on more automated analyses that identify leaky images

讨论每个解决方案不足

The drawback of this fix is that ...

On the downside, implementing this defense may ...

However, this mitigation cannot defend against ...

require the developers to be aware of the vulnerability in the first place. 事先就知道这个漏洞的存在

和GDPR很搭，要求设计上默认保护 would be in the spirit of the newly adopted European Union's Genearl Data Protection Regulation which requires data protection by design and by default.

需要更多研究来深入分析可用性+兼容性+部署开销，to aid the browser vendors to take an informed decision, future work should perform an in-depth analysis of all these defenses in terms of usability, compatibility and deployment cost, in the style of ... [9], and possibly propose additional solutions.

补充了一种新的攻击Leaky images adds a privacy-related attack to the set of existing targeted attacks.

厂商修复说明问题很重要 This feedback shows that the problem we identified is important to practitioners.

我们的论文帮助提高开发者和研究者的意识以后避免这个问题 Our paper helps raising awareness among developers and researchers to avoid this privacy issue in the future.

技术：

不用onload也可以去检查已经加载的图片的宽高

用<object data="...">嵌套加载 在外面的失败就会加载里面的，这个方案不需要js

图片分享服务能分享给一批用户，把每个受害者编码成bit vector，每个bit关联一个图 encode each victim with a bit vector and to associate each bit with one shared image

这个攻击假设了用户已经在浏览器登录，Skype大部分用户都是通过电脑手机的client，所以这个攻击的影响有限 hence the impact of this attack is limited to the web users

根本原因 js和图片都不受同源策略影响

缓解措施：

服务端：这些方法都要求开发者首先就意识到这种攻击的存在

禁用基于cookie的鉴权，不同用户url不同 不足：链接的私密性可以被comrpmised如不安全的通道，浏览器的侧信道或者直接让用户以不安全的方式处理链接

更严格的cookie鉴权，不同用户url不同 不足：实现可能困难要维护用户和url的映射，而且增加了新的访问控制性能损耗

用CSRF的方式 检查origin头 不足：不能防御信任域的攻击 Facebook之前都允许用户设置html代码

浏览器：

默认不使用cookie加载第三方图片

只有当浏览器确定鉴权与否不影响加载图片的内容 扩展就是默认禁止 用csp扩展来允许双重加载

类似于ShareMeNot[32] 实现默认阻止第三方图片请求 除非用户显式同意

浏览器做information flow control 保证图片是否加载成功的信息不发给服务器 但这个侧信道太多

高级用户：

图片共享服务商应该提供用户更多控制，例如应该决定谁有权限共享给他，以及当前被共享的列表 但大部分用户不关心while the majority of the users are unlikely to take advantage of such fine-grained controls.

