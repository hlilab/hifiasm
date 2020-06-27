## Getting Started

```sh
# Install hifiasm (requiring g++ and zlib)
git clone https://github.com/chhylp123/hifiasm
cd hifiasm && make

# Run on test data (use -f0 for small datasets)
wget https://github.com/chhylp123/hifiasm/releases/download/v0.7/chr11-2M.fa.gz
./hifiasm -o test -t4 -f0 chr11-2M.fa.gz 2> test.log   # this takes ~90 sec

# Assemble inbred/homozygous genomes (-l0 disables duplication purging)
hifiasm -o CHM13.asm -t32 -l0 CHM13-HiFi.fa.gz
# Assemble heterozygous with built-in duplication purging
hifiasm -o HG002.asm -t32 HG002-file1.fq.gz HG002-file2.fq.gz

# Trio binning assembly (requiring https://github.com/lh3/yak)
yak count -b37 -t16 -o pat.yak <(cat pat_1.fq.gz pat_2.fq.gz) <(cat pat_1.fq.gz pat_2.fq.gz)
yak count -b37 -t16 -o mat.yak <(cat mat_1.fq.gz mat_2.fq.gz) <(cat mat_1.fq.gz mat_2.fq.gz)
hifiasm -o HG002.asm -t32 -1 pat.yak -2 mat.yak HG002-HiFi.fa.gz
```

## Introduction

Hifiasm is a fast haplotype-resolved de novo assembler for PacBio
Hifi reads. Unlike most existing assemblers, hifiasm starts from uncollapsed
genome. Thus, it is able to keep the haplotype information as much as possible.
Hifiasm supports both primary assembly and fully phased assembly (trio-binning assembly).



## Usage

For Hifi reads assembly, a typical command line looks like:

```sh
./hifiasm -o NA12878.asm -t 32 NA12878.fq.gz
```

where `NA12878.fq.gz` is the input reads and `-o` specifies the output files.
In this example, all output files can be found at `NA12878.asm.*`. `-t` specifies 
the number of CPU threads. Note that at first run, hifiasm will save all overlaps 
to disk, which can avoid the time-consuming all-to-all overlap calculation next time. 
For hifiasm, once the overlap information has been obtained during the previous run 
in advance, it is able to load all overlaps from disk and then directly do assembly. 
If you want to ignore the pre-computed overlap information, please specify `-i`. 
The example dataset can be found at: https://github.com/chhylp123/hifiasm/releases/download/v0.7/HG002-34X-chr11-19310012.-.21493943.fastq.gz.

Please note that some old Hifi reads may consist of short adapters. To improve
the assembly quality, adapters should be removed by `-z` as follow:

```sh
./hifiasm -o butterfly.asm -t 42 -z 20 butterfly.fq.gz
```

In this example, hifiasm will remove 20 bases from both ends of each read.

For trio assembly, first the trio indexes of paternal/maternal should be generated by 
[yak count](https://github.com/lh3/yak):

```sh
./yak count -k31 -b37 -t16 -o mat.yak mat.fq.gz
./yak count -k31 -b37 -t16 -o pat.yak pat.fq.gz
```

and then run hifiasm as follow:

```sh
./hifiasm -o NA12878.asm -t 32 -1 pat.yak -2 mat.yak NA12878_1.fq.gz NA12878_2.fq.gz
```

## Advance features

For primary assembly, hifiasm performs purge duplication step in default. This step is designed for diploid genomes or polyploid genomes to keep one set of haplotypes without duplications. For inbred genomes or homozygous samples including only one haplotype, purge duplication step may introduce misassemblies so that it should be disabled by option `-l0`.


## Output

For non-trio assembly, the input of hifiasm is the PacBio Hifi reads in fasta/fastq format, and its
outputs consist of: 

1. Haplotype-resolved raw [unitig][unitig] graph in [GFA][gfa] format
   (*prefix*.r\_utg.gfa). This graph keeps all haplotype information, including
   somatic mutations and recurrent sequencing errors.
2. Haplotype-resolved processed unitig graph without small bubbles
   (*prefix*.p\_utg.gfa). Small bubbles might be caused by somatic mutations or noise in data, 
   which are not the real haplotype information.
3. Primary assembly [contig][unitig] graph (*prefix*.p\_ctg.gfa). This graph collapses different
   haplotypes.
4. Alternate assembly contig graph (*prefix*.a\_ctg.gfa). This graph consists of all assemblies that
   are discarded in primary contig graph.

For trio assembly, the input of hifiasm is the PacBio Hifi reads in fasta/fastq format, and the paternal/maternal trio indexes generated by [yak count](https://github.com/lh3/yak). The outputs consist of:
1. Haplotype-resolved raw [unitig][unitig] graph in [GFA][gfa] format
   (*prefix*.r\_utg.gfa). This graph keeps all haplotype information. 

2. Phased paternal/haplotype1 contig graph (*prefix*.hap1.p\_ctg.gfa). This graph keeps the phased
   paternal/haplotype1 assembly.

3. Phased maternal/haplotype2 contig graph (*prefix*.hap2.p\_ctg.gfa). This graph keeps the phased
   maternal/haplotype2 assembly.


In addition, hifiasm also outputs three binary files that save all overlap information (*prefix*.ec.bin, *prefix*.ovlp.reverse.bin, *prefix*.ovlp.source.bin). With these files, hifiasm can avoid the time-consuming all-to-all overlap calculation step, and do the assembly
directly and quickly. This might be helpful when you want to get an optimized
assembly by multiple rounds of experiments with different parameters.

## Results

Hifiasm is a standalone and lightweight assembler, which does not need external
libraries (except zlib). For large genomes, it can generate high-quality primary
assembly in several hours. Hifiasm has been tested on various large and complex datasets. 
The results are as follows: 

|<sub>Dataset<sub>|<sub>Size<sub>|<sub>Cov.<sub>|<sub>Asm options<sub>|<sub>CPU time<sub>|<sub>Wall time<sub>|<sub>RAM<sub>|<sub> N50<sub>|
|:---------------|-----:|-----:|:---------------------|-------:|--------:|----:|----------------:|
|<sub>[Mouse (C57/BL6J)][mouse-data]</sub>|<sub>2.6Gb</sub> |<sub>&times;25</sub>|<sub>-t48 -l0</sub> |<sub>172.9h</sub> |<sub>4.8h</sub> |<sub>76G</sub> |<sub>21.1Mb</sub>|
|<sub>[Maize (B73)][maize-data]</sub>     |<sub>2.2Gb</sub> |<sub>&times;22</sub>|<sub>-t48 -l0</sub> |<sub>203.2h</sub> |<sub>5.1h</sub> |<sub>68G</sub> |<sub>36.7Mb</sub>|
|<sub>[Strawberry][strawberry-data]</sub> |<sub>0.8Gb</sub> |<sub>&times;36</sub>|<sub>-t48 -D10</sub>|<sub>152.7h</sub> |<sub>3.7h</sub> |<sub>91G</sub> |<sub>17.8Mb</sub>|
|<sub>[Frog][frog-data]</sub>             |<sub>9.5Gb</sub> |<sub>&times;29</sub>|<sub>-t48</sub>     |<sub>2834.3h</sub>|<sub>69.0h</sub>|<sub>463G</sub>|<sub>9.3Mb</sub>|
|<sub>[Redwood][redwood-data]</sub>       |<sub>35.6Gb</sub>|<sub>&times;28</sub>|<sub>-t80</sub>     |<sub>3890.3h</sub>|<sub>65.5h</sub>|<sub>699G</sub>|<sub>5.4Mb</sub>|
|<sub>[Human (CHM13)][CHM13-data]</sub>   |<sub>3.1Gb</sub> |<sub>&times;32</sub>|<sub>-t48 -l0</sub> |<sub>310.7h</sub> |<sub>8.2h</sub> |<sub>114G</sub>|<sub>88.9Mb</sub>|
|<sub>[Human (HG00733)][HG00733-data]</sub>|<sub>3.1Gb</sub>|<sub>&times;33</sub>|<sub>-t48</sub>     |<sub>269.1h</sub> |<sub>6.9h</sub> |<sub>135G</sub>|<sub>69.9Mb</sub>|
|<sub>[Human (HG002)][NA24385-data]</sub> |<sub>3.1Gb</sub> |<sub>&times;36</sub>|<sub>-t48</sub>     |<sub>305.4h</sub> |<sub>7.7h</sub> |<sub>137G</sub>|<sub>98.7Mb</sub>|

[mouse-data]:      https://www.ncbi.nlm.nih.gov/sra/?term=SRR11606870
[maize-data]:      https://www.ncbi.nlm.nih.gov/sra/?term=SRR11606869
[strawberry-data]: https://www.ncbi.nlm.nih.gov/sra/?term=SRR11606867
[frog-data]:       https://www.ncbi.nlm.nih.gov/sra?term=(SRR11606868)%20OR%20SRR12048570
[redwood-data]:    https://www.ncbi.nlm.nih.gov/sra/?term=SRP251156
[CHM13-data]:      https://www.ncbi.nlm.nih.gov/sra?term=(((SRR11292120)%20OR%20SRR11292121)%20OR%20SRR11292122)%20OR%20SRR11292123

Hifiasm also can produce high-quality fully phased assembly. We tested it on the following trio-binning datasets:
|<sub>Dataset<sub>|<sub>Cov.<sub>|<sub>CPU time<sub>|<sub>Elapsed time<sub>|<sub>RAM<sub>|<sub> N50<sub>|
|:---------------|-----:|-------:|--------:|----:|----------------:|
|<sub>[HG00733][HG00733-data], [\[father\]][HG00731-data], [\[mother\]][HG00732-data]</sub>|<sub>&times;33</sub>|<sub>269.1h</sub>|<sub>6.9h</sub>|<sub>135G</sub>|<sub>35.1Mb (paternal), 34.9Mb (maternal)</sub>|
|<sub>[HG002][NA24385-data],   [\[father\]][NA24149-data], [\[mother\]][NA24143-data]</sup>|<sub>&times;36</sub>|<sub>305.4h</sub>|<sub>7.7h</sub>|<sub>137G</sub>|<sub>41.0Mb (paternal), 40.8Mb (maternal)</sub>|
|<sub>[NA12878][NA12878-data], [\[father\]][NA12891-data], [\[mother\]][NA12892-data]</sub>|<sub>&times;30</sub>|<sub>180.8h</sub>|<sub>4.9h</sub>|<sub>123G</sub>|<sub>27.7Mb (paternal), 27.0Mb (maternal)</sub>|

[HG00733-data]: https://www.ebi.ac.uk/ena/data/view/ERX3831682
[HG00731-data]: https://www.ebi.ac.uk/ena/data/view/ERR3241754
[HG00732-data]: https://www.ebi.ac.uk/ena/data/view/ERR3241755
[NA24385-data]: https://www.ncbi.nlm.nih.gov/sra?term=(((SRR10382244)%20OR%20SRR10382245)%20OR%20SRR10382248)%20OR%20SRR10382249
[NA24149-data]: https://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/data/AshkenazimTrio/HG003_NA24149_father/NIST_HiSeq_HG003_Homogeneity-12389378/HG003Run01-13262252/
[NA24143-data]: https://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/data/AshkenazimTrio/HG004_NA24143_mother/NIST_HiSeq_HG004_Homogeneity-14572558/HG004Run01-15133132/
[NA12878-data]: https://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/data/NA12878/PacBio_SequelII_CCS_11kb/
[NA12891-data]: https://www.ebi.ac.uk/ena/data/view/ERR194160
[NA12892-data]: https://www.ebi.ac.uk/ena/data/view/ERR194161

Except NA12878, the assemblies above were produced by hifiasm v0.7 and can be
downloaded at
```txt
ftp://ftp.dfci.harvard.edu/pub/hli/hifiasm/submission/v0.7/
```
NA12878 was assembled with a more recent version of hifiasm and is available at
```txt
ftp://ftp.dfci.harvard.edu/pub/hli/hifiasm/NA12878-r253/
```


[unitig]: http://wgs-assembler.sourceforge.net/wiki/index.php/Celera_Assembler_Terminology
[gfa]: https://github.com/pmelsted/GFA-spec/blob/master/GFA-spec.md
[paf]: https://github.com/lh3/miniasm/blob/master/PAF.md

## Getting Help

For detailed description of options, please see `man ./hifiasm.1`.
The `-h` option of hifiasm also provides simple description of options. If you
have further questions, please raise an issue at the issue page.

## Limitations and future works

1. The running time and memory usage should be further reduced.

2. The N50 should be further improved. 
