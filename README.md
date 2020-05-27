# Poke
Report salient qPCR primer/probe mismatches info against a set of pathogen isolate genomes

## Pronuciation
/poʊˈkeɪ/ as in the [delicious Hawaiian raw fish dish](https://en.wikipedia.org/wiki/Poke_(Hawaiian_dish)), or P-okay as in "my PCR design is okay".

## Motivation
Does my PCR assay design detect all known sequenced examples of a pathogen?

This lighweight tools was developed to simplify the process of checking for continued detection sensitivity of PCR assays in light of available pathogen genomic data. For example, as of this writing the genomes of more than 30,000 SARS-CoV-2 samples are available in [GISAID](https://gisaid.org/CoV2020) and [at the NCBI](https://www.ncbi.nlm.nih.gov/genbank/sars-cov-2-seqs/) thanks to global sequencing efforts, and a lab may want to check if their PCR assay's primers/probes continue to be exact matches against all samples despite inevitable viral evolution. This software complements existing effort to track mismatche counts for published SARS-CoV-2 PCR assays, such as the very useful [EDGE validation portal](https://covid19.edgebioinformatics.org/#/assayValidation).

## Quick Start
This script depends on you having a preinstalled version of [NCBI BLAST](https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/LATEST/), and has only been tested using version 2.9.0+. It can be downloaded from the provided link, or if you are running the [conda manager](https://docs.conda.io/projects/conda/en/latest/user-guide/install/), this should be as easy as ```conda install blast```. It also requires any version of Perl (there are no module dependencies).

```shell
poke file_of_genomes.fasta file_of_genome_ids_to_exclude.txt pcr_primers_and_or_probes.fasta > salient_mismatch_info_output.txt
```

Here ```file_of_genome_ids_to_exclude.txt``` allows you to remove duplicate or didgy genomes from the analysis without having to edit the genomes file that you may have downloaded in bulk from somewhere like the GISAID download utility. The [excludes file](https://raw.githubusercontent.com/nextstrain/ncov/master/config/exclude.txt) format used in [NextStrain's COVID-19 portal](https://nextstrain.org/ncov) is compatible with Poke. If you have nothing to exclude, specify ```/dev/null``` as the excludes file.

## Output
This software optimizes the input parameters to BLASTN then parses the BLAST output, outputting only relevant mismatch information. Several post-processing steps are included in this script 1) to not report perfect full query matches, 2) to help deal with quirks introduced by IUPAC ambiguity codes in the genome data (common in SARS-CoV-2 genomes to date), and 3) give contextual information for off target partial gene matching. This can greatly reduce the amount of data that the user needs to sift through to check for genuine primer/probe mismatches, and spares them from digging through the original genomes FastA file for mismatch context information. 

For example, with a given set of 6 oligonucleotide queries targeting the E and RdRP genes of SARS-CoV-2, about 200/30000 genomes with mismatches are detected. In the raw BLAST output there are more than 600 imperfect matches found due to ambiguity codes in the genomes either truncating alignments or introducing mismatch characters. For example, Poke will *NOT* report partial alignments like this, since they are due to ambiguities in the genome:

```text
===hCoV-19/England/CAMB-741E6/2020|EPI_ISL_440338|2020-03-20|Europe===

Mismatch against COVID19_RdRP_For with BLAST exact matches: 21/25
1      TTTTAACATTTGTCAAGCTGTcacg 21
       |||||||||||||||||||||
15465  TTTTAACATTTGTCAAGCTGTnnnn 15485

Mismatch against COVID19_RdRP_Prb with BLAST exact matches: 19/23
5      cactTTTATCTACTGATGGTAAC 23
           |||||||||||||||||||
15507  nnnnTTTATCTACTGATGGTAAC 15525
```

But the following partial alignment will be reported, with Poke adding lower case bases to the 3' end of the BLAST alignment so the mismatch is immediately visible:

```text
===hCoV-19/England/BRIS-123C3F/2020|EPI_ISL_442779|2020-04-07|Europe===

Mismatch against COVID19_RdRP_For with BLAST exact matches: 24/25
1      TTTTAACATTTGTCAAGCTGTCACg 24
       ||||||||||||||||||||||||
15517  TTTTAACATTTGTCAAGCTGTCACt 15540
```

Poke will *NOT* report a BLAST mismatch where the ambiguity code is a valid match:

```text
===hCoV-19/England/CAMB-72B35/2020|EPI_ISL_440003|2020-03-23|Europe===

Mismatch against COVID19_RdRP_Rev with BLAST exact matches: 25/26
1      GTTGTAAATTGCGGACATACTTATCG 26
       ||||||||||||||||| ||||||||
15556  GTTGTAAATTGCGGACAWACTTATCG 15531
```

But will report genuine mismatches:

```text
===hCoV-19/England/CAMB-1AECB4/2020|EPI_ISL_448069|2020-05-06|Europe===

Mismatch against COVID19_E_For with BLAST exact matches: 27/28
1      GAGACAGGTACGTTAATAGTTAATAGCG 28
       |||| |||||||||||||||||||||||
26266  GAGATAGGTACGTTAATAGTTAATAGCG 26293
```

An additional 45 genomes have no match to at least one of the six query oligos. This information is not easily discernable from the raw BLAST output, but in the Poke output such misses are designated with "NO MATCH". Is the lack of match due to changes in the genome, or was consensus not called in that part of the genome (e.g. due to low sequence coverage)? In the following example output, we see that an RdRP assay oligo matches position 15465 in ```England/CAMB-72E3C/2020``` (this position varies up to a few hundred bases from sample to sample in extant SARS-CoV-2 genomes). 

```text
===hCoV-19/England/CAMB-72E3C/2020|EPI_ISL_440122|2020-03-23|Europe===
Mismatch against COVID19_RdRP_For with BLAST exact matches: 24/25
1      TTTTAACATTTGTCAAGCTGTCACG 25
       ||||||||||||||||||||| |||
15465  TTTTAACATTTGTCAAGCTGTTACG 15489

===hCoV-19/England/CAMB-74043/2020|EPI_ISL_440320|2020-03-18|Europe===
Template length 29782, N stretches: [4635,4649], [4963,4984], [13930,13951], [14245,14272], [14547,14568], [14872,14899], [15192,15213], [15506,15526], [15832,15856], [16155,16177], [18618,18639], [18925,18946], [19341,19460], [26859,26881], [27161,27173], [27538,27618]
NO MATCH FOUND for COVID19_RdRP_For (length 25)
NO MATCH FOUND for COVID19_RdRP_Prb (length 23)
```

"NO MATCH" was found for two of the RdRP oligos in the next sample ```England/CAMB-74043/2020```, but the "Template length" line preceding it provides information that may be useful in determining why: there are large stretches of N's in that genome around RdRP, including `[15506,15526]` which is the likely culprit, rather than a genuine extreme genome consensus mismatch not detectable by BLASTN.

In some cases, rather than NO MATCH, the best match reported is a very poor match against a non-target part of the genome.
For example:

```
===hCoV-19/England/CAMB-7959A/2020|EPI_ISL_441189|2020-04-03|Europe===
Template length 29782, N stretches: [7278,7517], [19516,19810], [20306,20434], [20442,20564], [21181,21246], [26413,26589], [27754,28050]

Mismatch against COVID19_E_Rev with BLAST exact matches: 9/23
8    caatattGCAGCAGTAcgcacac 16
            |||||||||
904  actgctgGCAGCAGTAactgctg 896
```

The E gene is located around position 26300 in the genome. Having the bet match being only 9 bases starting at position 904 in the genome is [fitting a square peg into a round hole](https://www.collinsdictionary.com/dictionary/english/square-peg-in-a-round-hole), made all the more obvious by Poke providing the extended context of the mismatch to include the whole length of the query. The N stretches information shows us that there is a long stretch of N's around 26400. This may be the cause of the NO MATCH, but may require further investigation.

## Acknowledgements
Dr. Gordon is a co-investigator on COVID-19-related grant funding from [CIHR](https://cihr-irsc.gc.ca/e/51868.html) and [Genome Alberta](https://genomealberta.ca/funding/funding_blog_04032001.aspx). 
