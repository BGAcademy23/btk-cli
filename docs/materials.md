# Session Materials

## Slides

## Demo 1

[Launch Gitpod Workspace](https://gitpod.io/#https://github.com/bgacademy23/btk-cli){ : target="_blank" .md-button .md-button--primary }

When you launch the Gitpod workspace, these tasks have already been done for you:

1. **Conda/mamba** has been installed
2. **BlobToolKit (BTK) viewer and api docker containers** are already running with an example dataset
3. **/workspace/glClaSqua9** folder is ready with the assembly fasta file and mapped reads, blast hits, busco files, etc

### Overview

We are going to work with an initial PacBio HiFi genome assembly of a sample of **_Cladonia squamosa_**, the dragon cladonia lichen.

- **Explore the Gitpod interface**
- **Explore the BTK viewer**
- **Explore the input files**
- **Install BTK command line tools**
- **Create a blobdir**
- **Add read depth data to the blobdir**
- **Add blast hits to the blobdir**
- **View the blobdir in the BTK viewer**

### Details

- **Explore the Gitpod interface**
    - left: file browser
    - top right: preview window / editor
    - bottom right: linux terminal

- **Explore the BTK viewer**
    - the plot represents contigs in a genome assembly
    - each circle is a contig, the size is proportional to the length
    - the X axis is GC content of contig
    - the Y axis is the sequencing coverage or read depth of that contig
    - the colours are best blast hits 

- **Explore the input files**
```
cd /workspace/glClaSqua9
```
    Tip: use `less` to quickly check what large files contain

- **Install BTK command line tools**
```
mamba create -n btk -c conda-forge python=3.9 -y
mamba activate btk
pip install "blobtoolkit[full]"
```
    You should see a bash prompt beginning with `[btk]`, and if you type `blobtools` you should see some help text
```
blobtools
```
- **Create a blobdir**

    The first step is to create a yaml file with some very minimal information.
```
cd /workspace/glClaSqua9
code glClaSqua9.yaml
```
    In the text editor that opens top right, paste in this information and save it:
```
assembly:
  alias: glClaSqua9
  record_type: contig
taxon:
  name: Cladonia squamosa
  taxid: 174074
```
    Now, run this command to create a new folder: glClaSqua9_blobdir
```
blobtools create \
    --fasta glClaSqua9.fasta \
    --meta glClaSqua9.yaml \
    ./glClaSqua9_blobdir
```
    And take a look at the contents of glClaSqua9_blobdir

- **Add read depth data to the blobdir**

    To add the sequencing coverage or read depth for each contig from a bam file to the same blobdir:
```
blobtools add \
    --cov ./precomputed/mapped_reads/glClaSqua9.m64174e.bam \
    --threads 8 \
    ./glClaSqua9_blobdir
```

- **Add blast hits to the blobdir**

    We've now got the GC (X axis) and Cov (Y axis), so all we need is some way of colouring the contigs by their best hits to known databases. The precomputed/diamond folder has diamond blast hits in a tabular format, so we can add that.

    But this time we need one additional input - the NCBI taxonomy - because BTK needs it to extrapolate hits at different taxonomy levels (such as phylum, class, order, etc)
```
mkdir ./taxdump
cd    ./taxdump
wget ftp://ftp.ncbi.nih.gov/pub/taxonomy/new_taxdump/new_taxdump.tar.gz
tar -xzf new_taxdump.tar.gz
cd ../
```
    Now we can add the diamond blast hits, telling BTK to get additional taxonomy information from this folder:
```
blobtools add \
    --hits ./precomputed/diamond/glClaSqua9.diamond.busco_genes.out \
    --taxrule bestsumorder \
    --taxdump ./taxdump \
    ./glClaSqua9_blobdir
```

- **View the blobdir in the BTK viewer**

    Move the blobdir into a folder that has all the blobdirs:
```
cp -r glClaSqua9_blobdir /workspace/btk_example/src/data/example/
```
    Send an instruction to the BTK API to reload and reindex that folder:
```
curl $(gp url 8000)/api/v1/search/reload/testkey
```
    Refresh the browser top right, and click on the new dataset: glClaSqua9_blobdir. By default, BTK viewer shows binned plots if there are more than 2000 contigs, so to get contigs plotted as circles, click on _Settings > shape > circle_

## Exercise

Add the other diamond blast hit file in `./precomputed/diamond/glClaSqua9.diamond.reference_proteomes.out`. Hint: change the `blobtools add --hits` command above by changing the input filename, and also change the taxrule to `--taxrule bestsumorder_blastx`. If you don't change the taxrule name, blobtools will assume you are referring to the same 

??? note "Solution"
    ```
    cd /workspace/glClaSqua9

    blobtools add \
        --hits ./precomputed/diamond/glClaSqua9.diamond.reference_proteomes.out \
        --taxrule bestsumorder_blastx \
        --taxdump ./taxdump \
        ./glClaSqua9_blobdir
    
    cp -r glClaSqua9_blobdir /workspace/btk_example/src/data/example/
    
    curl $(gp url 8000)/api/v1/search/reload/testkey
    ```

## Demo 2

You can add any categorical value to a contig, it does not have to be a blast hit.

For example, the deep-learning based tool **[tiara](https://academic.oup.com/bioinformatics/article/38/2/344/6375939){ : target="_blank" }** can very rapidly estimate if a contig is a eukaryote, bacteria, or organelle.

So, to get a very quick annotation of a BTK plot, you could colour it using tiara. In this demo we install tiara, run tiara, and assign the tiara categories in BTK.

Install tiara:
```
mamba create -n tiara -c conda-forge tiara python=3.7 -y
mamba activate  tiara
```
Run tiara:
```
cd /workspace/glClaSqua9 
tiara -i glClaSqua9.fasta -o glClaSqua9.tiara -t 8 --pr --tf all -m 1000
mamba deactivate

blobtools add \
    --text glClaSqua9.tiara \
    --text-delimiter "\t" \
    --text-cols "sequence_id=identifiers,class_fst_stage=tiara" \
    --text-header \
    --key plot.cat=tiara \
    ./glClaSqua9_blobdir
```

## Demo 3

We can add other file types as well:

- **BUSCO full_table.tsv**
```
blobtools add \
    --busco ./precomputed/busco/glClaSqua9.busco.eukaryota_odb10.full_table.tsv \
    --busco ./precomputed/busco/glClaSqua9.busco.fungi_odb10.full_table.tsv \
    ./glClaSqua9_blobdir
```

- **Numeric fields**
```
blobtools add \
    --text ./precomputed/window_stats/glClaSqua9.window_stats.tsv \
    --text-delimiter '\t' \
    --text-header \
    --text-cols sequence=identifiers,eukaryota_odb10_count,fungi_odb10_count \
    ./glClaSqua9_blobdir/
```

- **Copy the blobdir and update the BTK API**
```
    cp -r glClaSqua9_blobdir /workspace/btk_example/src/data/example/
    
    curl $(gp url 8000)/api/v1/search/reload/testkey
```
