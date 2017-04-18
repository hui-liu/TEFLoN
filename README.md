*TEFLoN*
=======

TEFLoN uses paired-end illumina sequence data to discover the positions and identity of TEs present in one or multiple samples.
Input data can either be individually sequenced or sequenced as a pool, and multiple samples can be analyzed simultaneously to improve sensitivity.

## Required packages

* [python](www.python.org) v2.7
* [samtools](www.samtools.sourceforge.net) (tested using v.1.3)
* [Burrows-Wheeler Aligner](www.bio-bwa.sourceforge.net) (tested using v.0.7.10)
* [RepeatMasker](www.repeatmasker.org) (Only required if you do not have a reference TE annotation)

## Testing 
Sample files are provided to ensure TEFLoN and its dependencies are running correctly.

## Usage
### Data prep
Step 1a) If you have a reference TE annotation in BED format, use teflon_prep_w_anno.py to prepare your reference genome for mapping.

NOTE: You must also manually create a file where each entry in the annotation has at least one label (idealy this label would indicate the family or order for each TE instance, but you can use any label you like.)
TEFLoN will cluster or split reads mapping to the locations specified in the annotation by their respective labels from this user created hierarchy file.
The first line of the file must include identifying headers and each ID/label must be separated by a tab.
A good example of what this hierarchy file should look like is provided in the sample files.

```
usage: python /usr/local/teflon_prep_w_anno.py <required> [optional] 
    -wd <full path to working directory>
    -a <full path to reference TE annotation in BED format> 
    -t <full path to user generated TE hierarchy>
    -g <full path to reference genome in fasta format>
    -p <prefix for all newly created files>
```

Step 1b) If you *do not* have an existing TE annotation in BED format, use teflon_prep_no_anno.py to prepare your reference genome for mapping.

NOTE: In this case you will not need to manually create a hierarchy file, as TEFLoN will automatically create one based on the labels provided in the repBase library.

```
usage: python /usr/local/teflon_prep_no_anno.py <required> [optional] 
    -wd <full path to working directory>
    - <full path to repeatMasker executable> 
    -g <full path to reference genome in fasta format>
    -l <full path to repBase_library.ref>
    -p <prefix for all newly created files>
    -r <average length of sequenced reads>
    -n [number of threads]
```

Important output files:
* /usr/local/prep_TF/prefix.te.pseudo.bed
* /usr/local/prep_TF/prefix.genomeSize.txt
* /usr/local/prep_TF/prefix.hier
* /usr/local/prep_TF/prefix.pseudo2ref.pickle.gz
* /usr/local/prep_TF/prefix.ref2pseudo.pickle.gz
* /usr/local/prep_MP/prefix.mappingRef.fa

Step 2) Index the prepped reference genome. 
```
bwa index <prefix.mappingRef.fa>
```
Step 3) For each sample, map reads with BWA mem or a similar mapping software package (i.e. one that is able to identify and mark soft-clipped reads).
```
bwa mem -t <nThreads> -Y <prefix.mappingRef.fa> <read1.fq> <read2.fq> > alignment.sam
```
Step 4) For each sample, sort and then index the alignment produced by BWA.
```
samtools view -Sb <alignment.sam> | samtools sort -@ <nThreads> - -o alignment.sorted.bam

samtools index <alignment.sorted.bam>
```
Note: Ideally, you should have also QC processed your raw reads and removed duplicates from the alignment.

Step 5) Create a .txt file (e.g. samples.txt), where each new line contains both the full path to the indexed and sorted alignment.bam file and a unique prefix/nickname for that sample separated by a tab.
Currently, this step is necessary even if you are only analyzing a single sample.

```
Example File:
/usr/local/sample1.sorted.bam  s1
/usr/local/sample2.sorted.bam  s2
```

You are now ready to proceed to using TEFLoN.

### Using TEFLoN
There are four modules to TEFLoN: teflonDiscover, teflonCollapse, teflonCount, and teflonGenotype.
You must run teflonDiscover and teflonCount seperately for each sample. These modules may run independently for each sample (i.e. you can run all samples simultaneosly with enough threads).
TeflonCollapse and teflonGenotype only need to run once per analysis.

Step 1) For each sample, run teflonDiscover.
```
usage: python usr/local/teflon.v0.2.py <required> [optional]
    -wd <full path to working directory>
    -ex <full path to samtools executable>
    -g <full path to genomeSize.txt> #created by teflon_prep
    -s <full path to samples.txt> #user created
    -b <full path to sorted and indexed bam file>
    -a <full path to te.psuedo.bed> #created by teflon_prep
    -t <full path to TE hierarchy file> #created by teflon_prep
    -l <level of the hierarchy file to guide initial TE search> #recommended "family" (note: level must appear in the first line of the TE hierarch file)
    -cl <level of the hierarchy to cluster similar TEs> #can be same "level" of hierarchy used in -l or higher (raising level will reduce the number of TE instances found)
    -e [newline separated file of any te families to ignore from analysis]
    -q <map quality threshold> #mapped reads with map qualities lower than this number will be discarded
    -sd [int] #use to manually override the insert size sd identified by samtools stat (check this number to ensure it seems more or less correct based on knowledge of sequencing library!)
    -x [number of threads]
```

Step 2) Run teflonCollapse.
```
usage: python usr/local/teflon_collapse.py <required> [optional]
    -wd <full path to working directory>
    -s <full path to samples.txt>
    -t [number of threads]
```

Step 3) For each sample, run teflonCount.
```
usage: python /usr/local/teflon_count.py <required> [optional]
    -wd <full path to working directory>
    -ex <full path to samtools executable>
    -g <full path to genomeSize.txt>
    -s <full path to samples.txt>
    -b <full path to sorted and indexed bam file>
    -a <full path to te.psuedo.bed>
    -t <full path to TE hierarchy file>
    -q <map quality threshold>
    -x [number of threads]
```

Step 4) Run teflonGenotype.
NOTE: Currently haploid and diploid data types are under construction!
```
usage: python usr/local/teflon_genotype.py <required> [optional]
    -wd <full path to working directory>
    -s <full path to samples.txt>
    -p <full path to psuedo2ref.pickle.gz file>
    -t <data type> #must be either haploid, diploid, or pooled #Currently, all types must use pooled for read counts
```

### Output
The output "genotype" file is currently a tab-separated file with the following columns:
```
C1: chromosome
C2: 5' breakpoint (if available)
C3: 3' breakpoint (if available)
C4: search level id
C5: cluster level id
C6: strand (currently under construction)
C7: annotated reference TE id (if available)
C8: 5' breakpoint is supported by soft-clipped reads (if TRUE + else -)
C9: 3' breakpoint is supported by soft-clipped reads (if TRUE + else -)
C10: read count for "presence reads"
C11: read count for "absence reads"
C12: read count for "other reads"
C13: allele frequency/genotype (fq if pooled data, genotype for haploid/diploid currently under construction)

```

### Citation
If you use TEFLoN in your work please cite:
```
Adrion, J.R., M.J. Song, D.R. Schrider, M.W. Hahn, and S. Schaack. 2017. Genome-wide estimates of transposable element insertion and deletion rates in *Drosophila melanogaster*. Genome Biology and Evolution. https://doi.org/10.1093/gbe/evx050.
```
