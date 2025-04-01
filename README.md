

# Genome annotation pipeline

jianghao, AGIS
2025-3-25 init.

## 1 Genome repeat annotation


### 1.1 RepeatModeler

First, we build de novo lib use RepeatModeler.
```
singularity exec tetools_latest.sif BuildDatabase \
        -name genome.db \
        a056.fa

singularity exec tetools_latest.sif RepeatModeler \
        -database genome.db \
        -threads 28 \
        -LTRStruct 1>repeatmodeler.log 2>&1
```

### 1.2 RepeatMasker

RepeatMasker version 4.1.2-p1

#### 1.2.1 Configure Libraries

Download RepBase
```
cd /path/to/RepeatMasker
wget https://github.com/chenyangkang/Repbase-Dfam/blob/main/RepBaseRepeatMaskerEdition-20181026.tar.gz
tar -zxvf RepBaseRepeatMaskerEdition-20181026.tar.gz
```
Extract to `Libraries/RMRBSeqs.embl` and Libraries/README.RMRBSeqs`

Download `Dfam` to `/path/to/RepeatMasker/Libraries/`
```
wget -c https://www.dfam.org/releases/Dfam_3.4/families/Dfam_curatedonly.embl.gz
wget -c https://www.dfam.org/releases/Dfam_3.4/families/Dfam_curatedonly.h5.gz
wget -c https://www.dfam.org/releases/Dfam_3.4/families/Dfam_curatedonly.hmm.gz

gunzip Dfam_curatedonly.embl.gz
gunzip Dfam_curatedonly.h5.gz
gunzip Dfam_curatedonly.hmm.gz

mv Dfam_curatedonly.embl Dfam.embl
mv Dfam_curatedonly.h5 Dfam.h5
mv Dfam_curatedonly.hmm Dfam.hmm
```
Then, `./configure`.

Building reference repeat database
```
famdb.py -i Libraries/RepeatMaskerLib.h5 families -f embl -a -d potato  > potato_ad.embl
util/buildRMLibFromEMBL.pl potato_ad.embl > potato_ad.fa
cat potato_ad.fa genome.db-families.fa > repeat_ad.fa
```
#### 1.2.2 Run RepeatMasker

```
RepeatMasker -e rmblast -xsmall -gff -html -lib repeat_db.fa -dir . -pa 28 a056.fa > RepeatMasker.log
```
>Note: Maybe It's not work in lustre file system.

https://darencard.net/blog/2022-07-09-genome-repeat-annotation/

### 1.3 EDTA

```
singularity exec edta_2.2.2.sif \
        EDTA.pl --genome ../a056.fa --species others --sensitive 1 --anno 1 --evaluate 1 --threads 30
```



## 2 Gene Structure Prediction

### 2.1 Transcriptome based gene structure prediction

Input: 

- genome.fa (masked)

- RNA-seq data

#### HISAT2 transcriptome alignment

First, we need build index of genome.fa (masked)

```shell
hisat2-build -p 8 potato.sm.fa potato
```

Then, mapping RNA-seq reads to genome.

> Note: We utilize Stringtie2 to assemble RNA-seq data, so must add the `--dta` option.

```shell
hisat2 -p 16 -x $index --dta \
	-1 clean_data/${id}_R1_clean.fq.gz \
	-2 clean_data/${id}_R2_clean.fq.gz | samtools sort -@ 16 > bams/${id}.sorted.bam
```

Bam merge and transcripts assemble.

```shell
samtools merge -@ 16 merged.bam `ls bams/*bam`
stringtie -p 16 -o stringtie.gtf merged.bam
```

Identifying candidate open reading frame (ORF) with transcripts by Transdecoder.

```shell
gtf_genome_to_cdna_fasta.pl stringtie.gtf \
	potato.sm.fa > transcripts.fasta
gtf_to_alignment_gff3.pl stringtie.gtf > transcripts.gff3
TransDecoder.LongOrfs -t transcripts.fasta
```

Download data and make database

```shell
wget https://ftp.uniprot.org/pub/databases/uniprot/current_release/knowledgebase/complete/uniprot_sprot.fasta.gz
pigz -d uniprot_sprot.fasta.gz
wget https://ftp.ebi.ac.uk/pub/databases/Pfam/current_release/Pfam-A.hmm.gz
pigz -d Pfam-A.hmm.gz
makeblastdb -in uniprot_sprot.fasta -dbtype prot -out uniprot_db
hmmpress Pfam-A.hmm
```

homology search

```shell
blastp -query transcripts.fasta.transdecoder_dir/longest_orfs.pep \
	-db uniprot_db -max_target_seqs 1 -outfmt 6 -evalue 1e-5 -num_threads 28 > blastp.outfmt6
	
hmmscan --cpu 28 --domtblout pfam.domtblout Pfam-A.hmm transcripts.fasta.transdecoder_dir/longest_orfs.pep

TransDecoder.Predict -t transcripts.fasta --retain_pfam_hits pfam.domtblout --retain_blastp_hits blastp.outfmt6cdna_alignment_orf_to_genome_orf.pl transcripts.fasta.transdecoder.gff3 transcripts.gff3 transcripts.fasta > transcripts.fasta.transdecoder.genome.gff3
```

Prediction complete gene structure
```shell
cdna_alignment_orf_to_genome_orf.pl transcripts.fasta.transdecoder.gff3 transcripts.gff3 transcripts.fasta > transcripts.fasta.transdecoder.genome.gff3

mv transcripts.fasta.transdecoder.genome.gff3 transcript_alignments.gff3
```

- transcript_alignments.gff3

### 2.2 BRAKER based *Ab initio* gene prediction

Input:

- genome.fa (masked)
- Viridiplantae.fa

> Download from https://bioinf.uni-greifswald.de/bioinf/partitioned_odb11/Viridiplantae.fa.gz

```shell
singularity exec braker3.sif \
        braker.pl --prot_seq=Viridiplantae.fa \
        --genome=potato.sm.fa \
        --bam=1.bam,2.bam \
        --thread 30 \
        --gff3 --workingdir=out \
        --softmasking
```

