# Tractor HAUL
Implementation of Tractor local ancestry pipeline not relying on HAIL  
Revised Dec 8, 2021 - Michael

Uses [Gnomix](https://github.com/AI-sandbox/gnomix) for local ancestry inference

## Prerequisites
Libraries:
- plink (1.9)
- plink2
- eagle
- shapeit

Note all of these libraries must be available in your path, they are all installed by `00_install_gnomix.sh`, but make sure to double check that installations worked properly

Other:
- Supercomputer with SLURM workload manager
- Working directory path assigned to `WORKING_DIR` variable and study name assigned to `study` variable in the .env file
- Directory containing reference data in a folder, with the path to the folder assigned to `REF_DIR` in the .env file, with reference data for each chromosome in files named as `refpanel_chr${CHR}.vcf.gz`, where ${CHR} is the chromosome number (1-22, this has not been tested on X)
- A txt file containing subjects to use as the reference panel from the reference data, with the path assigned to `ref_subjects` in the .env file
- A .bed, .bim, .fam file containing genetic data of chromosomes 1-22 for the study, with the path and prefix assigned to `input_data` in the .env file (e.g. if your files are in /home/user/data and they are named mystudy.bed, mystudy.bim, mystudy.fam, then you would set `input_data=/home/user/data/mystudy`)
- A .fam file, with phenotype information (could be the same file used from above, but with .fam extension), and column order [FID, IID, PAT, MAT, SEX, PHENO1] (no header), with the path assigned to `fam_file` in the .env file

## Configuration
Gnomix has a few configuration settings that you can change; the pipeline will still work with non-default settings, but you may need to add lines in the training & running to copy result files from the LISA TMPDIR to a permanent directory. In changing the inference type, the main thing that changes is the runtime and the size of the models, so make sure to increase the time limit for submitted SLURM jobs, otherwise the job will terminate prematurely.

## Usage
### 1) Edit .env
- Set WORKING_DIR to the absolute path of the directory to run the ancestry pipeline inside
- Set REF_DIR to the absolute path of the directory with the reference VCF files, split by chromosome, with each file named `refpanel_chr${CHR}.vcf.gz`, where ${CHR} is the chromosome number (1-22, this has not been tested on X)
    - Note that these reference subjects should contain most of the SNPs contained in the sample data you plan to run local ancestry inference (LAI) on.

### 2) Install Gnomix
```
bash 00_install_gnomix.sh
```

### 3) Prepare for training

#### Get ambiguous SNPs
```
bash 00a_get_ambiguous_snps.sh
```

#### Split bed/bim/fam files by chromosome (this creates VCFs in `${study}/unphased`)
```
export $(cat .env | xargs); sbatch --array=1-22 --time=00:35:00 --error ${WORKING_DIR}/errandout/${study}/splitting/split_%a.e --output ${WORKING_DIR}/errandout/${study}/splitting/split_%a.o  --export=ALL,WORKING_DIR=$WORKING_DIR,study=$study  00b_split_by_chr.sh -D $WORKING_DIR
```

#### Phase chromosome vcf files (this creates VCFs in `${study}/phased`)
```
export $(cat .env | xargs); sbatch --array=1-22 --time=12:00:00 --error $WORKING_DIR/errandout/${study}/phasing/phase_%a.e --output $WORKING_DIR/errandout/${study}/phasing/phase_%a.o  --export=ALL,study=$study,WORKING_DIR=$WORKING_DIR  00c_phasing.sh -D $WORKING_DIR
```

### 4) Run training

#### Run training
Note: this usually takes ~15min-1hr depending on data size for the "default" inference type, and ~16-24-hrs for the "best" inference type (default is already so fast it doesn't really make sense to use the "fast" type)
```
export $(cat .env | xargs); sbatch --array=1-22 --time=02:00:00 --error ${WORKING_DIR}/errandout/${study}/training/train_%a.e --output ${WORKING_DIR}/errandout/${study}/training/train_%a.o  --export=ALL,study=$study,WORKING_DIR=$WORKING_DIR,REF_DIR=$REF_DIR,ref_subjects=$ref_subjects  01_train_gnomix.sh -D $WORKING_DIR
```

### 5) Run xgmix model local ancestry prediction
Note: this usually ~10min for the "default" inference type
WAIT FOR TRAINING TO FINISH, then:
```
export $(cat .env | xargs); sbatch --array=1-22 --time=00:30:00 --error ${WORKING_DIR}/errandout/${study}/running/run_%a.e --output ${WORKING_DIR}/errandout/${study}/running/run_%a.o  --export=ALL,study=$study,WORKING_DIR=$WORKING_DIR,REF_DIR=$REF_DIR,ref_subjects=$ref_subjects  02_run_gnomix.sh -D $WORKING_DIR
```

### 6) Expand local ancestry predictions from start and stop windows to SNPs

#### local ancestry expansion
```
export $(cat .env | xargs); sbatch --array=1-22 --time=12:00:00 --ntasks=1 --cpus-per-task=16 --error ${WORKING_DIR}/errandout/${study}/expansion/lanc_expansion_%a.e --output ${WORKING_DIR}/errandout/${study}/expansion/lanc_expansion_%a.o  --export=ALL,study=$study,WORKING_DIR=$WORKING_DIR  03_run_lanc_expansion.sh -D $WORKING_DIR
```

### 7) Plot local ancestry predictions
```
export $(cat .env | xargs); sbatch --time=12:00:00 --error ${WORKING_DIR}/errandout/${study}/plotting/plot_all.e --output ${WORKING_DIR}/errandout/${study}/plotting/plot_all.o  --export=ALL,study=$study,WORKING_DIR=$WORKING_DIR  04_run_lanc_plotting.sh -D $WORKING_DIR
```


### 8) Run plink covariate (using ancestries) regression
Note: this step has not been tested yet (will likely need adjustment/fixing)
```
export $(cat .env | xargs); sbatch --array=22 --time=12:00:00 --error ${WORKING_DIR}/errandout/${study}/regression/regression_%a.e --output ${WORKING_DIR}/errandout/${study}/regression/regression_%a.o  --export=ALL,study=$study,WORKING_DIR=$WORKING_DIR  05_run_plink_glm.sh -D $WORKING_DIR

```

## Other Usage Notes:  

The pipeline is a bit primitive, and for most will be a hackable example rather than a perfect out-of-the-box implementation

  A) jobs are not automatically resubmitted if failed.  
  B) There is no job dependency programmed in, you have to manually check if a job finished before proceeding to the next step  

#### SLURM:
 This assumes that you have access to a SLURM computing system (eg LISA)

##### If you do not have SLURM:  
   Run the contents of the job script commands in the shell. Because chromosomes are by default indexed by the array index number, you will have to create a for loop to replace the array indexing variable. ie.

     for (SLURM_ARRAY_TASK_ID in seq 1 1 22) ; do ... job script contents ... ; done  ;  

   You'll also need to install whatever compiling libraries are necessary to install xgmix by yourself

   You also may need to use shorter blocks for splitting the chromosomes (eg 30 mega-basepairs) as xgmix uses a lot of memory