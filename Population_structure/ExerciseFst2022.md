# Exercise on measuring population differentiation with F<sub>ST</sub>


We will use the same dataset that you used for the Monday exercise on analyzing population structure. 
The following commands will make a new folder and copy again the dataset to that new folder,
but you are free to work in the `structure` folder you created in the last exercise (and in that
case you don't need to copy the data again).

```bash
cd ~/exercises   # if you do not have a directory called exercises, make one:  mkdir ~/exercises 
mkdir structure_fst 
cd structure_fst

# Download data (remember the . in the end)
cp ~/groupdirs/SCIENCE-BIO-Popgen_Course/exercises/structure/pa/* .

# Show the dowloaded files
ls -l
```


Today, we are going to calculate the fixation index between subspecies
(*F<sub>ST</sub>*), which is a widely used statistic in population
genetics. This is a measure of population differentiation and thus, we
can use it to distinguish populations in a quantitative way.
It is worth noticing that what *F<sub>ST</sub>* measures is the
reduction in heterozygosity compared to a pooled population.


There are several methods for estimating *F<sub>ST</sub>* from genotype data. We will not
cover them in the course, but if you are interested in getting an overview of some of these
estiamtors and the how they differ you can take a look at [this article](https://genome.cshlp.org/content/23/9/1514.full.pdf).

Here we use the [Weir and Cockerham *F<sub>ST</sub>* calculator from 1984](https://onlinelibrary.wiley.com/doi/pdfdirect/10.1111/j.1558-5646.1984.tb05657.x) to
calculate *F<sub>ST</sub>* on the chimpanzees. Again, the theory behind it and the estimator itself are
not directly part of the course, but if you are interested you can see the formula that is implemented in the following R function in either
the Weir and Cockerham 1984 article, or in equation 6 from the Bhatia 2011 article linked above.


Open R and copy/paste the following function:

#### \>R
```R
WC84<-function(x,pop){
  # function to estimate Fst using Weir and Cockerham estimators.

  #number ind each population
  n<-table(pop)
  ###number of populations
  npop<-nrow(n)
  ###average sample size of each population
  n_avg<-mean(n)
  ###total number of samples
  N<-length(pop)
  ###frequency in samples
  p<-apply(x,2,function(x,pop){tapply(x,pop,mean)/2},pop=pop)
  ###average frequency in all samples (apply(x,2,mean)/2)
  p_avg<-as.vector(n%*%p/N )
  ###the sample variance of allele 1 over populations
  s2<-1/(npop-1)*(apply(p,1,function(x){((x-p_avg)^2)})%*%n)/n_avg
  ###average heterozygotes
  # h<-apply(x==1,2,function(x,pop)tapply(x,pop,mean),pop=pop)
  #average heterozygote frequency for allele 1
  # h_avg<-as.vector(n%*%h/N)
  #faster version than above:
  h_avg<-apply(x==1,2,sum)/N
  ###nc (see page 1360 in wier and cockerhamm, 1984)
  n_c<-1/(npop-1)*(N-sum(n^2)/N)
  ###variance betwen populations
  a <-n_avg/n_c*(s2-(p_avg*(1-p_avg)-(npop-1)*s2/npop-h_avg/4)/(n_avg-1))
  ###variance between individuals within populations
  b <- n_avg/(n_avg-1)*(p_avg*(1-p_avg)-(npop-1)*s2/npop-(2*n_avg-1)*h_avg/(4*n_avg))
  ###variance within individuals
  c <- h_avg/2
  ###inbreedning (F_it)
  F <- 1-c/(a+b+c)
  ###(F_st)
  theta <- a/(a+b+c)
  ###(F_is)
  f <- 1-c(b+c)
  ###weigted average of theta
  theta_w<-sum(a)/sum(a+b+c)
  list(F=F,theta=theta,f=f,theta_w=theta_w,a=a,b=b,c=c,total=c+b+a)
}
```

Now we will read in our data and apply to the three pairs of subspecies the funciton above to estiamte their
*F<sub>ST</sub>*. We want to make three comparisons.

#### \>R
```R
library(snpMatrix)

# read genotype data using read.plink function form snpMatrix package
data <- read.plink("pruneddata")

# extract genotype matrix, convert to normal R integer matrix
geno <- matrix(as.integer(data@.Data),nrow=nrow(data@.Data))

# original format is 0: missing, 1: hom minor, 2: het, 3: hom major
# convert to NA: missing, 0: hom minor, 1: het, 2: hom major
geno[geno==0] <- NA
geno <- geno - 1

# keep only SNPs without missing data
g <- geno[,complete.cases(t(geno))]

# load population infomration
popinfo <- read.table("pop.info", stringsAsFactors=F, col.names=c("pop", "ind"))

# get names of the three subspecies
subspecies <- unique(popinfo$pop)

# get all pairs of subspecies
subsppairs <- t(combn(subspecies, 2))

# apply fsts funciton to each of the three subspecies pairs
fsts <- apply(subsppairs, 1, function(x) WC84(g[popinfo$pop %in% x,], popinfo$pop[popinfo$pop %in% x]))

# name each fst 
names(fsts) <- apply(subsppairs, 1, paste, collapse="_")

# get global fsts for each pair
lapply(fsts, function(x) mean(x$theta, na.rm=T))

```

**Q13:** Does population differentiation fit with the geographical
distance between subspecies and their evolutionary history?




# Scanning for loci under selection using an *F<sub>ST</sub>* outlier approach

In the previous section, we have estimated *F<sub>ST</sub>* across all SNPs for which we have data, and then estiamted
a global *F<sub>ST</sub>* as the average across all SNPs. Now we will visualize local *F<sub>ST</sub>* in sliding windows across
the genome, with the aim of finding regions with outlying large *F<sub>ST</sub>*, that are candidate for regions under recent
positive selection in one of the populations.

We will now calculate and plot *F<sub>ST</sub>* values across the genome in sliding windows.
This is a common approach to scan the genome for candidate genes to have been under positive
selection in different populations.

First of all, we will copy the function we will use for plotting  a Manhattan plot of local *F<sub>ST</sub>* values across the genome in sliding windows in R:

``` R

manhattanWindowPlot <- function(mainv, xlabv, ylabv, ylimv=NULL, window.size, step.size,chrom,input_y_data, colpal = c("lightblue", "darkblue")){

    k <- ! is.na(input_y_data)
    input_y_data <- input_y_data[k]
    chrom <- chrom[k]
    
    chroms <- unique(chrom)
    step.positions <- c()
    win.chroms <- c()
    
    for(c in chroms){
        whichpos <- which(chrom==c)
        chrom.steps <- seq(whichpos[1] + window.size/2, whichpos[length(whichpos)] - window.size/2, by=step.size)
        step.positions <- c(step.positions, chrom.steps)
        win.chroms <- c(win.chroms, rep(c, length(chrom.steps)))
    }
    
    n <- length(step.positions)
    means_y <- numeric(n)
    
    for (i in 1:n) {
        chunk_y <- input_y_data[(step.positions[i]-window.size/2):(step.positions[i]+window.size/2)]
        means_y[i] <-  mean(chunk_y,na.rem=TRUE)
    }

    
    plot(x=1:length(means_y),y=means_y,main=mainv,xlab=xlabv,ylab=ylabv,cex=1,
         pch=20, cex.main=1.25, col=colpal[win.chroms %% 2 + 1], xaxt="n")

    yrange <- range(means_y)

    text(y=yrange[1] - c(0.05, 0.07) * diff(yrange) * 2, x=tapply(1:length(win.chroms), win.chroms, mean), labels=unique(win.chroms), xpd=NA)

	zz <- means_y[!is.na(means_y)]
	abline(h=quantile(zz,0.999,na.rem=TRUE),col="red", lty=2, lwd=2)
	abline(h=mean(input_y_data), lty=2, lwd=2)
}


pairnames <- apply(subsppairs, 1, paste, collapse=" ")

windowsize <- 10
steps <- 1

par(mfrow=c(3,1))
for(pair in 1:3){
  mainvv = paste("Sliding window Fst:", pairnames[pair], "SNPs =", length(fsts[[pair]]$theta), "Win: ", windowsize, "Step: ", steps)	
  manhattanWindowPlot(mainvv, "Chromosome", "Fst", window.size=windowsize, step.size=steps, input_y_data =fsts[[pair]]$theta, chrom=snpinfo$chr)
}

```

Using this function, we will now produce a Manhattan plot for each of the three sub species pairs:


``` R
# read bim file to get info on snp location
bim <- read.table("pruneddata.bim", h=F, stringsAsFactors=F)

# keep only sites without missing data (to get same sites we used for fst)
bim <- bim[,complete.cases(t(geno))]
# keep chromosome and bp coordinate of eachsnp
snpinfo <- data.frame(chr=bim$V1, pos=bim$V4)

pairnames <- apply(subsppairs, 1, paste, collapse=" ")

windowsize <- 10
steps <- 1

par(mfrow=c(3,1))
for(pair in 1:3){
  mainvv = paste("Sliding window Fst:", pairnames[pair], "SNPs =", length(fsts[[pair]]$theta), "Win: ", windowsize, "Step: ", steps)	
  manhattanWindowPlot(mainvv, "Chromosome", "Fst", window.size=windowsize, step.size=steps, input_y_data =fsts[[pair]]$theta, chrom=snpinfo$chr)
}
```

