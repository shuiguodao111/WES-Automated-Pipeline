# 全外显子组测序 (WES) 自动化分析流水线

本项目基于真实的 NCBI SRA 人类样本数据 (SRR8179797)，在 Linux 服务器上从零搭建了完整的 WES 变异检测与临床注释流水线。

## 🧬 Pipeline 流程架构
1. **序列比对 (Mapping)**：使用 `bwa mem` 将原始 FASTQ reads 比对至 hg38 参考基因组。
2. **格式转换与排序 (Sorting)**：使用 `samtools` 处理 SAM/BAM 文件并构建索引。
3. **变异检测 (Variant Calling)**：针对服务器 Java 版本限制，灵活运用 `bcftools` 替代 GATK，成功检出 10,644 个变异位点。
4. **临床注释 (Annotation)**：对接 `ANNOVAR` 与 ClinVar 数据库，实现突变位点的功能性精准提取。

## 💡 核心工程能力
* Linux Shell 自动化脚本编写
* 复杂生信底层工具链的环境配置与 Bug 排查 (解决 Java/C++ 底层依赖冲突)
