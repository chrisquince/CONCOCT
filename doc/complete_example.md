Complete Example
================
This documentation page aims to be a complete example walk through for the usage of the CONCOCT package.


Required software
----------------------
To run the entire example you need the following software:

* Assembling Metagenomic Reads
    * Ray version >= 2.1.0
* Map the Reads onto the Contigs
    * BEDTools version >= 2.15.0 (only genomeCoverageBed)
    * Picard tools version >= 1.77
    * samtools version >= 0.1.18
    * bowtie2 version >= 2.1.0
    * parallel version >= 20130422

It is not required to run all steps. The output files for each step are in the test data repository. At the end of this example the results should be the same as the results in the test data repository: https://github.com/BinPro/CONCOCT-test-data.

Downloading test data
-----------------------
Clone the test data repository of CONCOCT:

    git clone https://github.com/BinPro/CONCOCT-test-data
    
It contains the reads that we start this example with as well as the output from each step. It could take a while to download.

Assembling Metagenomic Reads
----------------------------
After obtaining the test data, create a folder where you want all the output from this example to go:

    mkdir CONCOCT-complete-example
    cd CONCOCT-complete-example

Set three variables with full paths. One pointing to the root directory of ```CONCOCT```, one pointing to the test data ```CONCOCT_TEST``` and one to the directory we just created e.g.

    CONCOCT=/home/username/src/CONCOCT
    CONCOCT_TEST=/home/username/src/CONCOCT-test-data
    CONCOCT_EXAMPLE=/home/username/CONCOCT-complete-example

Change the paths to the actual locations where we downloaded ```CONCOCT``` and ```CONCOCT-test-data```. After that run Ray on the reads:

    cd $CONCOCT_EXAMPLE
    mpiexec -n 1 Ray -k 31 -o ray_output_31 \
        -p $CONCOCT_TEST/reads/Sample118_s1e5_R1.fa $CONCOCT_TEST/reads/Sample118_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample120_s1e5_R1.fa $CONCOCT_TEST/reads/Sample120_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample127_s1e5_R1.fa $CONCOCT_TEST/reads/Sample127_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample134_s1e5_R1.fa $CONCOCT_TEST/reads/Sample134_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample177_s1e5_R1.fa $CONCOCT_TEST/reads/Sample177_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample215_s1e5_R1.fa $CONCOCT_TEST/reads/Sample215_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample230_s1e5_R1.fa $CONCOCT_TEST/reads/Sample230_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample234_s1e5_R1.fa $CONCOCT_TEST/reads/Sample234_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample244_s1e5_R1.fa $CONCOCT_TEST/reads/Sample244_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample261_s1e5_R1.fa $CONCOCT_TEST/reads/Sample261_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample263_s1e5_R1.fa $CONCOCT_TEST/reads/Sample263_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample290_s1e5_R1.fa $CONCOCT_TEST/reads/Sample290_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample302_s1e5_R1.fa $CONCOCT_TEST/reads/Sample302_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample321_s1e5_R1.fa $CONCOCT_TEST/reads/Sample321_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample330_s1e5_R1.fa $CONCOCT_TEST/reads/Sample330_s1e5_R2.fa \
        -p $CONCOCT_TEST/reads/Sample343_s1e5_R1.fa $CONCOCT_TEST/reads/Sample343_s1e5_R2.fa

After the assembly is finished create a directory with the resulting contigs and copy the result of Ray there (this output is also in ```$CONCOCT_TEST/contigs```):

    mkdir contigs
    cp ray_output_31/Contigs.fasta contigs/raynoscaf_31.fa
    

Map the Reads onto the Contigs
------------------------------
After assembly we map the reads of each sample back to the assembly using bowtie2 and remove PCR duplicates with MarkDuplicates. The coverage histogram for each bam file is computed with BEDTools' genomeCoverageBed. The script that calls these programs is provided with CONCOCT. All one needs to do is set the environment variable for MarkDuplicates ```$MRKDUP``` which should point to the MarkDuplicates jar file e.g.

    export MRKDUP=/home/username/src/picard-tools-1.77/MarkDuplicates.jar

The following command is to be executed in the ```$CONCOCT_EXAMPLE``` dir you created in the previous part. First create the index on the assembly for bowtie2:

    cd $CONCOCT_EXAMPLE
    bowtie2-build contigs/raynoscaf_31.fa contigs/raynoscaf_31.fa
    
Then create a folder map. The parallel command reates a folder for each sample, and runs ```map-bowtie2-markduplicates.sh``` for each sample:

    mkdir map
    parallel mkdir -p map/{/} '&&' \
        cd map/{/} '&&' \
        bash $CONCOCT/scripts/map-bowtie2-markduplicates.sh \
            -ct 1 -p '-f' {} '$('echo {} '|' sed s/R1/R2/')' pair \
            ../../contigs/raynoscaf_31.fa asm bowtie2 \
        ::: $CONCOCT_TEST/reads/*_R1.fa

The parameters used for `map-bowtie2-markduplicates.sh` are:

* `-c` option to compute coverage histogram with genomeCoverageBed
* `-t` option is number of threads (1 since we are already running the ```map-bowtie2-markduplicates.sh``` script in parallel.
* `-p` option is the extra parameters given to bowtie2. In this case `-f`.

The five arguments are:
* pair1, the fasta/fastq file with the #1 mates
* pair2, the fasta/fastq file with the #2 mates
* pair_name, a name for the pair used to prefix output files
* assembly, a fasta file of the assembly to map the pairs to
* assembly_name, a name for the assembly, used to postfix outputfiles
* outputfolder, the output files will end up in this folder

Generate coverage table
------------------------
Use the bam files of each sample to create a table with the coverage of each contig per sample.

    cd $CONCOCT_EXAMPLE/map
    python $CONCOCT/scripts/gen_input_table.py --isbedfiles \
        --samplenames <(for s in Sample*; do echo $s | cut -d'_' -f1; done) \
        ../contigs/raynoscaf_31.fa */bowtie2/asm_pair-smds.coverage \
    > concoct_inputtable.tsv
    mkdir $CONCOCT_EXAMPLE/concoct-input
    mv concoct_inputtable.tsv $CONCOCT_EXAMPLE/concoct-input/

Generate linkage table
------------------------
The same bam files can be used to give linkage per sample between contigs:

    cd $CONCOCT_EXAMPLE/map
    python $CONCOCT/scripts/bam_to_linkage.py -m 8 \
        --regionlength 500 --fullsearch \
        --samplenames <(for s in Sample*; do echo $s | cut -d'_' -f1; done) \
        ../contigs/raynoscaf_31.fa Sample*/bowtie2/asm_pair-smds.bam \
    > concoct_linkage.tsv
    mv concoct_linkage.tsv $CONCOCT_EXAMPLE/concoct-input/
    

Run concoct
-----------
To see possible parameter settings with a description run

    concoct --help

We will only run concoct for some standard settings here. First we need to parse the input table to just contain the mean coverage for each contig in each sample:

    cut -f1,11-26 concoct-input/concoct_inputtable.tsv > concoct-input/concoct_inputtableR.tsv

Then run concoct on 8 cores `-m 8` over a range of cluster sizes `-c 2,16,1`, from two to sixteen with a step size of one, that we guess is appropriate for this data set:

    concoct -m 8 -c 2,16,1 --coverage_file concoct-input/concoct_inputtableR.tsv --composition_file contigs/raynoscaf_31.fa -b concoct-output

In practice, you can chose the cluster range based on taxonomic classification or contigs but it is easy to determine if the optimum value lies outside the initial guess. When concoct has finished the message "CONCOCT Finished, the log shows how it went." is piped to stdout. The program generates a number of files in the output directory, this can be set with the `-b` command or is the present working directory by default. 

Evaluate output
---------------

First we can examine the model selection based on BIC by running:

    BICPlot.R -b concoct-output/bic.csv -o evaluation-output/bic.pdf

This will require that you have added the CONCOCT/script directory to your path and have Rscript with the R packages gplots, reshape, ggplot2, ellipse, getopt and grid installed. This generates a plot of the BIC as a function of the cluster number K. The best model minimises K. In this case that occurs for K = 4:

<https://github.com/BinPro/CONCOCT-test-data/tree/master/evaluation-output/bic.pdf>

Then we can visualise the clusters in the first two PCA dimensions:

    ClusterPlot.R -c concoct-output/clustering_gt1000.csv -p concoct-output/PCA_transformed_data_gt1000.csv -m concoct-output/pca_means_gt1000.csv -r concoct-output/pca_variances_gt1000_dim -l -o evaluation-output/ClusterPlot.pdf

<https://github.com/BinPro/CONCOCT-test-data/tree/master/evaluation-output/ClusterPlot.pdf>

We can also compare the clustering to species labels. For this test data set we know these labels, they are given in the file 'clustering_gt1000_s.csv'. For real data labels may be obtained through taxonomic classification, e.g. using:

<https://github.com/umerijaz/TAXAassign>

In either case we provide a script Validate.pl for computing basic metrics on the cluster quality:

    Validate.pl --cfile=concoct-output/clustering_gt1000.csv --sfile=evaluation-output/clustering_gt1000_s.csv --ofile=evaluation-output/clustering_gt1000_conf.csv

This script requires the clustering output by concoct `concoct-output/clustering_gt1000.csv` these have a simple format of a comma separated file listing each contig id followed by the cluster index and the species labels that have the same format but with a text label rather than a cluster index. The script should output:

N	M	S	K	Rec.	Prec.	NMI	Rand	AdjRand  
443	443	6	4	0.920993	0.963883	0.855371	0.933097	0.850947  

This gives the no. of contigs N clustered, the number with labels M, the number of unique labels S, the number of clusters K, the recall, the precision, the normalised mutual information (NMI), the Rand index, and the adjusted Rand index. It also generates a file called a `confusion matrix` with the frequencies of each species in each cluster. We provide a further script for visualising this as a heatmap:

    ConfPlot.R  -c evaluation-output/clustering_gt1000_conf.csv -o  evaluation-output/clustering_gt1000_conf.pdf

This generates a file with normalised frequencies of contigs from each cluster across species:

<https://github.com/BinPro/CONCOCT-test-data/tree/master/evaluation-output/clustering_gt1000_conf.pdf>

Validation using single-copy core genes
---------------------------------------

We can also evaluate the clustering based on single-copy core genes. You first need to find genes on the contigs and functionally annotate these. Here we used prodigal (http://prodigal.ornl.gov/) for gene prediction, the corresponding protein sequences are here:

<https://github.com/BinPro/CONCOCT-test-data/tree/master/evaluation-output/annotations/proteins/raynoscaf_31.faa>

And we used WebMGA (http://weizhong-lab.ucsd.edu/metagenomic-analysis/) to COG annotate the protein sequences. The output file is here:

<https://github.com/BinPro/CONCOCT-test-data/tree/master/annotations/cog-annotations/raynoscaf_31.cog>

We can now run the script Validate_scg.pl to generate a table of counts of single-copy core genes in the different clusters generated by CONCOCT. 

    perl Validate_markers.pl -c output/clustering_gt1000.csv -a annotation/raynoscaf_31.cog -m scg/scg_cogs_min0.97_max1.03_unique_genera.txt –s “_” > evaluation-output/clustering_gt1000_scg.tab

The script requires the clustering output by concoct `concoct-output/clustering_gt1000.csv`, an annotation file `annotations/cog-annoations/cogs.txt`, a tab-separated file with contig ids in the first and gene ids in the second column (one contig can appear on multiple lines), and a file listing a set of SCGs (e.g. a set of COG ids) to use `scgs/scg_cogs_min0.97_max1.03_unique_genera.txt`.

The parameter –s indicates a separator character in which case only the string before (the first instance of) the separator will be used as contig id in the annotation file. We may want to do this because prodigal and other programs name the proteins on the same contig as contigXXXX_1, contigXXXX_2, etc.

The output file is a tab-separated file with basic information about the clusters (cluster id, ids of contigs in cluster and number of contigs in cluster) in the first three columns, and counts of the different SCGs in the following columns.

<https://github.com/BinPro/CONCOCT-test-data/tree/master/evaluation-output/clustering_gt1000_scg.tab>

This can also be visualised graphically using the R script. First we trim a little extraneous information from the tab files and then we generate the plot with an R script:

    cut -f1,4- evaluation-output/clustering_gt1000_scg.tab > evaluation-output/clustering_gt1000_scg.tsv
    COGPlot.R -s evaluation-output/clustering_gt1000_scg.tsv -o evaluation-output/clustering_gt1000_scg.pdf

The plot is downloadable here:

<https://github.com/BinPro/CONCOCT-test-data/tree/master/evaluation-output/clustering_gt1000_scg.pdf>
