# Example rRNA dada2 pipeline used for single-nucleotide resolution (ASV) analysis of marine amplicons:


a16sF primer (A2F/Arch21f): 5’-TTCCGGTTGATCCYGCCGGA-3’

a16sR primer (519 R*): 5’-GWATTACCGCGGCKGCTG-3’

Archaea a16s rRNA truncated at:(R1= 228; R2= 228)


### Setup

```r
library(R.utils);
library(dada2);
library(ShortRead); packageVersion("ShortRead") # dada2 depends on this
library(tidyverse); packageVersion("dplyr") # for manipulating data
library(Biostrings);  # for creating the final graph at the end of the pipeline
library(Hmisc); packageVersion("Hmisc") # for creating the final graph at the end of the pipeline
library(plotly); packageVersion("plotly") # enables creation of interactive graphs, especially helpful for quality plots
library(here); packageVersion("here");
here();
library(readr);
library(Biostrings);
library(DECIPHER); packageVersion("DECIPHER");
```

### STEP 1

Setup import and export dirs


```r
home.dir <- ("/shared/c3/bio_db/BPA/amplicons/a16s/"); #BPA dataset location
setwd(home.dir);
plates<-read_csv('plates.1');

args = commandArgs(trailingOnly=TRUE);
p=args[1];
#p=4; #MAKE SURE THIS IS NOT RUN, ONLY REMOVE HASHTAG IF YOU WANT TO RUN ONE PLATE, in this case p=4 is just plate 4
print(p);
plates[p,1];

home.dir <- ("/shared/c3/bio_db/BPA/amplicons/a16s/"); #BPA dataset location

base.dir <- (paste0(home.dir, plates[p,1]));
dir.create(paste0(plates[p,1], ".final2020.exports"));
export_dir <-(paste0(plates[p,1], ".final2020.exports/"));
dir.create(paste0(export_dir, "pseudo.run_228_228"));
trimmed_dir <-(paste0(plates[p,1], ".trimmed/"));
dir.create(paste0(trimmed_dir));
trunc_dir <-(paste0(export_dir, "pseudo.run_228_228"));
trimLeng <-(paste0(trimmed_dir, "pseudo.run_228_228"));
dir.create(paste0(trimLeng));
```


### STEP2a

UNZIP your files if needed, here the pattern is fastq if your files are still zipped they will be unzipped


```r
files.fp.gz<-list.files(path=base.dir,  pattern=".fastq.gz");
data.fp.gz <- paste0(plates[p,1], "/");
print(data.fp.gz);

for (i in seq_along (files.fp.gz)){
              gunzip(filename=paste0(data.fp.gz,files.fp.gz[i]), overwrite=T)};
              
files.fp.gz<-list.files(path=base.dir,  pattern=".fastq");
data.fp.gz <- paste0(plates[p,1], "/");
print(data.fp.gz);

fnFs <- sort(list.files(data.fp.gz, pattern="_R1.fastq", full.names = TRUE));
fnRs <- sort(list.files(data.fp.gz, pattern="_R2.fastq", full.names = TRUE));

sample.names <- sapply(strsplit(basename(fnFs), "_"), `[`, 1);

fwd.plot.quals <- plotQualityProfile(fnFs[1:6])
ggsave(file = paste0(export_dir,"fwd.qualplot.pdf"), fwd.plot.quals, device="pdf");

rev.plot.quals <- plotQualityProfile(fnRs[1:6])
ggsave(file = paste0(export_dir,"rev.qualplot.pdf"), rev.plot.quals, device="pdf");


FWD <- "TTCCGGTTGATCCYGCCGGA";  ## CHANGE ME Archaea 16S: 2Af V1_V3
REV <- "GWATTACCGCGGCKGCTG"; ## CHANGE ME...Bacteria/Archaea 16S: 519r V1_V3

allOrients <- function(primer) {
    # Create all orientations of the input sequence
    require(Biostrings)
    dna <- DNAString(primer)  # The Biostrings works w/ DNAString objects rather than character vectors
    orients <- c(Forward = dna, Complement = complement(dna), Reverse = reverse(dna), 
        RevComp = reverseComplement(dna))
    return(sapply(orients, toString))  # Convert back to character vector
}

FWD.orients <- allOrients(FWD);
REV.orients <- allOrients(REV);

fnFs.filtN <- file.path(trimmed_dir, "filtN", basename(fnFs)); # Put N-filterd files in filtN/ subdirectory
fnRs.filtN <- file.path(trimmed_dir, "filtN", basename(fnRs));
filterAndTrim(fnFs, fnFs.filtN, fnRs, fnRs.filtN, maxN = 0, multithread = 10, compress=F, matchIDs=TRUE);

passed.filtN <- file.exists(fnFs.filtN) # TRUE/FALSE vector of which samples passed the filter
fnFs.filtN <- fnFs.filtN[passed.filtN] # Keep only those samples that passed the filter
fnFs <- fnFs[passed.filtN] # Keep only those samples that passed the filter
fnRs.filtN <- fnRs.filtN[passed.filtN] # Keep only those samples that passed the filter
fnRs <- fnRs[passed.filtN] # Keep only those samples that passed the filter

primerHits <- function(primer, fn) {
    # Counts number of reads in which the primer is found
    nhits <- vcountPattern(primer, sread(readFastq(fn)), fixed = FALSE)
    return(sum(nhits > 0));
}
rbind(FWD.ForwardReads = sapply(FWD.orients, primerHits, fn = fnFs.filtN[[1]]), 
    FWD.ReverseReads = sapply(FWD.orients, primerHits, fn = fnRs.filtN[[1]]), 
    REV.ForwardReads = sapply(REV.orients, primerHits, fn = fnFs.filtN[[1]]), 
    REV.ReverseReads = sapply(REV.orients, primerHits, fn = fnRs.filtN[[1]]));
 ```  
### STEP 4

Check and remove primers with cutadapt

    
```r
cutadapt <- "/shared/c3/apps/anaconda3/bin/cutadapt"; # This is the path for the HPC
#cutadapt <- "/usr/bin/cutadapt"; # this is the path for the FAST machine BETH/MARTIN

data.fp <- paste0(trimmed_dir, "filtN");

#system2(cutadapt, args = "--version"); # Run shell commands from R

path.cut <- file.path(trimmed_dir, "cutadapt");
if(!dir.exists(path.cut)) dir.create(path.cut);
fnFs.cut <- file.path(path.cut, basename(fnFs));
fnRs.cut <- file.path(path.cut, basename(fnRs));

FWD.RC <- dada2:::rc(FWD);
REV.RC <- dada2:::rc(REV);
# Trim FWD and the reverse-complement of REV off of R1 (forward reads)
R1.flags <- paste("-g", FWD, "-a", REV.RC);
# Trim REV and the reverse-complement of FWD off of R2 (reverse reads)
R2.flags <- paste("-G", REV, "-A", FWD.RC);

# Run Cutadapt
for(i in seq_along(fnFs)) {
  system2(cutadapt, args = c(R1.flags, R2.flags, "-n", 2, # -n 2 required to remove FWD and REV from reads
                             "-o", fnFs.cut[i], "-p", fnRs.cut[i], # output files
                             fnFs.filtN[i], fnRs.filtN[i])) # input files Default error rate: 0.1
}

data.fp <- paste0(trimmed_dir, "cutadapt");

primerHits <- function(primer, fn) {
    # Counts number of reads in which the primer is found
    nhits <- vcountPattern(primer, sread(readFastq(fn)), fixed = FALSE)
    return(sum(nhits > 0));
}

rbind(FWD.ForwardReads = sapply(FWD.orients, primerHits, fn = fnFs.cut[[1]]), 
    FWD.ReverseReads = sapply(FWD.orients, primerHits, fn = fnRs.cut[[1]]), 
    REV.ForwardReads = sapply(REV.orients, primerHits, fn = fnFs.cut[[1]]), 
    REV.ReverseReads = sapply(REV.orients, primerHits, fn = fnRs.cut[[1]]));
```

### STEP 5

Dada2 trim step (CHANGE TRIM lengths in this step!)

here you are renaming your new files to be sample name_R1_trim.fastq, no need to change


```r
trimFs <- file.path(trimLeng, paste0(sample.names, "_R1_trim.fastq"));
trimRs <- file.path(trimLeng, paste0(sample.names, "_R2_trim.fastq"));
names(trimFs) <- sample.names;
names(trimRs) <- sample.names;
head(sample.names);

out <- filterAndTrim(fnFs.cut, trimFs, fnRs.cut, trimRs, truncLen=c(228,228),
              maxN=0, maxEE=c(2,6), truncQ=6, rm.phix=TRUE,
              compress=FALSE, multithread=10, minLen = 50, matchIDs=TRUE); # On Windows set multithread=FALSE
head(out);

saveRDS(out, file = paste0(trunc_dir,"/", "out.RDS"), ascii = FALSE, version = NULL,
        compress = TRUE, refhook = NULL)
        
passed.trim <- file.exists(trimFs); # TRUE/FALSE vector of which samples passed the filter
trimFs <- trimFs[passed.trim]; # Keep only those samples that passed the filter
trimRs <- trimRs[passed.trim]; # Keep only those samples that passed the filter

data.fp <- paste0(trimLeng);
```


### STEP 6

Estimate the error rates

```r


errF <- learnErrors(trimFs, nbases =1e8, verbose=TRUE, multithread=10, MAX_CONSIST=20);
errR <- learnErrors(trimRs, nbases =1e8, verbose=TRUE, multithread=10, MAX_CONSIST=20);

fwd.plot.errors <- plotErrors(errF, nominalQ=TRUE);
ggsave(file = paste0(trunc_dir,"/","fwd.errors.pdf"), fwd.plot.errors, width = 10, height = 10, device="pdf");
rev.plot.errors <- plotErrors(errR, nominalQ=TRUE);
ggsave(file = paste0(trunc_dir,"/","rev.errors.pdf"), rev.plot.errors, width = 10, height = 10, device="pdf");
```



### STEP 7

DEREPLICATION, DADA2 Step, MERGE, and COLLAPSE if the same! SAVE RDS

```r
derepFs<-derepFastq(trimFs);
derepRs<-derepFastq(trimRs);

dadaFs <- dada(derepFs, err=errF, multithread=10, pool="pseudo");
dadaRs <- dada(derepRs, err=errR, multithread=10, pool="pseudo"); 
mergers <- mergePairs(dadaFs, derepFs, dadaRs, derepRs, minOverlap = 10, maxMismatch = 1, verbose=TRUE);

# Inspect the merger data.frame from the first sample
head(mergers[[1]]);
seqtab <- makeSequenceTable(mergers);
dim(seqtab);
saveRDS(seqtab, file = paste0(trunc_dir,"/",plates[p,1],"seqtab.RDS"), ascii = FALSE, version = NULL,
        compress = TRUE, refhook = NULL)
```



### STEP 8

Dada2 chimera removal step

using 2 different thresholds for removal (minFoldParentOverAbundance=4 and 1), 1 is standard

Inspect distribution of sequence lengths

```r
table(nchar(getSequences(seqtab)));
seqtab.nochim.4 <- removeBimeraDenovo(seqtab, method="consensus", multithread=10, verbose=TRUE, minFoldParentOverAbundance=4);
dim(seqtab.nochim.4);
sum(seqtab.nochim.4)/sum(seqtab);

saveRDS(seqtab.nochim.4, file = paste0(trunc_dir,"/",plates[p,1],"seqtab.4.RDS"), ascii = FALSE, version = NULL,
        compress = TRUE, refhook = NULL)
```



### STEP9

WRITE TABLES!

changeG information in the following CSVs to match what you used in step 5 so that you have a record of your parameters when you should need it


```r
getN <- function(x) sum(getUniques(x));
track.4 <- cbind(out, sapply(dadaFs, getN), sapply(dadaRs, getN), sapply(mergers, getN), rowSums(seqtab.nochim.4));
# If processing a single sample, remove the sapply calls: e.g. replace sapply(dadaFs, getN) with getN(dadaFs)
colnames(track.4) <- c("input", "filtered", "denoisedF", "denoisedR", "merged", "nonchim.4");
rownames(track.4) <- sample.names;
head(track.4);

# If processing a single sample, remove the sapply calls: e.g. replace sapply(dadaFs, getN) with getN(dadaFs)
track.table4 <- as.data.frame(t(track.4));
track.table4$truncL <- 228;
track.table4$truncR <- 228;
track.table4$EE <- 2.6;
track.table4$truncQ <- 6;
track.table4$pooled <- "pseudo";
track.table4$Bimera <- 4;
track.table4$nochim4_precent <- sum(seqtab.nochim.4)/sum(seqtab);
write.csv(track.table4, file = paste0(trunc_dir, "/",plates[p,1],".trackB4.csv"), col.names=NA);


table(nchar(getSequences(seqtab)));
seqtab.nochim.1 <- removeBimeraDenovo(seqtab, method="consensus", multithread=10, verbose=TRUE, minFoldParentOverAbundance=1);
dim(seqtab.nochim.1);
sum(seqtab.nochim.1)/sum(seqtab);

track.1 <- cbind(out, sapply(dadaFs, getN), sapply(dadaRs, getN), sapply(mergers, getN), rowSums(seqtab.nochim.1));
# If processing a single sample, remove the sapply calls: e.g. replace sapply(dadaFs, getN) with getN(dadaFs)
colnames(track.1) <- c("input", "filtered", "denoisedF", "denoisedR", "merged", "nonchim.1");
rownames(track.1) <- sample.names;
head(track.1);

track.table1 <- as.data.frame(t(track.1));
track.table1$truncL <- 228;
track.table1$truncR <- 228;
track.table1$EE <- 2.6;
track.table1$truncQ <- 6;
track.table1$pooled <- "pseudo";
track.table1$Bimera <- 1;
track.table1$nochim1_precent <- sum(seqtab.nochim.1)/sum(seqtab);
write.csv(track.table1, file = paste0(trunc_dir,"/",plates[p,1],".trackB1.csv"), col.names=NA);

```
# TRACK READS THROUGH THE PIPELINE FOR CHIM.4


```r
getN <- function(x) sum(getUniques(x));
track.4 <- cbind(out, sapply(dadaFs, getN), sapply(dadaRs, getN), sapply(mergers, getN), rowSums(seqtab.nochim.4));

```
If processing a single sample, remove the sapply calls: e.g. replace sapply(dadaFs, getN) with getN(dadaFs)

```
colnames(track.4) <- c("input", "filtered", "denoisedF", "denoisedR", "merged", "nonchim.4");
rownames(track.4) <- sample.names;
head(track.4);

track.4 <- as.data.frame(t(track.4));

plotLengthDistro <- function(st) {
  require(ggplot2)
  tot.svs <- table(nchar(colnames(st)))
  tot.reads <- tapply(colSums(st), nchar(colnames(st)), sum)
  df <- data.frame(Length=as.integer(c(names(tot.svs), names(tot.reads))),
                   Count=c(tot.svs, tot.reads),
                   Type=rep(c("SVs", "Reads"), times=c(length(tot.svs), length(tot.reads))))
  pp <- ggplot(data=df, aes(x=Length, y=Count, color=Type)) + geom_point() + facet_wrap(~Type, scales="free_y") + theme_bw() + xlab("Amplicon Length")
  pp
  }

plotLengthDistro(seqtab.nochim.4);
plotLengthDist.log10.chim4 <-plotLengthDistro(seqtab.nochim.4) + scale_y_log10();

ggsave(plotLengthDist.log10.chim4, file = paste0(trunc_dir,"/","plotLengthDist.log10.chim4.pdf"), width = 10, height = 10, device="pdf")

sample <- rownames(seqtab.nochim.4);
sequence <- colnames(seqtab.nochim.4);
```
#check the col names and check how many ASVs that you are losing in the pipeline

```r
colnames(seqtab.nochim.4);
#what % had chimera's vs non-chimeras?
sum(seqtab.nochim.4)/sum(seqtab);
#this is the %
sum(rev(sort(colSums(seqtab.nochim.4)))[1:1000])/sum(colSums(seqtab.nochim.4));

# Flip table
seqtab.t.4 <- as.data.frame(t(seqtab.nochim.4));
write.csv(seqtab.t.4, file = paste0(trunc_dir,"/",plates[p,1],".ASV.table.chim.4.csv"), col.names=NA);
```


tracking reads by percentage

```r
track.4 <- cbind(out, sapply(dadaFs, getN), sapply(dadaRs, getN), sapply(mergers, getN), rowSums(seqtab.nochim.4));
colnames(track.4) <- c("input", "filtered", "denoisedF", "denoisedR", "merged", "nonchim.4");
rownames(track.4) <- sample.names;
head(track.4);

track_pct <- track.4 %>% 
  data.frame() %>%
  mutate(Sample = rownames(.),
         filtered_pct = ifelse(filtered == 0, 0, 100 * (filtered/input)),
         denoisedF_pct = ifelse(denoisedF == 0, 0, 100 * (denoisedF/filtered)),
         denoisedR_pct = ifelse(denoisedR == 0, 0, 100 * (denoisedR/filtered)),
         merged_pct = ifelse(merged == 0, 0, 100 * merged/((denoisedF + denoisedR)/2)),
         nonchim_pct = ifelse(nonchim.4 == 0, 0, 100 * (nonchim.4/merged)),
         total_pct = ifelse(nonchim.4 == 0, 0, 100 * nonchim.4/input)) %>%
  select(Sample, ends_with("_pct"));


track_pct_avg <- track_pct %>% summarize_at(vars(ends_with("_pct")), 
                                            list(avg = mean));
head(track_pct_avg);

track_pct_med <- track_pct %>% summarize_at(vars(ends_with("_pct")), 
                                            list(avg = stats::median));
head(track_pct_avg);

head(track_pct_med);

track_plot.4 <- track.4 %>% 
  data.frame() %>%
  mutate(Sample = rownames(.)) %>%
  gather(key = "Step", value = "Reads", -Sample) %>%
  mutate(Step = factor(Step, 
                       levels = c("input", "filtered", "denoisedF", "denoisedR", "merged", "nonchim.4"))) %>%
  ggplot(aes(x = Step, y = Reads)) +
  geom_line(aes(group = Sample), alpha = 0.2) +
  geom_point(alpha = 0.5, position = position_jitter(width = 0)) + 
  stat_summary(fun.y = median, geom = "line", group = 1, color = "steelblue", size = 1, alpha = 0.5) +
  stat_summary(fun.y = median, geom = "point", group = 1, color = "steelblue", size = 2, alpha = 0.5) +
  stat_summary(fun.data = median_hilow, fun.args = list(conf.int = 0.5), 
               geom = "ribbon", group = 1, fill = "steelblue", alpha = 0.2) +
  geom_label(data = t(track_pct_avg[1:5]) %>% data.frame() %>% 
               rename(Percent = 1) %>%
               mutate(Step = c("filtered", "denoisedF", "denoisedR", "merged", "nonchim.4"),
                      Percent = paste(round(Percent, 2), "%")),
             aes(label = Percent), y = 1.1 * max(track.4[,2])) +
  geom_label(data = track_pct_avg[6] %>% data.frame() %>%
               rename(total = 1),
             aes(label = paste("Total\nRemaining:\n", round(track_pct_avg[1,6], 2), "%")), 
             y = mean(track.4[,6]), x = 6.5) +
  expand_limits(y = 1.1 * max(track.4[,2]), x = 7) +
  theme_classic();

ggsave(track_plot.4, file = paste0(trunc_dir,"/","track_plot.4.pdf"), width = 10, height = 10, device="pdf")
```
# TRACK READS THROUGH THE PIPELINE FOR CHIM.8

```r
getN <- function(x) sum(getUniques(x));
track.1 <- cbind(out, sapply(dadaFs, getN), sapply(dadaRs, getN), sapply(mergers, getN), rowSums(seqtab.nochim.1));

# If processing a single sample, remove the sapply calls: e.g. replace sapply(dadaFs, getN) with getN(dadaFs)
colnames(track.1) <- c("input", "filtered", "denoisedF", "denoisedR", "merged", "nonchim.1");
rownames(track.1) <- sample.names;
head(track.1);

track.1 <- as.data.frame(t(track.1));

plotLengthDistro <- function(st) {
  require(ggplot2)
  tot.svs <- table(nchar(colnames(st)))
  tot.reads <- tapply(colSums(st), nchar(colnames(st)), sum)
  df <- data.frame(Length=as.integer(c(names(tot.svs), names(tot.reads))),
                   Count=c(tot.svs, tot.reads),
                   Type=rep(c("SVs", "Reads"), times=c(length(tot.svs), length(tot.reads))))
  pp <- ggplot(data=df, aes(x=Length, y=Count, color=Type)) + geom_point() + facet_wrap(~Type, scales="free_y") + theme_bw() + xlab("Amplicon Length")
  pp
  }

plotLengthDistro(seqtab.nochim.1);
plotLengthDist.log10.chim.1 <-plotLengthDistro(seqtab.nochim.1) + scale_y_log10();

ggsave(plotLengthDist.log10.chim.1, file = paste0(trunc_dir,"/","plotLengthDist.log10.chim.1.pdf"), width = 10, height = 10, device="pdf")

sample <- rownames(seqtab.nochim.1);
sequence <- colnames(seqtab.nochim.1);

#check the col names and check how many ASVs that you are losing in the pipeline
colnames(seqtab.nochim.1);
#what % had chimera's vs non-chimeras?
sum(seqtab.nochim.1)/sum(seqtab);
#this is the %
sum(rev(sort(colSums(seqtab.nochim.1)))[1:1000])/sum(colSums(seqtab.nochim.1));

# Flip table
seqtab.t.1 <- as.data.frame(t(seqtab.nochim.1));
write.csv(seqtab.t.1, file = paste0(trunc_dir,"/",plates[p,1],".ASV.table.chim.1.csv"), col.names=NA);

# tracking reads by percentage

track.1 <- cbind(out, sapply(dadaFs, getN), sapply(dadaRs, getN), sapply(mergers, getN), rowSums(seqtab.nochim.1));
colnames(track.1) <- c("input", "filtered", "denoisedF", "denoisedR", "merged", "nonchim.1");
rownames(track.1) <- sample.names;
head(track.1);

track_pct <- track.1 %>% 
  data.frame() %>%
  mutate(Sample = rownames(.),
         filtered_pct = ifelse(filtered == 0, 0, 100 * (filtered/input)),
         denoisedF_pct = ifelse(denoisedF == 0, 0, 100 * (denoisedF/filtered)),
         denoisedR_pct = ifelse(denoisedR == 0, 0, 100 * (denoisedR/filtered)),
         merged_pct = ifelse(merged == 0, 0, 100 * merged/((denoisedF + denoisedR)/2)),
         nonchim_pct = ifelse(nonchim.1 == 0, 0, 100 * (nonchim.1/merged)),
         total_pct = ifelse(nonchim.1 == 0, 0, 100 * nonchim.1/input)) %>%
  select(Sample, ends_with("_pct"));


track_pct_avg <- track_pct %>% summarize_at(vars(ends_with("_pct")), 
                                            list(avg = mean));
head(track_pct_avg);

track_pct_med <- track_pct %>% summarize_at(vars(ends_with("_pct")), 
                                            list(avg = stats::median));
head(track_pct_avg);

head(track_pct_med);

track_plot.1 <- track.1 %>% 
  data.frame() %>%
  mutate(Sample = rownames(.)) %>%
  gather(key = "Step", value = "Reads", -Sample) %>%
  mutate(Step = factor(Step, 
                       levels = c("input", "filtered", "denoisedF", "denoisedR", "merged", "nonchim.1"))) %>%
  ggplot(aes(x = Step, y = Reads)) +
  geom_line(aes(group = Sample), alpha = 0.2) +
  geom_point(alpha = 0.5, position = position_jitter(width = 0)) + 
  stat_summary(fun.y = median, geom = "line", group = 1, color = "steelblue", size = 1, alpha = 0.5) +
  stat_summary(fun.y = median, geom = "point", group = 1, color = "steelblue", size = 2, alpha = 0.5) +
  stat_summary(fun.data = median_hilow, fun.args = list(conf.int = 0.5), 
               geom = "ribbon", group = 1, fill = "steelblue", alpha = 0.2) +
  geom_label(data = t(track_pct_avg[1:5]) %>% data.frame() %>% 
               rename(Percent = 1) %>%
               mutate(Step = c("filtered", "denoisedF", "denoisedR", "merged", "nonchim.1"),
                      Percent = paste(round(Percent, 2), "%")),
             aes(label = Percent), y = 1.1 * max(track.1[,2])) +
  geom_label(data = track_pct_avg[6] %>% data.frame() %>%
               rename(total = 1),
             aes(label = paste("Total\nRemaining:\n", round(track_pct_avg[1,6], 2), "%")), 
             y = mean(track.1[,6]), x = 6.5) +
  expand_limits(y = 1.1 * max(track.1[,2]), x = 7) +
  theme_classic();

ggsave(track_plot.1, file = paste0(trunc_dir,"/",plates[p,1],".track_plot.1.pdf"), width = 10, height = 10, device="pdf")
```
# save RDS files of the seqtab files of importance:

```r

saveRDS(seqtab.nochim.1, file = paste0(trunc_dir,"/",plates[p,1],"seqtab.1.RDS"), ascii = FALSE, version = NULL,
        compress = TRUE, refhook = NULL)

q()
```
