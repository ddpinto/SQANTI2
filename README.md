# SQANTI2

Last Updated: 2018/10/15

## Updates

2018.10.15 updated to version 1.1. modified use of SAM to GFF with added `source` parameter.

Private repo for Liz's modified SQANTI. The original [SQANTI](https://bitbucket.org/ConesaLab/sqanti) is by Ana Conesa lab.

![](https://github.com/Magdoll/images_public/blob/master/github_isoseq3_wiki_figures/wiki_SQANTI_sample_output.png)

Until the SQANTI authors have finalized their agreement with Liz on how to integrate Liz's changes, the script names are modified to reflect this as a temporary working version.

For example, `sqanti_qc.py` is named currently `sqanti_qc2.py`.

## Prerequisite

* Perl
* Minimap2 

### Python-related libraries

* Python (2.7)
* pysam
* psutil
* bx-python
* BioPython
* BCBioGFF
* [cDNA_Cupcake](https://github.com/Magdoll/cDNA_Cupcake/wiki/Cupcake-ToFU:-supporting-scripts-for-Iso-Seq-after-clustering-step#install)

### R-related libraries

* R (>= 3.4.0)
* R packages (for `sqanti_qc2.py`): ggplot2, scales, reshape, gridExtra, grid, dplyr

## Installing Python dependencies

I recommend using Anaconda which makes installing all the Python packages much easier. If you already have Anaconda installed because you use [Iso-Seq3](https://github.com/PacificBiosciences/IsoSeq_SA3nUP/wiki/Tutorial:-Installing-and-Running-Iso-Seq-3-using-Conda) or [cDNA_Cupcake](https://github.com/Magdoll/cDNA_Cupcake/wiki/Cupcake-ToFU:-supporting-scripts-for-Iso-Seq-after-clustering-step), you can activate your current environment and directly go to step (4).

(1)  Here's the generic Anaconda installation for [Linux environment](http://docs.continuum.io/anaconda/install/#linux-install). Currently only Linux environment supported.

```
bash ~/Downloads/Anaconda2-5.2.0-Linux-x86_64.sh
export PATH=$HOME/anaconda5.2/bin:$PATH
conda -V
conda update conda
```

(2) Create a virutal environment. I will call it `anaCogent5.2`. Type `y` to agree to the interactive questions.

```
conda create -n anaCogent5.2 python=2.7 anaconda
source activate anaCogent5.2
```

(3) Once you have activated the virtualenv, you should see your prompt changing to something like this:

```
(anaCogent5.2)-bash-4.1$
```

(4) Install additional required libraries:

```
conda install -n anaCogent5.2 -c bioconda pysam
conda install -n anaCogent5.2 psutil
conda install -n anaCogent5.2 biopython
conda install -n anaCogent5.2 -c http://conda.anaconda.org/cgat bx-python
conda install -n anaCogent5.2 -c bioconda bcbiogff
```

If you don't already have [cDNA_Cupcake](https://github.com/Magdoll/cDNA_Cupcake/wiki/Cupcake-ToFU:-supporting-scripts-for-Iso-Seq-after-clustering-step#install) installed, you can do that now:

```
$ git clone https://github.com/Magdoll/cDNA_Cupcake.git
$ cd cDNA_Cupcake
$ python setup.py build
$ python setup.py install
```

No installation for SQANTI2 itself is required. The scripts can be run directly.


## Running SQANTI2

Activate the Anaconda environment. Make sure minimap2 works. Add `cDNA_Cupcake/sequence` to `$PYTHONPATH`.

```
$ source activate anaCogent5.2
(anaCogent5.2)-bash-4.1$ export PYTHONPATH=$PYTHONPATH:<path_to>/cDNA_Cupcake/sequence/
(anaCogent5.2)-bash-4.1$ minimap2 --version
2.11-r797
```

#### Input to SQANTI QC

* *Iso-Seq output*. Preferably already mapped to the genome and [collapsed to unique transcripts](https://github.com/Magdoll/cDNA_Cupcake/wiki/Cupcake-ToFU:-supporting-scripts-for-Iso-Seq-after-clustering-step#collapse). (FASTA/FASTQ)
* *Reference annotation* in GTF format. For example [GENCODE](https://www.gencodegenes.org/releases/current.html) or [CHESS](http://ccb.jhu.edu/chess/).
* *Reference genome*, in FASTA format. For example hg38. *Make sure your annotation GTF is based on the correct ref genome version!*

### Running SQANTI QC

The script usage is:

```
python sqanti_qc2.py [-t cpus] [--skipORF] [-c shortread_STAR_junction_out]
     <input_fasta> <annotation_gtf> <genome_fasta>
```

If you don't feel like running the ORF prediction part, use `--skipORF`. Just know that all your transcripts will be annotated as non-coding.
If you have short read data, you can run STAR to get the junction file (usually called `SJ.out.tab`, see [STAR manual](https://github.com/alexdobin/STAR/blob/master/doc/STARmanual.pdf)) and supply it to SQANTI2.


For example:

```
python sqanti_qc2.py -t 30 example/touse.rep.fasta gencode.v28.annotation.gtf hg38.fa
```

### SQANTI QC output

You can look at the [example](https://github.com/Magdoll/SQANTI2/tree/master/example) subfolder for a sample output. The PDF file shows all the figures drawn using R script [SQANTI_report2.R](https://github.com/Magdoll/SQANTI2/blob/master/utilities/SQANTI_report2.R), taking the `_classification.txt` and `_junctions.txt` as the two input. If you know R well, you are free to modify the R script to add new figures! I will be constantly adding new figures as well.

Detailed explanation of `_classification.txt` and `_junctions.txt` <a href="#explain">below</a>.


### Filtering Isoforms using SQANTI


I've made a lightweight filtering script based on SQANTI2 output that filters for two things: (a) intra-priming and (b) short read junction support.  

The script usage is:

```
python sqanti_filter2.py <classification_txt> <input_fasta>
         [-a INTRAPRIMING] [-c MIN_COV]
```

where `-a` determines the fraction of genomic 'A's above which the isoform will be filtered. The default is `-a 0.8`. `-c` is the filter for the minimum short read junction support (looking at the `min_cov` field in `.classification.txt`), and can only be used if you have short read data.


For example:

```
python sqanti_filter2.py touse.rep_classification.txt touse.rep.fasta
```

<a name="explain"/>

### SQANTI2 Output Explanation


SQANTI/SQANTI2 categorizes each isoform by finding the best matching reference transcript in the following order:

* FSM (*Full Splice Match*): meaning the reference and query isoform have the same number of exons and each internal junction agree. The exact 5' start and 3' end can differ by any amount.

* ISM (*Incomplete Splice Match*): the query isoform has 5' exons than the reference, but each internal junction agree. The exact 5' start and 3' end can differ by any amount.

* NIC (*Novel In Catalog*): the query isoform does not have a FSM or ISM match, but is using a combination of known donor/acceptor sites.

* NNC (*Novel Not in Catalog*): the query isoform does not have a FSM or ISM match, and has at least one donor or acceptor site that is not annotated.

* *Antisense*: the query isoform does not have overlap a same-strand reference gene but is anti-sense to an annotated gene. 

* *Genic Intron*: the query isoform is completely contained within an annotated intron.

* *Genic Genomic*: the query isoform overlaps with introns and exons.

* *Intergenic*: the query isoform is in the intergenic region.


![sqanti_cat_explain](https://github.com/Magdoll/images_public/blob/master/github_isoseq3_wiki_figures/wiki_SQANTI_categorization_explanation.png)


#### Classification Output Explanation

The output `.classification.txt` has the following fields:

1. `isoform`: the isoform ID. Usually in `PB.X.Y` format.
2. `chrom`: chromosome.
3. `strand`: strand.
4. `length`: isoform length.
5. `exons`: number of exons.
6. `structural_category`: one of the categories ["full-splice_match", "incomplete-splice_match", "novel_in_catalog", "novel_not_in_catalog", "genic", "antisense", "fusion", "intergenic", "genic_intron"]
7. `associated_gene`: the reference gene name.
8. `associated_transcript`: the reference transcript name.
9. `ref_length`: reference transcript length.
10. `ref_exons`: reference transcript number of exons.
11. `diff_to_TSS`: distance of query isoform 5' start to reference transcript start end. Negative value means query starts downstream of reference.
12. `diff_to_TTS`: distance of query isoform 3' end to reference annotated end site. Negative value means query ends upstream of reference.
13. `subcategory`: A/B/C. Ignore for now.
14. `RTS_stage`: TRUE if one of the junctions could be a RT switching artifact.
15. `all_canonical`: TRUE if all junctions have canonical splice sites.
16. `min_sample_cov`: sample with minimum coverage.
17. `min_cov`: minimum junction coverage based on short read STAR junction output file. NA if no short read given.
18. `min_cov_pos`: the junction that had the fewest coverage. NA if no short read data given.
19. `sd_cov`: standard deviation of junction coverage counts from short read data. NA if no short read data given.
20. `FL`: currently always NA. I will add back this information later.
21. `n_indels`: total number of indels based on alignment.
22. `n_indels_junc`: number of junctions in this isoform that have alignment indels near the junction site (indicating potentially unreliable junctions).
23. `bite`: TRUE if all junctions match reference junctions completely.
24. `iso_exp`: currently always NA. I will add back this information later.
25. `gene_exp`: currently always NA. I will add back this information later.
26. `ratio_exp`: currently always NA. I will add back this information later.
27. `FSM_class`: ignore this field for now.
28. `ORF_length`: predicted ORF length.
29. `CDS_length`: predicted CDS length. 
30. `CDS_start`: CDS start.
31. `CDS_end`: CDS end.
32. `perc_A_downstreamTTS`: percent of genomic "A"s in the downstream 20 bp window. If this number if high (say > 0.8), the 3' end site of this isoform is probably not reliable.


