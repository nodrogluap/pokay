# Pokay
Report salient DNA (qPCR primer/probe) or amino acid (epitope) mismatches info against a set of pathogen isolate genomes.

## Pronunciation
/poʊˈkeɪ/ as in the [delicious Hawaiian raw fish dish](https://en.wikipedia.org/wiki/Poke_(Hawaiian_dish)), or Pokay as in "my PCR design is okay" or "my epitope is okay".

## Motivation
Does my PCR assay design detect all known sequenced examples of a pathogen? Or has the pathogen mutated to escape immune detection by epitope mismatches?

This lighweight tools was developed to simplify the process of checking and reporting published SARS-CoV-2 genomes for subsequences you expect to be stable, but actually have evolved variants. Considerable effort has gone into handling the many ambiguous bases and no-coverage (N) regions in [GISAID](https://gisaid.org/CoV2020)-deposited genomes.

Use Case 1: continued detection sensitivity of PCR assays in light of available pathogen genomic data. As of this writing the genomes of more than 115,000 SARS-CoV-2 samples are available in GISAID and [at the NCBI](https://www.ncbi.nlm.nih.gov/genbank/sars-cov-2-seqs/) thanks to global sequencing efforts, and a lab may want to check if their PCR assay's primers/probes continue to be exact matches against all samples despite inevitable viral evolution. Pokay takes about 3 minutes to run, and complements existing efforts to track mismatch counts for published SARS-CoV-2 PCR assays, such as the very useful [EDGE validation portal](https://covid19.edgebioinformatics.org/#/assayValidation), and [PCR_strainer](https://github.com/KevinKuchinski/PCR_strainer) which suggests filtering out genomes with base ambiguities.

Use Case 2: continued effectiveness of post-infection or post-vaccination immunity in light of available pathogen genomic data. As of this writing there are 150 experimentally-determined epitopes for the SARS-CoV-2 spike protein (the target of most vaccines that are in the works) as aggregated in [IEDB](http://www.iedb.org/sourceOrgId/2697049), and 799 overall for the virus. Pokay takes about 2 hours to report all spike protein epitope mismatches, handling both linear and discontinuous epitopes (requiring special alignment), and optimistic interpretation of ambiguous genome bases. 

## Quick Start
This script depends on you having a preinstalled version of [NCBI BLAST](https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/LATEST/), and has only been tested using version 2.9.0+. It can be downloaded from the provided link, or if you are running the [conda manager](https://docs.conda.io/projects/conda/en/latest/user-guide/install/), this should be as easy as ```conda install blast```. It also requires any version of Perl (there are no module dependencies).

Here file_of_genomes.fasta might be downloaded from GISAID. Here ```file_of_genome_ids_to_exclude.txt``` allows you to remove duplicate or dodgy genomes from the analysis without having to edit the genomes file that you may have downloaded in bulk from somewhere like the GISAID download utility. The [excludes file](https://raw.githubusercontent.com/nextstrain/ncov/master/config/exclude.txt) format used in [NextStrain's COVID-19 portal](https://nextstrain.org/ncov) is compatible with Pokay. If you have nothing to exclude, specify ```/dev/null``` as the excludes file.

### PCR design

```shell
pokay file_of_genomes.fasta file_of_genome_ids_to_exclude.txt pcr_primers_and_or_probes.fasta > salient_mismatch_info_output.txt
```

### Epitope immunity escape
First, download the epitopes of interest from IEDB as a CSV file.

```shell
./iedb_csv2fasta epitope_table_export.csv epitopes.fasta
pokay file_of_genomes.fasta file_of_genome_ids_to_exclude.txt epitopes.fasta > salient_mismatch_info_output.txt
```

## Output
This software optimizes the input parameters to BLASTN then parses the BLAST output, outputting only relevant mismatch information. Several post-processing steps are included in this script 1) to not report perfect full query matches, 2) to help deal with quirks introduced by IUPAC ambiguity codes in the genome data (common in SARS-CoV-2 genomes to date), and 3) give contextual information for off target partial gene matching. This can greatly reduce the amount of data that the user needs to sift through to check for genuine primer/probe mismatches, and spares them from digging through the original genomes FastA file for mismatch context information. 

### PCR Design
For example, with a given set of 6 oligonucleotide queries targeting the E and RdRP genes of SARS-CoV-2, about 200/30000 genomes with mismatches are detected. In the raw BLAST output there are more than 600 imperfect matches found due to ambiguity codes in the genomes either truncating alignments or introducing mismatch characters. For example, Pokay will *NOT* report partial alignments like this, since they are due to ambiguities in the genome:

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

But the following partial alignment will be reported, with Pokay adding lower case bases to the 3' end of the BLAST alignment so the mismatch is immediately visible:

```text
===hCoV-19/England/BRIS-123C3F/2020|EPI_ISL_442779|2020-04-07|Europe===

Alignment against COVID19_RdRP_For with mismatches: 24/25
1      TTTTAACATTTGTCAAGCTGTCACg 24
       ||||||||||||||||||||||||
15517  TTTTAACATTTGTCAAGCTGTCACt 15540
```

Pokay will *NOT* report a BLAST mismatch where the ambiguity code is a valid match:

```text
===hCoV-19/England/CAMB-72B35/2020|EPI_ISL_440003|2020-03-23|Europe===

Alignment against COVID19_RdRP_Rev with mismatches: 25/26
1      GTTGTAAATTGCGGACATACTTATCG 26
       ||||||||||||||||| ||||||||
15556  GTTGTAAATTGCGGACAWACTTATCG 15531
```

But will report genuine mismatches:

```text
===hCoV-19/England/CAMB-1AECB4/2020|EPI_ISL_448069|2020-05-06|Europe===

Alignment against COVID19_E_For with mismatches: 27/28
1      GAGACAGGTACGTTAATAGTTAATAGCG 28
       |||| |||||||||||||||||||||||
26266  GAGATAGGTACGTTAATAGTTAATAGCG 26293
```

An additional 45 genomes have no match to at least one of the six query oligos. This information is not easily discernable from the raw BLAST output, but in the Pokay output such misses are designated with "NO MATCH". Is the lack of match due to changes in the genome, or was consensus not called in that part of the genome (e.g. due to low sequence coverage)? In the following example output, we see that an RdRP assay oligo matches position 15465 in ```England/CAMB-72E3C/2020``` (this position varies up to a few hundred bases from sample to sample in extant SARS-CoV-2 genomes). 

```text
===hCoV-19/England/CAMB-72E3C/2020|EPI_ISL_440122|2020-03-23|Europe===
Alignment against COVID19_RdRP_For with mismatches: 1/25
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

Alignment against COVID19_E_Rev with mismatches: 12/23
8    caatattGCAGCAGTAcgcacac 16
            |||||||||
904  actgctgGCAGCAGTAactgctg 896
```

The E gene is located around position 26300 in the genome. Having the best match being only 9 bases starting at position 904 in the genome is [fitting a square peg into a round hole](https://www.collinsdictionary.com/dictionary/english/square-peg-in-a-round-hole), made all the more obvious by Pokay providing the extended context of the mismatch to include the whole length of the query. The N stretches information shows us that there is a long stretch of N's around 26400. This may be the cause of the NO MATCH, but may require further investigation.

### Epitope immunity escape

Results for protein sequences are fixed by Pokay for the same issues as noted above for PCR DNA sequences (ambiguity codes, end-effects), but also have special formatting to show both the nucleotides and AA translation of subject genomes. For example, the display of epitope mismatches for a [re-infection case of SARS-CoV-2](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3680955):

```text
===USA/NV-NSPHL-A0207/2020===
Alignment against <http://www.iedb.org/epitope/1074860> with mismatches: 1/38
1       A  T  I  P  I  Q  A  S  L  P  F  G  W  L  I  V  G  V  A  L  L  A  V  F  Q  S  A  S  K  I  I  T  L  K  K  R  W  Q    38/38
        A  T  I  P  I  Q  A  S  L  P  F  G  W  L  I  V  G  V  A  L  L  A  V  F  +  S  A  S  K  I  I  T  L  K  K  R  W  Q 
25392  GCAACGATACCGATACAAGCCTCACTCCCTTTCGGATGGCTTATTGTTGGCGTTGCACTTCTTGCTGTTTTTCATAGCGCTTCCAAAATCATAACCCTCAAAAAGAGATGGCAA 25505/25505
                                                                                H                                        

Alignment against <http://www.iedb.org/epitope/1074897> with mismatches: 3/14
1      F  T  E  Q  P  I  D  L  V  P  N  Q  p  y  12/14
       F  T  E  Q  P  I  D  L  V  P  N  +       
5886  TTCACAGAGCAACCAATTGATCTTGTACCAAACCACaccatn 5921/5927
                                        H  T    
```

In the first alignment, a glutamine (Q) has been substituted for a histidine (H), conservatively (+) according to the PAM30 matrix used in the TBLASTN alignment. In the second alignment, a similar substitution happened in another part of the genome (~5.9k), but the adjacent codon is a non-conservative P -> T, and the ambiguous N base in the next codon cannot be resolved with any of A, C, G or T to make a tyrosine (Y) codon. Both of the two last codons were added to the TBLASTN output by Pokay, hence the lower case DNA bases (the 12/14 and 5921/5927 end coordinates also indicate this two codon extension). Amino acid translations of the subject are never shown in lower case, but query AAs are.

Pokay also handles alignment of discontinuous epitopes (where the pathogen protein's amino acids that bind the antibody are not linear, but rather near each other in the protein's 3D structure). In this case, the query is defined by placing X's between the otherwise linear AA sequnece residues that matter (this is what the included script iedb_csv2fasta does).

In the following example, the X's do not contribute to the mismatch count, nor do the stretches of n's in the subject genome. The mismatch that is counted is in the upstream Pokay extension of the original TBLASTN alignment, which sees a tyrosine (Y) changed to a stop codon (indicated as "TER" for termination). "..." has been included here for brevity, the full AA translation of the subject through the discontinuous epitope X's is provided so that quality (e.g. more TER codons) can be assessed by the user.

```text
===Australia/VIC1724/2020===
Alignment against <http://www.iedb.org/epitope/997006> with mismatches: 1/149
2       y  N  S  A  X  F  X  X  F  K  C  Y  G  V  S  P  T  K  X  X  X  L  x  x  x  x  ... x  f  t  x  ...  x  f  e  l  22/149
           N  S  A              F  K  C  Y  G  V  S  P  T  K           L                     F  T                                                                 
22329  taaAATTCCGCATCANNNTCCACTTTTAAGTGTTATGGAGTGTCTCCTACTAAATTAAATGATCTCtgctttactaatg...gattttacaggc ... nnnnnnnnnnnn 22391/22518
       TER          S     S  T                                L  N  D     C  F  T  N  ... D        G  
```
       
N.B. Insertions and deletions in the subject relative to the number of X's in discontinuous epitopes will also be noted as mismatches.

## Acknowledgements
Dr. Gordon is a co-investigator on COVID-19-related grant funding from [CIHR](https://cihr-irsc.gc.ca/e/51868.html) and [Genome Alberta](https://genomealberta.ca/funding/funding_blog_04032001.aspx). 
