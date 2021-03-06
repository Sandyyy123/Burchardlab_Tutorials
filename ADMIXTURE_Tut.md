# ADMIXTURE Tutorial
#### Pagé Goddard
#### Sep 28, 2017
___
## Content
* [ADMIXTURE Overview](#summary)
* [Set up: Choose your file locations](#files)
* [Pipeline](#format)
    - [Formatting genotype data](#format)
    - [Build popfile](#popfile)
    - [Run Admixture](#admix)
* [Parsing Results](#results)
    - [ADMIXTURE Output](#output)
    - [Reformat results](#reformat)
    - [Plotting Results](#plot)
* [Bonus: Thinning Files](#thin)
___

Tutorial for calculating global ancestry proportions from **known ancestral populations** using ADMIXTURE.

Sample data: SAGE 2

[ADMIXTURE manual](https://www.genetics.ucla.edu/software/admixture/admixture-manual.pdf)

<a name="summary"></a>
### ADMIXTURE Summary
Input | Output
-----------|--------
test.bed \(*plink*\) | test.k.Q \(*ancestry fractions*\)
test.bim \(*plink*\) | test.k.P \(*allele freq of inferred pop*\)
test.fam \(*plink*\)
test.pop \(*generate in R*\)

* test includes cohort of interest AND reference populations
* test binary files are 1/2 encoded
* k = number of reference populations

<a name="files"></a>
### File Locations
*script variables and directories*
```bash
date="171231" # update current yymmdd
wkdir="$HOME/wkdir" # choose your working directory
ancestraldir="path/to/your/ancestry.reference.panels.directory"
datadir="path/to/your/genotype.data.dir"

# data files
sagegenofile="$datadir/genotype.data" # I'v set it up this way so that you can make a second object (eg: galagenofile) for another dataset
ancestralgenofile="$ancestraldir/ref.panels" # reference panel genotypes

# all your data should be in PLINK binary format (.bim .bed. fam)
```
<a name="format"></a>
## PLINK: format data 
ADMIXTURE requires 1 input that contains your population of unknown ancestry and the references of known ancestry. 
If you do not have a cohort-reference panel file, you must merge them together in PLINK. 
**NB**: plink will merge the files in alphabetical order by FID, so you may have your reference panels merged into the middle of the file rather than appended.
```bash
# PLINK variables
merge="genotype_ancestry_attempt1_${date}"
ancestry_flip="ref_panels_flippedalleles_${date}"
myadmix="genotype_ancestry_merge_${date}"

# merge cohort + reference pops
plink --bfile $sagegenofile --bmerge $ancestralgenofile.bed $ancestralgenofile.bim $ancestralgenofile.fam --make-bed --out $merge

# if error due to missed snps, flip the problematic snps and merge again
plink --bfile $ancestralgenofile --flip $merge_missnp --make-bed --out $ancestry_flip
plink --bfile $sagegenofile --bmerge $ancestry_flip --make-bed --out $merge

# if error persists, remove the snps with --exclude missnp

# recode Cohort + Ref binary files in 1/2 format
plink --bfile $merge --recode 12 --out $myadmix
```

*optional: thin files for admixture, see bottom*

<a name="popfile"></a>
## R: build popfile
.pop is required for supervised admixture estimation; Each line of the .pop file corresponds to individual listed on the same line number in the .fam file. If the individual is a population reference, the .pop file line should be a string (beginning with an alphanumeric character) designating the population. If the individual is of unknown ancestry, use “-” (or a blank line, or any non-alphanumeric character) to indicate that the ancestry should be estimated. The final format should be a **single column** with **no header** that lines up with the fam file as shown here:

fam_ID_source | popfile
------|------
sage | -
sage | -
sage | -
euro_ref | ceu
euro_ref | ceu
afr_ref | yri
afr_ref | yri
sage | -
sage | -

```R
# R variables
setwd("PATH_TO/wkdir")
date <- "171231" # update current yymmdd
mydat <- "genotype_ancestry_merge_" # same prefix as myadmix
merge <- read.table(file=paste(mydat, date, ".fam", sep=""), header = F, sep = ' ') # .fam file has the samples for your population and reference populations
ceu <- read.table("samples_ceu.txt", header = F, sep = ' ') # european ancestry
yri <- read.table("samples_yri.txt", header = F, sep = ' ') # african ancestry
nam <- read.table("samples_nam.txt", header = F, sep = ' ') # native american ancestry

# make popfile
merge$pop = ifelse(merge$X1 %in% ceu$V1, 'CEU', ifelse(merge$X1 %in% yri$V1, 'YRI', '-')) 
    # if IID is from CEU ref, write CEU; 
    # if IID is from YRI ref, write YRI; 
    # if IID is from NAM ref, write NAM;
    # if IID is in neither ref, it is a SAGE individual with unknwon ancestry; write - as placeholder
popfile <- as.data.frame(merge$pop)

# write out
write.table(popfile, file=paste(mydat, date, ".pop", sep=""), row.names = F, quote = F)
    # check that same prefix as $myadmix input
```
<a name="admix"></a>
## ADMIXTURE: calculate global ancestry
```bash
# ADMIXTURE variables

## required
myadmix="genotype_ancestry_merge_${date}"
popfile=${myadmix}.pop
npop="2" # must reflect number of populations in reference panel

## optional
nthreads="64"
accelnum="qn3"
seed="2017"
#bootnum="200" # optional


# admixture script
admixture $myadmix.ped $npop --supervised --seed=$seed -j$nthreads
```

<a name="results"></a>
## Parsing ADMIXTURE results

<a name="output"></a>
### Admixture output:
myadmix.2.P - allele frequencies of the inferred ancestral populations (SNP x ancestry)
*order: CEU YRI*
```bash
wc -l $myadmix.2.P
    # 801660 lines
head $myadmix.2.P
    # 0.006379 0.000010
    # 0.827382 0.404886
    # 0.048119 0.021815
    # 0.874641 0.306484
```
myadmix.2.Q - ancestry fractions for each individual (sample x ancestry)
*order: CEU YRI*
```bash
wc -l $myadmix.2.Q
    # 2165 lines
head $myadmix.2.Q
    # 0.142886 0.857114
    # 0.123782 0.876218
    # 0.208400 0.791600
    # 0.109842 0.890158
```
<a name="reformat"></a>
### Reformat results
The following output can be used with programs such as [REAP](http://faculty.washington.edu/tathornt/software/REAP/REAP_Documentation.pdf) to calculate genetic relatedness matrices

```bash
# REAP variables
reapinput="genotype_transposed_${date}"
myID="${reapinput}_IID_only.txt"
admixprop="admixporp_${date}"
admixprop_sorted="admixprop_sorted_${date}"

# transpose genotype files
plink --bfile $sagegenofile --recode 12 transpose --output-missing-genotype 0 --out $reapinput

# reformat
cut -d' ' -f1 $reapinput.tfam > $myID # SAGE ID list
cut -d' ' -f1-2 merged_trial.fam | paste -d' ' - $myadmix.2.Q > admix_out_IDs_${date} # paste global ancestries to IDs; FID IID CEU YRI
    # -d' ' reads space as delimiter
    # -f1-2 selects fields 1 through 2 to extract with cut
    # - directs paste to use the standard input form the pipe instead of a file
grep -Fwf $myID admix_out_IDs_${date} > $admixprop # extract sage only
    # -f read patterns from file1
    # -F read patterns as plain strings
    # -w match patterns as whole word
awk 'FNR==NR {x2[$1] = $0; next} $1 in x2 {print x2[$1]}' $admixprop $reapinput.tfam > $admixprop_sorted
    # sorts $admixprop to match SAGE fam file
    # FNR==NR holds true for first named file
    # current line $0 is stored in associative array named x2 indexed by the first field [$1]
    # $1 in x2 will only start after first name file has been read completely
    # looks at the first field of line in second file, and prints the corresponding line from first
```

**output**
*order: FID IID EUR AFR*
```bash
head $admixprop_sorted
    # CH30380 CH30380 0.153140 0.846860
    # VA30171 VA30171 0.120529 0.879471
    # VA30167 VA30167 0.180591 0.819409
    # CH30357 CH30357 0.134801 0.865199
```
<a name="plot"></a>
### Plotting results

```R
#### files must be formatted with: race, eur, afr, natam
  # no subjectID, race column must come first. check example inputs

### Call libraries
require(truncnorm)
require(ggplot2)
require(gridExtra)
require(reshape2)

# set up space
setwd("PATH_TO/wkdir")
date <- "171231" # update current yymmdd
sage <- read.delim(print("$myadmix.2.Q"))
colnames(sage) <- c("EUR","AFR")
cbind(race = "AA", sage)
head(sage)
    #  race      EUR      AFR
    #  AA 0.142886 0.857114
    #  AA 0.123782 0.876218
    #  AA 0.208400 0.791600
    #  AA 0.109842 0.890158

# order data by increasing european ancestry
eur.sort = sage[order(sage$EUR),]
eur.sort$rank = 1:length(eur.sort$EUR)

# format data into geom_bar input
eur.sort.long = melt(eur.sort, id = c('rank','race'), variable.name = 'Ancestry')
head(eur.sort.long)
    #  race rank  Ancestry    value
    #    AA 1      EUR 0.088553
    #    AA 2      EUR 0.100432
    #    AA 3      EUR 0.134868
    #    AA 4      EUR 0.181657

# create plots
afr.plot <- ggplot(eur.sort.long, aes(x=rank, y=value, fill=Ancestry))
afr.plot <- afr.plot + geom_bar(width=1, stat='identity')           # play with width to see what it does
afr.plot <- afr.plot +  theme(legend.position="bottom")                 # move legend
afr.plot <- afr.plot +  xlab('Individual') + ylab('Genetic ancestry')# label the axes
afr.plot <- afr.plot +  ggtitle('SAGE 2')                 # create a title
afr.plot <- afr.plot +  scale_fill_manual(labels=c("Eur","Afr"),values=c("#cc9999","#660099"))
afr.plot <- afr.plot + theme(axis.text.x=element_blank(),
                             axis.ticks.x=element_blank())

afr.plot
```

![resulting plot](file:///C:/Users/page/Desktop/sage_ancestry_plot.png)


\# **QED** \#

***
<a name="thin"></a>
## Thinning files
[Documentation](https://www.genetics.ucla.edu/software/admixture/admixture-manual.pdf) SEC 2.3

* speeds up ADMIXTURE

*SAGE 2*
```bash
# PLINK variables
datadir="path/to/your/genotype.data.dir"
genofile=$myadmix
slidingwindowsize=50
slidewidth=10
r2val="0.1"
thingenofile="${myadmix}_thinned_indeppairwise_${slidingwindowsize}_${slidewidth}_${r2val}"

# run PLINK
plink --bfile $genofile --indep-pairwise $slidingwindowsize $slidewidth $r2val --make-bed --out $thingenofile
```
You can then continue through the popfile and admixture steps with the thinned files.
**NB**: Remember to test the order of thingenofile before making popfile
