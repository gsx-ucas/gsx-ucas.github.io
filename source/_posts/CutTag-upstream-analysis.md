---
title: CUT&Tag 上游分析流程
date: 2023-6-10 18:12
tags: 
  - CUT&Tag
  - Chromatin
categories:
  - Analysis Tutorial
---

> 代码参考[cuttag文章分析流程推荐的代码](https://yezhengstat.github.io/CUTTag_tutorial/#VII_Visualization)，基于自己的数据和理解，对部分分析内容做了改动。
>
> **需要用到的软件如下：**
>
> - fastp: 去接头，生成质控报告
> - bowtie2: 比对测序数据到基因组
> - samtools/sambamba: 处理比对后的bam/sam文件，例如排序（sort）和去重（markdup）等
> - bedtools: 用大肠杆菌的基因组进行校准(Spike-in calibration)
> - MACS2: 进行peak calling，原流程还推荐了SEACR
> - homer: 对peak进行注释距离最近的基因、外显子、内含子、转录起始位点、基因间区等。
>
> **需要用到的数据如下：**
>
> - 参考基因组：可以从[NCBI](https://www.ncbi.nlm.nih.gov/genome/), [ENSEMBL](https://asia.ensembl.org/info/data/ftp/index.html) 和 [UCSC](https://hgdownload.soe.ucsc.edu/downloads.html) 三个数据库下载到。
> - 大肠杆菌基因组：官网推荐要用大肠杆菌的基因组进行校准, 可从[NCBI](https://www.ncbi.nlm.nih.gov/genome?term=DH5alpha&cmd=DetailsSearch)进行下载
> - blacklist_region: [基因组黑名单区域](https://github.com/Boyle-Lab/Blacklist/tree/master/lists)，这些区域在多种NGS测序数据中产生异常信号值，影响数据结果。因此进行peak calling之后需要将这些区域去除。
> - chromsize file: 记录基因组染色体大小的文件，这个文件可以在UCSC数据库中下载到，如人的基因组对应的该文件[hg38.chrom.sizes](https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/)

## 1. Reads去接头

在[原教程中](https://yezhengstat.github.io/CUTTag_tutorial/#311_Alignment_to_HG38)提到 *There is no need to trim reads from out standard 25x25 PE sequencing, as adapter sequences will not be included in reads of inserts >25 bp. However, for users performing longer sequencing, reads will need to be trimmed by Cutadapt and mapped by `--local --very-sensitive --no-mixed --no-discordant --phred33 -I 10 -X 700` to ignore any remaining adapter sequence at the 3’ ends of reads during mapping.* 

其意思是说对于PE25(reads长度为25的双端测序数据)，插入片段大多会大于25bp，所以不会有大量的接头序列存在，因此才不需要进行去接头的操作。另外，指定比对参数`--local`后bowtie2进行局部比对，这种模式下bowtie2会从比对的reads一端或者两端修剪掉那些不能比对的碱基，从而最大化比对分数。具体的解释可以参考原教程作者在[github issue](https://github.com/yezhengSTAT/CUTTag_tutorial/issues/3#issuecomment-841950393)中的回复。另外，教程作者也提到，可以尝试常用的ChIP-seq数据的比对方法，比较看一下哪种比对率更合理。

**以我的理解** 1. 长测序reads比如PE150，最好还是先执行去接头的操作，然后再进行比对。2. 只要比对率能够接受，用常用的比对参数进行比对也是可以的。
去接头的软件我推荐[fastp](https://github.com/OpenGene/fastp), 一是它去的干净且能够自动检测接头序列，二是它的速度真的很快。

``` snakemake
# 以下代码利用snakemake进行流程控制，具体规则可参考[snakemake文档](https://snakemake.readthedocs.io/en/stable/)。
input:
    r1='SeqData/LR/CD8_BCL11B_CUTTag/{samples}_R1.fastq.gz',
    r2='SeqData/LR/CD8_BCL11B_CUTTag/{samples}_R2.fastq.gz'
output:
    r1='Trim/{samples}_R1.fq.gz',
    r2='Trim/{samples}_R2.fq.gz'
shell:
    '''
        fastp -w 10 \
        --compression=2 \
        --detect_adapter_for_pe \
        -i {input.r1} \
        -I {input.r2} \
        -o {output.r1} \
        -O {output.r2} \
        -h Trim/{wildcards.samples}.html \
        -j Trim/{wildcards.samples}.json
    '''
```

## 2. Reads比对至基因组

这一步将进行两次比对，首先将reads比对到参考基因组(如人的hg38，小鼠的mm10)，然后将reads比对到大肠杆菌基因组. [原教程中](https://yezhengstat.github.io/CUTTag_tutorial/#V_Spike-in_calibration)提到*E. coli DNA is carried along with bacterially-produced pA-Tn5 protein and gets tagmented non-specifically during the reaction.* 因为在建库的过程中使用的重组蛋白A/G-MNase残留有大肠杆菌DNA，假设一系列样本，每个样本使用相同数量的细胞，那么比对到到参考基因组的reads与大肠杆菌基因组的比例是相同的。因此，可以通过计算比对到大肠杆菌基因组的reads比例来进行比对信号的标准化。

### 2.1 建立基因组index

``` bash
# 建立人基因组参考hg38的index
bowtie2-build hg38.chrom.fasta hg38_bt2index/genome
# 建立大肠杆菌E.coli的index
bowtie2-build E.coli.fasta E.coli_bt2index/genome
```

### 2.2 Reads比对到参考基因组

目前常用的人类参考基因组是hg38，但是hg38版本的参考基因组有很多未知位置的基因组片段(alternative contigs, 例如)，建议将这些序列去除，只用常规染色体(chr1..chr22, chX, chrY)进行分析。否则，reads可能不是唯一比对，因此将被分配低比对质量分数。[Biostars](https://www.biostars.org/p/342482/)上有关于这个问题的讨论。

``` snakemake
rule bowtie2:
input:
    r1='Trim/{samples}_R1.fq.gz',
    r2='Trim/{samples}_R2.fq.gz'
output:
    bam = 'Mapping/{samples}.bam',
    sum = 'Mapping/{samples}_aln_sum.txt'
shell:
    '''
        bowtie2 -X 700 -p 10 --end-to-end --very-sensitive \
        --no-mixed --no-discordant --phred33 -I 10 \
        -x hg38_bt2index/genome \
        -1 {input.r1} \
        -2 {input.r2} \
        2> {output.sum} | \
        samtools view -@ 10 -ShuF 4 -f 2 -q 30 - | \
        samtools sort -@ 10 -n - -T Mapping/{wildcards.samples}_tmp -o {output.bam}
    '''
```
---------------------------------------------------------------------------- 未完待续 ------------------------------------------------------------------------

