好的，根据你实际的完整操作记录（包括成功使用 `bcftools` 和 `ANNOVAR` 以及遇到的 Java 版本问题），我重新整理了一份更精准的项目报告。你可以直接使用。

---

# 全外显子组测序（WES）数据分析项目报告（真实数据版）

## 1. 项目目标
构建并执行一套完整的全外显子组测序（WES）数据分析流程，从原始 FASTQ 数据出发，经过比对、排序、变异检测，最终获得变异位点并进行功能注释。项目使用真实人类样本数据，验证流程的可行性和工具链的正确性。

## 2. 数据来源
- **样本**：NCBI SRA 真实人类样本，Accession：SRR8179797（NA12878 标准品的一部分，来源于 HG001）
- **数据量**：10,000 条双端 reads，每个 FASTQ 文件大小约 982 KB
- **参考基因组**：hg38（路径：`/home/database/reference/hg38/hg38.fa`）
- **数据位置**：`/home/database/pipeline/WES/input/10k_SRR8179797_1.fastq.gz` 和 `_2.fastq.gz`

## 3. 使用工具与环境
- **操作系统**：Linux（远程服务器，通过 Termius 连接）
- **核心工具及版本**：
  - `bwa` (0.7.17) —— 序列比对
  - `samtools` (1.22) —— SAM/BAM 处理
  - `bcftools` (1.22) —— 变异检测（因 GATK 4.6.1 需要 Java 17，服务器 Java 版本过低，改用 bcftools）
  - `ANNOVAR` (2025-07-21 数据库) —— 变异注释
- **环境配置**：将工具所在目录添加至 `PATH` 环境变量，解决了命令找不到的问题。

## 4. 分析流程与命令（按实际执行顺序）

### 4.1 数据准备
```bash
cd ~/my_wes_project
cp /home/database/pipeline/WES/input/10k_SRR8179797_1.fastq.gz .
cp /home/database/pipeline/WES/input/10k_SRR8179797_2.fastq.gz .
ls -lh 10k_SRR8179797_*.fastq.gz   # 确认文件大小（各 982K）
```

### 4.2 序列比对（bwa mem）
```bash
bwa mem -t 2 /home/database/reference/hg38/hg38.fa 10k_SRR8179797_1.fastq.gz 10k_SRR8179797_2.fastq.gz > SRR8179797.sam
```
**关键输出**：
- 共读取 20,000 条序列（双端各 10,000 条），总碱基数 3,020,000 bp。
- 候选唯一配对统计：FR 方向 4,870 对，RF 方向 7 对，FF 和 RR 为 0。
- 插入片段分布：均值 126 bp，标准差 51.78 bp，正常范围 (1, 380)。
- 比对耗时：CPU 22.956 秒，实际 14.633 秒。

### 4.3 格式转换与排序（samtools）
```bash
samtools view -bS SRR8179797.sam > SRR8179797.bam
samtools sort SRR8179797.bam -o SRR8179797.sorted.bam
samtools index SRR8179797.sorted.bam
```
生成排序后的 BAM 文件及其索引。

### 4.4 变异检测（bcftools）
由于服务器 Java 版本（仅支持到 Java 8）无法运行 GATK 4.6.1（需要 Java 17），改用轻量级工具 `bcftools`：
```bash
bcftools mpileup -f /home/database/reference/hg38/hg38.fa SRR8179797.sorted.bam | bcftools call -mv -o SRR8179797_bcftools.vcf
```
- 参数说明：`mpileup` 生成基因型似然，`call -mv` 进行变异检测（多等位基因，标准过滤）。
- 成功生成 VCF 文件，包含 10,644 个变异位点。

### 4.5 结果统计
```bash
# 总变异数
grep -v "^#" SRR8179797_bcftools.vcf | wc -l
# 输出：10644

# 变异类型分类（SNP vs Indel）
grep -v "^#" SRR8179797_bcftools.vcf | cut -f4-5 | awk '{if(length($1)==1 && length($2)==1) print "SNP"; else if(length($1)!=length($2)) print "Indel"; else print "Other"}' | sort | uniq -c
```
输出：
```
     70 Indel
  10574 SNP
```
SNP 占 99.34%，Indel 占 0.66%，符合预期。

### 4.6 变异注释（ANNOVAR）
检查数据库目录，发现实际文件名是 `hg38_clinvar_20250721.txt`，而非默认的 `hg38_clinvar.txt`。因权限不足无法创建软链接，直接在命令中指定正确的数据库名：
```bash
table_annovar.pl SRR8179797_bcftools.vcf /home/database/pipeline/WES_bin/annovar/humandb/ -buildver hg38 -out SRR8179797_ann -remove -protocol refGene,clinvar_20250721 -operation g,f -nastring . -vcfinput
```
**输出摘要**：
- 从 VCF 读取 10,697 行，通过 QC 的位点 10,644 个（10,574 SNP + 70 Indel）。
- 成功注释基因功能（refGene）和临床意义（clinvar_20250721）。
- 生成多注释文件：`SRR8179797_ann.hg38_multianno.txt` 和 VCF 格式输出。

### 4.7 查看注释结果示例
```bash
head SRR8179797_ann.hg38_multianno.txt
```
前几行显示了线粒体染色体（chrM）上的变异，位于基因间区或下游区域，ClinVar 数据为空（表示这些位点尚未在 ClinVar 中记录）。

### 4.8 保存操作记录
```bash
history > ~/wes_pipeline_commands.txt
```

## 5. 结果与讨论
- **变异总数**：10,644 个（SNP: 10,574；Indel: 70）
- **变异分布**：主要位于线粒体染色体（chrM）和常染色体，符合 WES 数据特征。
- **注释结果**：成功获取每个变异的基因名称、功能区域（exonic, intronic, intergenic 等）、氨基酸改变（如有）以及 ClinVar 致病性评级（部分位点有记录）。
- **流程验证**：通过真实人类数据完整运行了从 FASTQ 到注释的所有步骤，工具链工作正常，结果合理。虽然因 Java 版本问题未能使用 GATK，但 bcftools 同样可靠且快速，适合小规模数据或教学场景。

## 6. 遇到的问题及解决方案
| 问题 | 原因 | 解决方案 |
|------|------|----------|
| GATK 运行报错 `UnsupportedClassVersionError` | GATK 4.6.1 需要 Java 17，服务器只有 Java 8 | 改用 `bcftools` 进行变异检测 |
| ANNOVAR 报错 `hg38_clinvar.txt does not exist` | 数据库文件名是 `hg38_clinvar_20250721.txt` | 在 `-protocol` 中指定 `clinvar_20250721` 而非 `clinvar` |
| 无法创建软链接（`ln -s` 权限不足） | 数据库目录属于 `database` 用户，普通用户无写权限 | 不创建软链接，直接使用正确的数据库名 |

## 7. 结论
成功搭建并运行了完整的 WES 数据分析流程，使用真实人类样本数据（SRR8179797）完成了比对、排序、变异检测和注释，获得了超过 10,000 个变异位点及其功能注释。项目过程中解决了环境配置、Java 版本不兼容、数据库名称匹配等实际问题，积累了宝贵的经验。该流程可复现，为后续全外显子组测序分析（如疾病候选基因筛选）奠定了坚实基础。

## 8. 简历中的项目描述（建议）

> **项目：全外显子组测序（WES）数据分析流程构建与应用**  
> - 独立完成真实人类 WES 数据（SRR8179797，10k paired-end reads）的全流程分析，包括环境配置、序列比对（`bwa`）、SAM/BAM 处理（`samtools`）、变异检测（`bcftools`）和功能注释（`ANNOVAR`）。  
> - 共检出 10,644 个变异位点（其中 SNP 10,574 个，Indel 70 个），并成功注释基因功能和临床意义。  
> - 解决了 Java 版本不兼容、数据库名称匹配等实际问题，掌握了生信核心工具链的使用，具备处理真实测序数据的完整能力。  
> - 所有操作命令和结果已整理归档，流程可复现。

---

这份报告基于你实际执行的命令和输出，你可以直接保存为 `.md` 或 `.txt` 文件。如果需要调整格式或添加图表，可以再告诉我。恭喜你完成了这个高质量的项目！