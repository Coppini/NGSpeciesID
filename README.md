NGSpeciesID
===========

NGSpeciesID is a tool for clustering and consensus forming of targeted ONT reads. This repository is a modified version of [isONclust](https://github.com/ksahlin/isONclust), where consensus and polishing feautures have been added.

NGSpeciesID is distributed as a python package supported on Linux / OSX with python v3.6. [![Build Status](https://travis-ci.org/ksahlin/NGSpeciesID.svg?branch=master)](https://travis-ci.org/ksahlin/NGSpeciesID).

Table of Contents
=================

  * [INSTALLATION](#INSTALLATION)
    * [Using conda](#Using-conda)
    * [Testing installation](#testing-installation)
  * [USAGE](#USAGE)
    * [Filtering and subsampling](#filtering-and-subsampling)
    * [Removing primers](#removing-primers)
    * [Output](#Output)
  * [EXAMPLE WORKFLOW](#EXAMPLE-WORKFLOW)
  * [CREDITS](#CREDITS)
  * [LICENCE](#LICENCE)



INSTALLATION
----------------

**NOTE**: If you are experiencing issues (e.g. [this one](https://github.com/rvaser/spoa/issues/26)) with the third party tools  [spoa](https://github.com/rvaser/spoa) or [medaka](https://github.com/nanoporetech/medaka) in the all-in-one installation instructions below, please install the tools manually with their respective installation instructions [here](https://github.com/rvaser/spoa#installation) and [here](https://github.com/nanoporetech/medaka#installation).  

### Using conda
Conda is the preferred way to install NGSpeciesID.

1. Create and activate a new environment called NGSpeciesID

```
conda create -n NGSpeciesID python=3.6 pip 
conda activate NGSpeciesID
```

2. Install NGSpeciesID 

```
pip install NGSpeciesID
conda install --yes -c conda-forge -c bioconda medaka==0.11.5 openblas==0.3.3 spoa racon minimap2
```
3. You should now have 'NGSpeciesID' installed; try it:
```
NGSpeciesID --help
```

Upon start/login to your server/computer you need to activate the conda environment "NGSpeciesID" to run NGSpeciesID as:
```
conda activate NGSpeciesID
```



### Testing installation

0. Activate conda environment
```
conda activate NGSpeciesID
```

1. Make a new directory and navigate to it
```
mkdir test_ngspeciesID
cd test_ngspeciesID
```

2. Download the test fastq file called "sample_h1.fastq" (filesize 390kb)

```
curl -LO https://raw.githubusercontent.com/ksahlin/NGSpeciesID/master/test/sample_h1.fastq
```

3. Run the NGSpecies command on test file. Outputs will be saved in "/test_ngspeciesID/sample_h1/", where the final polished consensus file ("consensus.fasta") is located in the "/test_ngspeciesID/sample_h1/medaka_cl_id_<cluster number>" directory.

```
NGSpeciesID --ont --fastq sample_h1.fastq --outfolder ./sample_h1 --consensus --medaka
```


USAGE
-------

NGSpeciesID needs a fastq file generated by an Oxford Nanopore basecaller.

```
NGSpeciesID --ont --consensus --medaka --fastq [reads.fastq] --outfolder [/path/to/output] 
```
The argument `--ont` simply means `--k 13 --w 20`. These arguments can be set manually without the `--ont` flag. Specify number of cores with `--t`. 


NGSpeciesID can also run with racon as polisher. For example

```
NGSpeciesID --ont --consensus --racon --racon_iter 3 --fastq [reads.fastq] --outfolder [/path/to/output] 
```
will polish the consensus sequences with racon three times.

### Filtering and subsampling

NGSpeciesID employs quality filtering of the reads based on read Phred scores. However, we recommend also removing reads much shorter or longer than the intended target, which often represent chimeras or contaminations. This can be done by specifying the `--m (intended target length)` and `--s (maximum deviation from target length)`. NGSpeciesID also has the feature of subsampling reads using parameter `--sample_size`. Altogether, if we want to filter out reads outside the length interval [700,800] and using a subset of 300 reads (if the dataset consists of more reads) we could run

```
NGSpeciesID --ont --sample_size 300 --m 750 --s 50 --consensus --medaka --fastq [reads.fastq] --outfolder [/path/to/output]
```

By default, length filtering and subsampling are not invoked if parameters are not specified.

### Removing primers

If customized primers are to be expected in the reads thay can be detected and removed. The primer file is expected to be in fasta format. Here is an example of a primer file:

```
>MCB869_ONT_R
CGATCAATCCCCTAACAAACTAGG
>MCB398_ONT_F
TACCATGAGGACAAATATCATTCTG
```
NGSpeciesID searches for primes in a window of Xbp (parameter, default 150bp) at the beginning and end of each consensus.


Trimming of primers is performed after consensus forming and can be invoked as
```
NGSpeciesID --ont --consensus --medaka --fastq [reads.fastq] --outfolder [/path/to/output] --primer_file [primers.fa]
```

`NGSpeciesID` can also remove universal tails. Trimming of tails is performed after consensus forming and can be invoked as

```
NGSpeciesID --ont --consensus --medaka --fastq [reads.fastq] --outfolder [/path/to/output] --remove_universal_tails
```

The two options are mutually exclusive, i.e., only one of them can be run.

### Output

The output consists of the polished consensus sequences along with some information about clustering.

* Polished consensus sequence(s). A folder named “medaka_cl_id_X”[/"racon_cl_id_X"] is created for each predicted consensus. Each such folder contains a sequence “consensus.fasta” which is the final output of NGSpeciesID. 
* Draft spoa consensus sequences of each of the clusters are given as consensus_reference_X.fasta (where X is a number).
* The final cluster information is given in a tsv file `final_clusters.tsv` present in the specified output folder.


In the cluster TSV-file, the first column is the cluster ID and the second column is the read accession. For example:

```
0 read_X_acc
0 read_Y_acc
...
n read_Z_acc
```
if there are n reads there will be n rows. Some reads might be singletons. The rows are ordered with respect to the size of the cluster (largest first).


EXAMPLE WORKFLOW
-----------------

The bioinformatics workflow below was developed as part of a step-by-step protocol for field-deployable DNA amplicon sequencing with the Oxford Nanopore Technologies MinION. The full protocol manuscript is in submission; a link will be posted here when available. The steps below correspond to step numbers in the protocol.

#### P2 | Generate custom indexes for uniquely identifying samples using [`barcode_generator`](https://github.com/lcomai/barcode_generator). This software uses Python3.

```
python3 barcode_generator_3.4.py none 24 40 8
```

Here, the parameters are set as:
- table_excluded_barcodes = 'none'
- index length = 24 base pairs
- number of barcodes to generate = 40
- hamming distance = 8

After lab steps are complete:

#### B1 | Basecalling and quality check (optional) with [Guppy](https://community.nanoporetech.com/downloads)

Basecalling:

```
guppy_basecaller --input_path minKNOW_input/ --save_path basecalled_fastqs/ -c dna_r9.4.1_450bps_fast.cfg --recursive --disable_pings
```

Basecalling and filter reads by quality score (here, set to 7):

```
guppy_basecaller --input_path minKNOW_input/ --save_path basecalled_fastqs/ -c dna_r9.4.1_450bps_fast.cfg --recursive --disable_pings --min_qscore 7
```
 
#### B2 | Go to folder with the fastq files generated by Guppy

#### B3 | Concatenate all the read files into one large file

```
cat *.fastq > sequencing_reads.fastq
```

#### B4 | Check raw read quality/stats with [NanoPlot](https://github.com/wdecoster/NanoPlot)

```
NanoPlot --fastq_rich sequencing_reads.fastq -o sequencing_run -p sequencing_run
```

#### B5 | Demultiplexing of the sequencing data with [minibar](https://github.com/calacademy-research/minibar) or Guppy

Example files can be found in:
- [Supplementary Data 2](./test/Supplementary_File2_reads.fastq): 3,000 reads in fastq format from three fish species - Atlantic cod (*Gadus morhua*), Haddock (*Melanogrammus aeglefinus*), and Whiting (*Merlangius merlangus*) - sequenced on a Flongle flow cell.
- [Supplementary Data 3](./test/Supplementary_File3_minibar.txt): index file used for demultiplexing with minibar

#### B5a | minibar (using example files):

```
python minibar.py Supplementary_File3_minibar.txt Supplementary_File2_reads.fastq -T -F -e 3 -E 11
```

Here, the edit distance allowed between indexes (`-e`) is set to 3 base pairs and the edit distance allowed between primer sequences (`-E`) is set to 11 base pairs.

#### B5b | Guppy:

```
guppy_barcoder -i Supplementary_File2_reads.fastq -s demultiplex_folder --trim_barcodes --disable_pings
```

#### B6 | Read filtering, clustering, consensus generation and polishing with NGSpeciesID

For a single sample (using example primer file):

```
NGSpeciesID --ont --consensus --sample_size 500 --m 800 --s 100 --medaka --primer_file Supplementary_File4_primer.txt --fastq barcode0.fastq --outfolder barcode0_consensus
```

Here, the parameters are set as:
- the data is from ONT MinION (`--ont`)
- we want to generate consensus sequences (`--consensus`)
- subsample of reads (`--sample_size`) = 500 reads subsampled per sample to analyze 
- intended target length (`--m`) = 800 base pairs
- maximum deviation from target length (`--s`) = 100 base pairs
- use [Medaka](https://github.com/nanoporetech/medaka) to polish the final consensus sequences (`--medaka`)
- if a `--primer_file` is given, NGSpeciesID will check to remove any remaining primer sequence. The example primer file is available in [Supplementary Data 4](./test/Supplementary_File4_primer.txt). The primers were developed in Mikkelsen, P.M., Bieler, R., Kappner, I., & Rawlings, T.A. (2006). Phylogeny of Veneroidea (Mollusca: Bivalvia) based on morphology and molecules. *Zoological Journal of the Linnean Society*, 148(3), 439-521.
- the input file of demultiplexed reads is specified by `--fastq` (output from step B5)
- the output consensus files will be saved to `--outfolder`

To run this step on **more than one sample**, use a bash script with a for loop:

```
for all files in *.fastq; do
  bn = `basename $file .fastq`
  
  NGSpeciesID --ont --consensus --sample_size 500 --m 800 --s 100 --medaka --primer_file Supplementary_File4_primer.txt --fastq $file --outfolder ${bn}

done
```

This loop uses the wildcard `*` to indicate you want to analyze all files with the `.fastq` extension and assumes the command is run from the directory that contains the read files (if not, be sure to change the file path: `path/to/*.fastq`). 

This loop code can be entered at a UNIX/Mac terminal (be sure the spacing/indentation is correct) or saved as a script (see [`consensus.sh`](./test/consensus.sh). The script should be run from the terminal and in the directory that contains the read files as:

```
./consensus.sh
```

#### B7 | Compare consensus sequences to reference database with [BLAST](https://blast.ncbi.nlm.nih.gov/Blast.cgi?PAGE_TYPE=BlastDocs&DOC_TYPE=Download)

- Create/format database for BLAST search:

```
makeblastdb -in database.fasta -dbtype nucl -out database
```

- Conduct BLAST search:

```
blastn -db database -query barcode0_consensus.fasta -outfmt 6 -out barcode0_consensus_blast.out
```

Check the results and refine the search or database as needed to better identify the sequence identity of your samples!



CREDITS
----------------

Please cite [1] when using NGSpeciesID.

1. TBA



LICENCE
----------------

GPL v3.0, see [LICENSE.txt](https://github.com/ksahlin/NGSpeciesID/blob/master/LICENCE.txt).


