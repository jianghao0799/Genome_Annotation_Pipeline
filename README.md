# Genome annotation pipeline
jianghao, AGIS
2025-3-25 init.


## 1.RepeatModeler

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

## 2.RepeatMasker

RepeatMasker version 4.1.2-p1

### 2.1 Configure Libraries
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
### 2.2 Run RepeatMasker

```
RepeatMasker -e rmblast -xsmall -gff -html -lib repeat_db.fa -dir . -pa 28 a056.fa > RepeatMasker.log
```
>Note: Maybe It's not work in lustre file system.

https://darencard.net/blog/2022-07-09-genome-repeat-annotation/

## 3. EDTA

```
singularity exec edta_2.2.2.sif \
        EDTA.pl --genome ../a056.fa --species others --sensitive 1 --anno 1 --evaluate 1 --threads 30
```
