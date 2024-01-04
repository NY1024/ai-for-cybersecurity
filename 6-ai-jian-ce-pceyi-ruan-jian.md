# 6 AI检测PC恶意软件

## 6.1 安全问题

行业和学术界都提出了使用程序分析来检测恶意软件的方法。传统的方法，如基于行为的签名、动态污点跟踪和静态数据流分析，需要专家手动调查未知文件并找出恶意行为。然而，当需要检测大量恶意软件变种时，这些方法的可扩展性并不高。攻击者可以通过使用各种混淆技术，如代码加密、重新排序程序指令和死代码插入技术，专门修改其恶意软件以避免基于分析的恶意软件检测工具的检测。

为了更准确地检测恶意软件并更好地对抗这些检测逃避技术，最新的研究已经利用深度学习算法进行了具有静态和动态分析的恶意软件检测。动态分析识别运行时环境中的程序行为，需要足够的输入来覆盖所有行为。另一方面，静态分析是以程序为中心的，涉及代码反汇编、操作码序列提取、控制流图分析等。

在本章中，我们介绍了一种恶意软件检测方法，该方法将恶意软件转换为图像、序列数据和图，并利用深度学习来学习恶意软件样本之间的相似性以及恶意软件与良性软件之间的不同之处。该方法已由英特尔实验室和微软威胁情报团队演示过 \[69]。

## 6.2 原始数据

本章使用的恶意二进制文件来自VirusShare \[76]。VirusTotal \[77] 用于标记恶意样本并识别恶意软件家族。数据集包含五个家族的恶意软件样本：

* 病毒：病毒修改其他系统程序，因此当受害者的文件被执行时，病毒代码也被执行。
* 蠕虫：蠕虫程序将通过某些媒介（如电子邮件）将自身复制到其他主机。
* 木马：木马假装是良性程序，未经用户许可收集用户信息。
* 勒索软件：勒索软件侵入用户机器并加密其数据以获取加密货币。
* 广告软件：广告软件会向用户推送恶意广告。

良性程序直接从Ubuntu 18.04LTS的软件包中提取 \[73]。已安装软件包的类别各不相同，包括流行的应用程序（如Chrome、Firefox和Zoom）、安全应用程序（如Spybot2、SUPERAntiSpyware和Malwarebytes）、开发者工具（如Python、Github和PuTTY）等。

## 6.3 数据处理

恶意软件分类的目标是通过静态和动态特征，如控制流图和系统API调用，识别软件中的恶意行为。在本章中，使用静态分析来检查恶意软件，因此无需执行二进制文件并监视其行为。为了应用深度学习，第一步是以有意义的方式表示恶意软件数据。本章介绍了三种恶意软件数据表示方式：图像、序列和图。

### 6.3.1 图像数据转换&#x20;

原始二进制文件被读取为字节序列，然后被转换为介于0和255之间的值。然后，将顺序数据重新塑造为两个维度，以便可以直接应用基于计算机视觉的深度学习模型。生成的图像的宽度和高度取决于原始二进制文件的大小。正如Code 6.1所示，图像的高度被计算为二进制文件大小除以宽度。最终的结果四舍五入为整数。图6.1是生成的灰度图像的示例。

<figure><img src=".gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (40).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (41).png" alt=""><figcaption></figcaption></figure>

### 6.3.2 序列数据转换

序列数据的表示更为直接。二进制直接被转换为字节序列，可以解释为范围在0到255之间的无符号整数。由于二进制文件的大小不同，序列被划分为几个具有相同长度的子序列（在Code 6.2中的sequence\_length中）。为确保输入具有相同的大小，最后一个子序列会被填充到sequence\_length的长度。

<figure><img src=".gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure>

### 6.3.3 图数据转换

控制流图（CFG）是一个有向图表示，展示了程序在执行过程中的所有可达路径，如图6.2所示。CFG的节点是程序的基本块。每个基本块是一个连续的、单入口的代码，除了在序列的末尾外，没有任何分支。CFG中的边表示程序中可能的控制流。控制流仅在基本块的开始处进入，并仅在基本块的末尾离开。每个基本块可以有多个入/出边。每个边对应于一个潜在的程序执行。CFG中包含的结构信息表示程序的逐步执行过程，可用于查找无法访问的代码、查找语法结构（如循环）和预测程序的缺陷。

如Code 6.3所示，使用名为Angr的分析二进制文件的Python框架来提取这些数据集中的CFG。它被转换为一个有向图。从可执行文件中提取CFG后，我们将构建一个抽象的有向图 G =< V, E >，包括节点集 V 和边集 E。每个节点 vi 表示CFG中的一个基本块，而有向边 eij 从第一个基本块 vi 指向第二个基本块 vj。每个基本块的特征是使用与第6.3.2小节相同的方法提取的字节序列。

<figure><img src=".gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (45).png" alt=""><figcaption></figcaption></figure>

## 6.4 模型训练

在前一节中，我们用三种不同的方式表示了恶意软件数据。在这一节中，我们分别介绍三种DL模型，用于从基于图像的数据表示、基于序列的数据表示和基于图的数据表示中学习恶意软件特征。

### 6.4.1 模型架构

#### 6.4.4.1 CNN架构

本节利用在计算机视觉中广泛使用的迁移学习思想来训练恶意软件检测模型。迁移学习的基本思想是将从一个领域获得的知识应用到另一个领域，这样我们就不必为每个新模型从零开始重新开始。

本节使用的预训练模型是resnet-50 \[64]。残差网络（ResNets）\[64]是深度卷积网络，它学习卷积层之间的快捷连接，如图6.3所示。其直觉是通过跳过一个或多个层来避免梯度消失的问题。

如Code 6.4所示，使用Tensorflow很容易加载resnet-50网络及其权重。在训练过程中，resnet-50模型的权重将不再更新。在新数据上训练完模型后，建议解冻基础模型的全部或部分层，并以非常低的学习速率再次对整个模型进行微调。

<figure><img src=".gitbook/assets/image (46).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (48).png" alt=""><figcaption></figcaption></figure>

#### 6.4.1.2 RNN模型

如Code 6.5所示，RNN模型有5层：

1. 输入层：神经网络的输入是将二进制文件转换为整数序列生成的数据。
2. 嵌入层：在此层中，原始输入被转换为低维向量。
3. LSTM层：使用双向循环神经网络（BiRNN）基于嵌入层学习更高级别的特征。
4. 汇总层：我们总结整个序列的特征。
5. 输出层：在这一层中，我们使用序列级别的特征来确定分类结果。

<figure><img src=".gitbook/assets/image (49).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (50).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>

#### 6.4.1.3 GNN架构

在本节中，我们使用DGCNN模型\[94]来嵌入图形数据中固有的结构信息。图卷积层采用以下传播规则：

<figure><img src=".gitbook/assets/image (52).png" alt=""><figcaption></figcaption></figure>

图卷积层将节点特征传播到相邻节点以及节点本身，以提取局部子结构信息。如图6.4所示，以及代码片段代码6.6，多个图卷积层被堆叠以获取高层次的子结构特征，然后是一个SortPooling层和一个分类层。

<figure><img src=".gitbook/assets/image (53).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (54).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (55).png" alt=""><figcaption></figcaption></figure>

SortPooling层基于图中的结构角色提取并对顶点特征进行排序\[94]。在这里，提取的顶点特征是连续的Weisfeiler Lehman（WL）颜色。与DGCNN模型一样，我们在使用最后一个输出层的输出对所有顶点进行排序后，将前k个排序的顶点发送到卷积层和分类层。

<figure><img src=".gitbook/assets/image (59).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (57).png" alt=""><figcaption></figcaption></figure>

### 6.4.2 模型性能

良性程序和恶意软件被合并在一起，随机选择了20%的样本作为测试数据集。我们在70%的样本上训练了模型，并在10%的样本上验证了模型。为了减少在有限数据样本上的变异性，我们使用了5折交叉验证来训练评估模型。 所有实验都在Ubuntu 16.04上进行，使用Python 3.7和带有NVIDIA GTX980 Ti图形处理单元（GPU）的TensorFlow 2.3。表6.1显示了所选模型的最终数据集统计信息，表6.2显示了评估结果。选择了五个度量指标进行测试，它们是准确率（Acc）、召回率（Rec）、精确率（Pre）和F1分数（F1）。

## 6.5 模型部署

在模型训练和评估完成后，下一步是部署训练好的模型。关于生产系统设计，读者可以参考第3.7节。请注意，与第3.7节中显示的生产系统架构的主要区别是，这个用例使用静态分析而不是使用Android模拟器。

## 6.6 剩余的问题

传统的恶意软件检测会创建一个唯一的签名，以便利用扫描算法在其他潜在的恶意软件中搜索该签名。恶意软件分析用于创建在基于签名的检测系统中利用的签名。动态基于签名的检测监视程序的状态转换和行为签名，例如服务器到客户端的签名\[25]。静态基于签名的检测分析恶意软件的二进制代码，以识别代码序列作为签名\[70]。混合基于签名的检测使用静态和动态分析来确定恶意性\[55]。为了检测未知的恶意软件，检测系统需要保持其签名数据库的最新状态。这种方法已经被广泛应用于行业中，包括防病毒软件、防火墙、电子邮件和网络网关。基于签名的检测系统在检测已知恶意软件方面非常有效，但通常在检测先前未知的恶意软件方面效果不佳。攻击者可以重新排序恶意软件代码或插入无用的代码以避免这种检测。图6.5显示了由APT威胁行为者OceanLotus分发的一个知名恶意软件家族的代码片段（左侧），以及用于检测它的YARA签名（右侧）\[80]。

<figure><img src=".gitbook/assets/image (60).png" alt=""><figcaption></figcaption></figure>

\
动态分析在虚拟环境中执行程序，以监视其行为并观察其功能。可以使用多种工具安全地执行可疑程序：沙盒（例如Cuckoo、DefenseWall、Bufferzone）；虚拟机（例如HoneyMonkey、VGround）；模拟器（例如TTAnalyze、K-Tracer）。总体而言，它提供了一个模拟环境来运行可疑应用程序。动态分析获取的特征包括API调用、系统调用、注册表更改、内存写入、网络模式等\[18, 57, 72]。尽管动态分析具有潜在的全面性，但它计算成本更高且使用较少。这种分析技术更加耗时，且存在较高的误报率。恶意软件在运行在虚拟机上时通常会进行早期检查，并在发现后立即退出。更糟糕的是，一些恶意软件会故意展示一些良性行为，以引导人工分析人员对恶意软件意图做出错误的结论。

在基于静态分析的恶意软件检测中，在执行可疑程序文件之前，从可执行文件中提取某些静态特征，以确定文件是否恶意。一些工作利用二进制文件本身作为指标来检测恶意软件\[17, 62]。二进制文件的特征，如PE导入特征、元数据和字符串，也广泛应用于恶意软件检测\[19]。其他方法利用逆向工程来理解程序的体系结构。逆向工程用于将程序反汇编以提取高级表示，包括指令流图、控制流图、调用图和操作码序列\[19, 52, 86]。静态分析的一个优势是通常比动态分析快得多。另一个优势是它可以实现更好的覆盖范围。

尽管在这个用例中训练的模型的性能看起来很好，但在实际部署这些模型时应考虑对抗性攻击。例如，代码混淆对于躲避基于签名的检测是有效的，因为它可以显著改变原始恶意软件的语法。同样，混淆的二进制文件也可能能够躲避深度学习模型。代码混淆工具有两个主要目的：(a) 保护知识产权；(b) 躲避恶意软件检测系统。有各种代码混淆技术。代码混淆的基本要求是必须保留程序的语义。传统上，攻击者使用混淆技巧，包括插入无效代码、寄存器重新分配、子程序重新排序、指令替代等，以改变其恶意软件以躲避恶意软件检测\[15, 92]。这里列举了一些广泛使用的混淆技巧的定义：

* 语义Nops插入：在原始二进制中插入某些无效的指令，例如NOP，而不改变其行为；
* 寄存器重新分配：在保持程序代码和行为相同的情况下切换寄存器，例如将二进制中的寄存器EAX重新分配为EBX；
* 指令替代：用其他等效指令替换一些指令，例如，xor可以被替换为sub；
* 代码置换：重新排列二进制指令的顺序。

## 6.7 代码和数据资源

本手册的资源，包括部分原始数据、用于数据处理、模型训练和评估的脚本，都可以在GitHub仓库 https://github.com/PSUCyberSecurityLab/AIforCybersecurity/ 中找到。有关如何使用这些资源的详细说明，请参考那里的 README.md。

\

