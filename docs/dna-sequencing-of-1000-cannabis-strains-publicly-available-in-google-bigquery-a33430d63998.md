# 谷歌大查询中公开的 1000 种大麻的 DNA 测序

> 原文：<https://medium.com/google-cloud/dna-sequencing-of-1000-cannabis-strains-publicly-available-in-google-bigquery-a33430d63998?source=collection_archive---------0----------------------->

通过 Google BigQuery 对新测序的大麻品种进行标准的生物信息学分析，可以帮助我们更好地理解该物种的生物学和进化史。

今天，我很高兴地宣布发布一个新的开放数据集，作为谷歌云 [BigQuery 公共数据集](https://cloud.google.com/bigquery/public-data/)项目的一部分，也是第一个在谷歌云上可用的基因组数据集(更多正在进行中)。

2016 年 10 月， [Phylos Bioscience](http://phylosbioscience.com/) 通过[开放大麻项目](http://opencannabisproject.org/)发布了约 850 株[大麻](https://en.wikipedia.org/wiki/Cannabis)的基因组开放数据集。结合由[medical Genomics](http://www.medicinalgenomics.com/)、密西根州立大学、NCBI、[Sunrise medical](http://www.sunrisemedicinal.com/)、卡尔加里大学、多伦多大学和云南省农业科学院提供的其他基因组数据集，可公开获得的数据总量超过 1000 个样本，这些样本取自几乎同样多的独特菌株。我总结了近 1000 种菌株的可用数据，并在[big query genomics _ 大麻数据集](https://bigquery.cloud.google.com/dataset/bigquery-public-data:genomics_cannabis)中公开。随附的[数据集描述](https://cloud.google.com/bigquery/public-data/1000-cannabis)中提供了如何探索该数据集的示例。

**为什么是大麻？**

大量文献描述了大麻作为农业物种的重要性。简而言之，它是纤维和营养及药用化合物的重要来源(用于治疗:疼痛、恶心、焦虑和炎症，因为大麻素化合物模拟我们身体自然产生的化合物，称为内源性大麻素，并在一系列环境中快速生长。

根据美国联邦法律，1937 年的《大麻税法》实际上将持有或转移大麻定为犯罪。联邦对大麻的定罪早于 1953 年詹姆斯·沃森和弗朗西斯·克里克发现 DNA 双螺旋结构。这一发现导致了分子生物学的产生，分子生物学关注于描述生物系统中基因的功能。由于 1937 年的定罪，与其他重要的驯化植物如玉米、水稻、小麦和大豆相比，对大麻的遗传学了解相对较少，因此大麻在耐寒性和产量优化方面落后。因此，有机会将标准的生物信息学分析应用于新测序的大麻品系，以使我们对该物种的生物学和进化史的理解现代化。

[开放大麻项目](http://opencannabisproject.org/)正在以 DNA 序列的形式向公共领域发布文件，以促进这种创新。开放大麻项目的贡献者以 sequencer 产生的格式发布了数据，这些数据可在 Google Cloud 上获得。然而，由于项目参与者没有提供进一步的数据分析，为了方便用户，我对数据集进行了额外的事实上的标准分析。具体来说，今天我们将发布以下分析结果:

1.  将所有样品与可用于创建更高质量大麻基因组装配的 *Cannatonic* 进行序列比对，以及
2.  从(1)中检测到遗传变异。基因变异是生物多样性的一个原子单位。

参见下面的方法部分，具体了解数据是如何准备的。

**为什么是现在？基因组测序:个体和群体**

单个个体的基因组序列对于建立该个体物种的进化史是有用的，但是对于开发包含该物种的应用(例如农业、工业和医学应用)是无用的。对相关个体群体进行测序测量了物种内的生物多样性，并实现了这些应用。虽然一些大麻基因组数据已经公开好几年了，但 Phylos Bioscience 最近发布了大量菌株的数据，这是开始对现有数据进行更深入的生物信息学分析的好时机。

为了理解为什么这是真的，并更全面地理解基因组时代的下一阶段，人们需要更多地了解基础生物学，以及为什么基因组测序是重要的。

虽然 DNA 测序技术的性价比已经有了显著的提高，但是这种进步并没有带来更小的项目和更低的成本；相反，随着生产 DNA 序列的好处被认识到，许多项目的预算正在增长。为什么会这样？

基因组测序项目可以分为两大类:

*   *从头*基因组测序，对一个尚未测序的物种个体基因组进行测序(产生一个*参考基因组*，以及
*   *重测序*，其中*从头*基因组已经产生，并且其中新的个体(查询)基因组被测序并且相对于现有(参考)基因组进行比较*。*

越来越多的医学和经济相关的基因组已经产生了新的基因组序列。那么，为什么我们要对大量额外的个体进行重新排序呢？

为了回答关于生物系统机制的问题，例如:

*   为什么有些人对传染病有抵抗力，而有些人却感染了？
*   为什么有些疾病会跟随家谱/被遗传？
*   在相同的生长条件下，为什么有些植物比其他植物产量高？

我们需要知道每个个体的基因构成，这样我们才能在个体有机体之间建立因果联系

1.  基因组状态，以及
2.  表型状态
3.  在它生活的环境中

对个体进行重新排序并测量相对于参考基因组的差异让我们可以做到这一点。一组个体被称为*群体*，如前所述，项目预算不会减少，因为基因组数据的真正价值来自于测量一个物种相对于参考基因组的*生物多样性*。*群体基因组学*是基因组学的下一阶段，我们可以在其中测量*群体生物多样性*。

**接下来的步骤**

既然大麻基因组数据已经有了，接下来的几个行动是可能的。除了应变关系的简单描述性分析，还需要更详细的描述性工作，包括:

*   **产生一个更高精度的基因组组装体。用于产生今天结果的 Cannatonic 组件是一个早期草案。更好的组合使得数据的所有后续使用具有更高的精度。**
*   **改进基因组注释**。大麻属有许多相关物种，其参考基因组已经产生。这些相关数据可以用于大麻，以帮助注释基因组——例如，识别基因的位置，并提出这些基因的功能特征。
*   **关联成对的基因组/表型观察结果并改进育种和驯化过程**。此处提供的数据仅是基因组数据。在公开发布的数据中，没有可用的表型数据。下一步是关联表型数据，例如一个品系的平均开花时间、开花产量的克数或 X 天的生物量。这使得通过像 [GBLUP](https://www.ncbi.nlm.nih.gov/pubmed/23756897) 这样的方法将基因组变异推广到 [QTL](https://en.wikipedia.org/wiki/Quantitative_trait_locus) s。这使得育种者可以利用[标记辅助选择](https://en.wikipedia.org/wiki/Marker-assisted_selection)中的 QTL，利用植物等同的 [EPD](https://en.wikipedia.org/wiki/Expected_progeny_difference) s，加速植物的驯化。
*   **降低供应链风险**。鉴定与菌株相关的特定遗传标记能够对植物组织进行遗传鉴定。这反过来又能改善供应链质量控制。

接下来的每一步都需要进一步的生物信息学分析，所有这些都可以在谷歌云上进行，其中许多都可以通过使用今天发布到 BigQuery 的准备好的数据来大大简化。

总之:大麻是一种重要的农业和药用植物。最近公布的大量大麻品种数据标志着现在是开始更深入的生物信息学分析的好时机。Google Cloud 提供了高性能的工具来处理这些数据，如 BigQuery 和 Dataflow，包括通过 Genomics API 处理人口规模基因组数据的高度专业化的工具。

**方法**

使用 [sra-tools](https://github.com/ncbi/sra-tools) 从国家生物技术信息中心的序列读取档案(NCBI SRA)下载 FastQ 数据文件，用于九个[项目](https://www.ncbi.nlm.nih.gov/bioproject/?term=txid3483%5BOrganism:exp%5D)的大麻 DNA 读取数据。

同时，[参考基因组](https://www.ncbi.nlm.nih.gov/genome/genomes/11681)(针对品系*大麻*亚种。[](https://www.leafly.com/hybrid/cannatonic)**[*LA Confidential*](https://www.leafly.com/indica/la-confidential)*[*chem dawg 91*](https://www.leafly.com/hybrid/chemdawg-91)*[*紫库什*](https://www.leafly.com/indica/purple-kush) )也被下载，所有进一步的分析都使用了[](https://www.leafly.com/hybrid/cannatonic)*菌株作为参考(GenBank[mnprpr)SRA FastQ 原始数据可在 SRA-projects 文件夹的](https://www.ncbi.nlm.nih.gov/Traces/wgs/?val=MNPR01)[GS://GCS-public-data-genomics/大麻](https://storage.googleapis.com/gcs-public-data--genomics/cannabis)的 [Google 云存储](https://cloud.google.com/storage)中获得。*****

***使用 [BWA 记忆](http://bio-bwa.sourceforge.net/bwa.shtml)算法将每个样本的 DNA 读数与 [MNPR01](https://www.ncbi.nlm.nih.gov/Traces/wgs/?val=MNPR01) 进行比对。通过 [Google Genomics API](https://cloud.google.com/genomics/) 可以获得对齐的读数，作为数据集 ID [918853309083001239](https://pantheon.corp.google.com/genomics/datasets/918853309083001239) ，以及仅转录组数据的额外复制子集，作为数据集 ID [94241232795910911](https://pantheon.corp.google.com/genomics/datasets/94241232795910911) 。***

***然后使用 [Freebayes](https://github.com/ekg/freebayes) 算法处理比对读数，以确定每个菌株相对于参照的可能遗传变异的位置。对每个样本进行变异调用，并且对样本的变异调用可通过数据集 ID 17527604790083478309 下的[谷歌基因组 API](https://cloud.google.com/genomics/) 获得。***

***最后，变体被导出到 BigQuery [1000 大麻基因组:公共数据集](https://cloud.google.com/bigquery/public-data/1000-cannabis)。***