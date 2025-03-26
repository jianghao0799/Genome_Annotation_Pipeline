# Genome annotation pipeline
jianghao, AGIS
2025-3-25 init.


## 01.RepeatModeler

First, we build de novo lib use RepeatModeler.
```
singularity exec ~/03_software/07_annotation/tetools_latest.sif BuildDatabase \
        -name genome.db \
        /lustre/home/jianghao/01_database/01_ref/a056.fa

singularity exec ~/03_software/07_annotation/tetools_latest.sif RepeatModeler \
        -database genome.db \
        -threads 28 \
        -LTRStruct 1>repeatmodeler.log 2>&1
```

## 02.RepeatMasker

We download dfam39_full.6.h5, 
https://darencard.net/blog/2022-07-09-genome-repeat-annotation/
