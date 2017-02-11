---
title: 'TSIS: an R package to infer time-series isoform switch of alternative splicing'
author: "Wenbin Guo"
output:
  html_document:
    code_folding: show
    fig_caption: yes
    highlight: textmate
    theme: cerulean
    toc: yes
    toc_depth: 4
    toc_float:
      collapsed: false
      smooth_scroll: false
subtitle: User manual
---


#Description
[TSIS](https://github.com/wyguo/TSIS) is an R package for detecting transcript isoform switch for time-series data. Transcript isoform switch occurs when a pair of isoforms reverse the order of expression levels as shown in <a href="#fig1">Figure 1</a>. TSIS characterizes the transcript switch by 1) defining the isoform switch time points for any pair of transcript isoforms within a gene, 2) describing the switch using 5 different features, 3) filtering the results with user’s specifications and 4) visualizing the results using different plots for the user to examine further details of the switches. All the functions are available in the forms of a graphic interface implemented by [Shiny App](https://shiny.rstudio.com/) (a web application framework for R) ([Chang, et al., 2016](https://shiny.rstudio.com/)), in which users can implement the analysis as easy as mouse click. The tool can also be run just in command lines without graphic interface. This tutorial will cover both in the following sections. 


<h2 id="fig1"> </h2>
```{r,results='asis',echo=F,eval=TRUE}
if(figure.type=='html')
  cat('<img src="fig/figures_001.png" width="900px" height="200px" />')
if(figure.type=='word')
  cat('![](fig/figures_001.png)')
```
![](https://github.com/wyguo/TSIS/blob/master/vignettes/fig/figures_001.png)

**Figure 1:** Isoform switch analysis methods. Expression data with 3 replicates for each condition/time point is simulated for isoforms $iso_i$ and $iso_j$. The points in the plots represent the samples and the black lines are the average of samples. (A) is the iso-kTSP algorithm for comparisons of two conditions $c_1$ and $c_2$. The iso-kTSP is extended to time-series isoform switch (TSIS) in figure (B). The time-series with 6 time points is divided into 4 intervals by the intersection points of average expression. Five features for switch evaluation are determined based on the intervals before and after switch, e.g. the before and after intervals adjoined to switch point $P_i$. 



## Determine switch points
Given that a pair of isoforms $iso_i$ and $iso_j$ may have a number of switches in a time-series, we have offered two approaches to search for the switch time points in TSIS:

-  The first approach takes the average values of the replicates for each time points for each transcript isoform. Then it searches for the cross points for the average value of two isoforms across the time points (as the example in <a href="#fig1">Figure 1(B)</a>).
- The second approach uses natural spline curves to fit the time-series data for each transcript isoform and find cross points of the fitted curves for each pair of isoforms. 

It is reasonable to assume the isoform expression and time-series show curvilinear relationship. However, explicit average values of expression loss precision without having information of backward and forward time points. The spline method fit time-series of expression with control points (depending on spline degree of freedom provided) and weights of several neighbours to obtained designed precision (Hastie and Tibshirani, 1990).  The spline method is useful to find global trends of time-series when the data is very noisy. But it may sacrifice the local details of switch. For example, a rough spline fitting with very few control points may results in big shifts of switch points. Users can use both average and spline method to search for the switch points and determine optimal output by looking at the switch plots.

## Define switch scoring features
We define each transcript isoform switch by 1) the switch point $P_i$ , 2) time points between switch points $P_{i-1}$  and $P_i$ as interval before switch  $P_i$ and 3) time points between switch points $P_i$ and $P_{i+1}$  as interval after the switch $P_i$ (see <a href="#fig1">Figure 1(B)</a>).
We defined 5 measurements to score each isoform switch. The first two are the probability/frequency of switch and the sum of average sample differences before and after switch, which are similar to Score 1 and Score 2 in [iso-kTSP](https://bitbucket.org/regulatorygenomicsupf/iso-ktsp) method [(Sebestyen, et al., 2015)]( http://nar.oxfordjournals.org/content/early/2015/01/10/nar.gku1392.full.pdf) (see <a href="#fig1">Figure 1(A)</a>)). For Score 2, instead of rank differences as in [iso-kTSP](https://bitbucket.org/regulatorygenomicsupf/iso-ktsp) to avoid possible ties, we directly use the average sample differences. 

 - Firstly, for a switch point $P_i$ of two isoforms $iso_i$ and $iso_j$ with before interval $I_1$ and after interval $I_2$ (see example in <a href="#fig1">Figure1 (B)</a>), Score 1 is defined as
$$S_1 (iso_i,iso_j |I_1,I_2)=|p(iso_i>iso_j |I_1)+p(iso_i<iso_j |I_2)-1|,$$ 
Where  $p(iso_i>iso_j│I_1 )$ and $p(iso_i<iso_j│I_2 )$ are the frequencies/probabilities that the samples of one isoform is greater or less than in the other in corresponding intervals. 
- Secondly, sum of mean differences of samples in intervals $I_1$ and $I_2$ are calculated as
$$S_2=d(iso_i,iso_j│I_1 )+d(iso_i,iso_j |I_2)$$
Where $d(iso_i,iso_j|I_k)$ is the average expression difference in interval $I_k, k=1,2$ defined as
$$d(iso_i, iso_j|I_k)=\frac{1}{|I_k|}\sum_{m_{I_k}}\left|exp(iso_i|s_{m_{I_k}},I_k)-exp(iso_j|s_{m_{I_k}},I_k)\right|$$
$|I_k|$ is the number of samples in interval $I_k$ and $exp(iso_i|s_{m_{I_k}},I_k)$ is the expression of $iso_i$ of sample $s_{m_{I_k}}$ in interval $I_k$.
- Thirdly, a paired t-test is implemented for the two switched isoform sample differences within each interval. The dependency R function for testing is t.test(), i.e.
$$S_3 (iso_i,iso_j |I_k )=pval⇐t.test(x=iso_i  \text{ samples in } I_k,y=iso_j  \text{ samples in } I_k,\text{paired=TRUE})$$
Where $k=1,2$ represent the indices of the intervals before and after switch.
- Fourthly, the numbers of time points in intervals $I_1$ and $I_2$ were also provided, which indicate whether this switch is transient or long lived changes,
$$S_4(iso_i,iso_j|I_k)= \text{time points in interval } I_k$$
- Finally, the co-expressed isoform pairs often show good isoform switch patterns of interest. For example highly negative correlated isoforms may present opposite growing patterns along the time frame. As an additional score, we calculated the Pearson correlation of two isoforms across the whole time series
$$S_5 (iso_i,iso_j )=cor(\text{samples of } iso_1,\text{samples of  } iso_2 ,\text{method="pearson"})$$

#Installation and loading

## Check before installation 
Due to an issue with [devtools](https://cran.r-project.org/web/packages/devtools/index.html), if  R software is installed in a directory whose name has space character in it, e.g. in "C:\\Program Files", users may get error message "'C:\\Program' is not recognized as an internal or external command". This issue has to be solved by making sure that R is installed in a directory whose name has no space characters. 
Users can check the R installation location by typing
```{r}
R.home()
```

## Install dependency packages
install.packages(c("shiny", "shinythemes","ggplot2","plotly","zoo","gtools","devtools"), dependencies=TRUE)

## Install TSIS package
Install [TSIS](https://github.com/wyguo/TSIS)  package from Github using [devtools](https://cran.r-project.org/web/packages/devtools/index.html) package.
```{r}
library(devtools)
devtools::install_github("wyguo/TSIS")
```

## Loading
Once installed, TSIS package can be loaded as normal.
```{r}
library(TSIS)
```
## Example datasets
The [TSIS](https://github.com/wyguo/TSIS) package provides the example datasets "AtRTD2" with 2,666 genes and 6,307 isoforms, analysed in 26 time points, each with 3 biological replicates and 3 technical replicates. The experiments were designed to investigate the Arabidopsis gene expression response to cold. The isoform expression is in TPM (transcript per million) format. For the experiments and data quantification details, please see the AtRTD2 paper  [(Zhang, et al.,2016)](http://biorxiv.org/content/early/2016/05/06/051938). Other type of transcript quantifications, such as read counts, Percentage Splicing Ins (PSIs) can also be used in [TSIS](https://github.com/wyguo/TSIS).

The data loaded into the Shiny App must be in *.csv format. Users can download the example datasets from https://github.com/wyguo/TSIS/tree/master/data  or by typing the following codes in R console:
```{r}
AtRTD2.example()
```
The data will be saved in a folder "example data" in the working directory. <a href="#fig3">Figure 3</a> shows the examples of input data in csv format. 

#Shiny App -- as easy as mouse click
To make the implement more user friendly, TSIS analysis is integrated into a [Shiny App](https://shiny.rstudio.com/) ([Chang, et al., 2016](https://shiny.rstudio.com/)). By typing 
```{r}
TSIS.app() 
```
in R console after loading TSIS package, the App is opened in the default web browser. Users can upload input datasets, set parameters for switch analysis, visualize and save the results as easy as mouse click. The TSIS App includes three tab panels (see <a href="# whole1">Figure 2(A)</a>).

## Tab panel 1: Manual
The first tab panel includes this user manual.

##Tab panel 2: Isoform switch analysis
There are four sections in this panel (see <a href="#fig2">Figure 2</a>).

<br>
<h2 id="fig2"> </h2>
```{r,results='asis',echo=F,eval=TRUE}
if(figure.type=='html')
  cat('<img src="fig/figures_002.png" width="900px" height="400px" />')
if(figure.type=='word')
  cat('![](fig/figures_002.png)')
```
![](https://github.com/wyguo/TSIS/blob/master/vignettes/fig/figures_002.png)

**Figure 2:** Second tab panel in TSIS Shiny App. (A) is the three tab panels of the app; (B) is the data input interface; (C) is the interface for TSIS parameter setting; (D) provides the density/frequency plots of isoform switch time and (E) shows the output of TSIS analysis. 



###Input data files
Three *.csv format input files can be provided for [TSIS](https://github.com/wyguo/TSIS) analysis. 

- Time-series isoform expression data with first row of replicates labels and second row of time points. The remained lines are isoform names in the first column followed by the expression values corresponding to the sequential ordering of the first and second rows. All the replicates for each time point are adjoined from one to another in the table (see <a href="#fig3">Figure 3(A)</a>). 
- Gene and isoform mapping table with first column of gene names and second column of transcript isoform names (see <a href="#fig3">Figure 3(B)</a>)..
- Optional. Names of subset of isoforms. Users can output subset of results by providing a list of isoforms, for example the protein coding transcripts (see <a href="#fig3">Figure 3(C)</a>).

<br>
<h2 id="fig3"> </h2>
```{r,results='asis',echo=F,eval=TRUE}
if(figure.type=='html')
  cat('<img src="fig/figures_003.png" width="900px" height="400px" />')
if(figure.type=='word')
  cat('![](fig/figures_003.png)')
```
![](https://github.com/wyguo/TSIS/blob/master/vignettes/fig/figures_003.png)

**Figure 3: ** The format of input csv files for (A) transcript isoform expression, (B) two column table of gene-isoform mapping and (C) one column of a subset of isoform names.

<a href="#fig2">Figure 2(B)</a>  and <a href="#fig4">Figure 4(A)</a> shows the data input interface for time-series isoform expression and gene-isoform mapping. By clicking the "Browse…" button, a window is open for data loading (see <a href="#fig4">Figure 4(B)</a>). Users can use the interface shown in <a href="#fig4">Figure 4(C)</a>  to load the names of subset of isoforms. 

<br>
<h2 id="fig2"> </h2>
```{r,results='asis',echo=F,eval=TRUE}
if(figure.type=='html')
  cat('<img src="fig/figures_004.png" width="900px" height="400px" />')
if(figure.type=='word')
  cat('![](fig/figures_004.png)')
```
![](https://github.com/wyguo/TSIS/blob/master/vignettes/fig/figures_004.png)

**Figure 4: ** Interface for input information.  

###Parameter settings

#### Scoring parameters

The section in <a href="#fig2">Figure 2(C)</a> and <a href="#fig5">Figure 5</a>  is used to set the parameters for [TSIS](https://github.com/wyguo/TSIS). The parameters can be set by selecting or typing in corresponding boxes. Scoring process is starting by clicking the "Scoring" button. The parameter setting details are in the text followed the scoring button (see <a href="#fig5">Figure 5(A)</a>). Processing tacking bars for time-series intersection points searching and switch scoring (<a href="#fig5">Figure 5(B)</a>) for the isoform pairs will present in the bottom of the browser. 

#### Filtering parameters
<a href="#fig5">Figure 5(C)</a> is the interface for scoring feature filtering. Users can set cut-offs, such as for the probability/frequency of switch and sum of average differences, to further refine the switch results. The parameter setting details are in the text under the "Filtering" button. 

<br>
<h2 id="fig5"> </h2>
```{r,results='asis',echo=F,eval=TRUE}
if(figure.type=='html')
  cat('<img src="fig/figures_005.png" width="900px" height="400px" />')
if(figure.type=='word')
  cat('![](fig/figures_005.png)')
```
![](https://github.com/wyguo/TSIS/blob/master/vignettes/fig/figures_005.png)

**Figure 5:** TSIS parameter setting section. (A) is the scoring parameter input interface; (B) is the processing tracking bars and (C) is the switch score filtering interface.

### Density of switch points
The isoform switches occur at different time points in the time-series. To visualize the frequency and density plot of switch time, TSIS Shiny App provides the plot interface as shown in  <a href="#fig6">Figure 6</a>.  Frequency and density bar plots and line plots, which correspond to isoform switch time points after scoring and filtering processes, will present by clicking the corresponding radio buttons.  The plot can be saved in html, pdf and png format.

Note: The plot is made by using [plotly]( https://plot.ly/r/) R package. Users can move the mouse around the plot to show plot values and select part of the plot to zoom out. More actions are available by using the tool bar in the top right corner of the plot.

<br>
<h2 id="fig6"> </h2>
```{r,results='asis',echo=F,eval=TRUE}
if(figure.type=='html')
  cat('<img src="fig/figures_006.png" width="900px" height="400px" />')
if(figure.type=='word')
  cat('![](fig/figures_006.png)')
```
![](https://github.com/wyguo/TSIS/blob/master/vignettes/fig/figures_006.png)

**Figure 6:** Switch time density and frequency plot interface. 

### Output scores of isoform switch
The output table of TSIS analysisafter scoring or filtering. The columns include the information of isoform names, isoform ratios to genes, the intervals before and after switch, the coordinates of switch points and five scores of switch quality. Table columns can be sorted by clicking the small triangles beside the column names and contents can be searched by typing text in the search box. The explanations for each column are on the top of the table (see <a href="#fig7">Figure 7</a>). 

<br>
<h2 id="fig7"> </h2>
```{r,results='asis',echo=F,eval=TRUE}
if(figure.type=='html')
  cat('<img src="fig/figures_007.png" width="900px" height="400px" />')
if(figure.type=='word')
  cat('![](fig/figures_007.png)')
```
![](https://github.com/wyguo/TSIS/blob/master/vignettes/fig/figures_007.png)

**Figure 7:** The output switch measurements table. 

## Tab panel 3: Switch visualization

<br>
<h2 id="fig8"> </h2>
```{r,results='asis',echo=F,eval=TRUE}
if(figure.type=='html')
  cat('<img src="fig/figures_008.png" width="900px" height="400px" />')
if(figure.type=='word')
  cat('![](fig/figures_008.png)')
```
![](https://github.com/wyguo/TSIS/blob/master/vignettes/fig/figures_008.png)

**Figure 8:** The third tab panel of TSIS Shiny App. (A) is the switch plot section by providing a pair of isoform names. (B) is used to save top $n$ plot into a local folder. 

### Switch plots
Any pair of switched transcript isoforms can be visualized by providing their names. Plot type options are error bar plot and ribbon plot (see functions geom_errorbar and geom_smooth in [ggplot2](http://ggplot2.org) package for details) as shown in <a href="#fig8">Figure 8(A)</a> and example plots of AT5G60930 in <a href="#fig9">Figure 9</a> and <a href="#fig10">Figure 10</a>. An option is provided to only label the features of switch points with probability/frequency of switch>cut-off in the time region for investigation. The plots can be saved in html ([plotly](https://plot.ly/) format plot), png or pdf format.

### Multiple plots for switch
Transcript isoform switch profiles can be plotted in batch by selecting top n (ranking with Score 1 probability/frequency of switch) pairs of isoforms into png or pdf format plots (see <a href="#fig8">Figure 8(B)</a>).

#TSIS scripts -- step by step analysis
In addition to the Shiny App, users can use scripts to do TSIS analysis in R console. The following examples show a step-by-step tutorial of the analysis. Please refer to the function details using help function, e.g. help(iso.switch) or ?iso.switch.

## Loading datasets

```{r,eval=F,echo=T}
##load the data
library(TSIS)
data.exp<-AtRTD2$data.exp
mapping<-AtRTD2$mapping
dim(data.exp);dim(mapping)
```

## Scoring
**Example 1: search intersection points with mean expression**

```{r,echo=T,eval=F}
##Scores
scores.mean2int<-iso.switch(data.exp=data.exp,mapping =mapping,
                     t.start=1,t.end=26,nrep=9,rank=F,
                     min.t.points =2,min.difference=1,spline =F,
                     spline.df = 9,verbose = F)
```

<br>
**Example 2: search intersection points with spline method**
```{r}
##Scores, set spline=T and define spline degree of freedom to spline.df=9
scores.spline2int<-iso.switch(data.exp=data.exp,mapping =mapping,
                     t.start=1,t.end=26,nrep=9,rank=F,
                     min.t.points =2,min.difference=1,spline =T,
                     spline.df = 9,verbose = F)

```

##Filtering
**Example 1: general filtering with cut-offs**

```{r,eval=F}
##intersection from mean expression
scores.mean2int.filtered<-score.filter(
  scores = scores.mean2int,prob.cutoff = 0.5,dist.cutoff = 1,
  t.points.cutoff = 2,pval.cutoff = 0.01, cor.cutoff = 0.5,
  data.exp = NULL,mapping = NULL,sub.isoform.list = NULL,
  sub.isoform = F,max.ratio = F,x.value.limit = c(9,17) 
)

scores.mean2int.filtered[1:5,]
```

```{r}
##intersection from spline method
scores.spline2int.filtered<-score.filter(
  scores = scores.spline2int,prob.cutoff = 0.5,
  dist.cutoff = 1,t.points.cutoff = 2,pval.cutoff = 0.01,
  cor.cutoff = 0.5,data.exp = NULL,mapping = NULL,
  sub.isoform.list = NULL,sub.isoform = F,max.ratio = F,
  x.value.limit = c(9,17) 
)
```

<br>
**Example 2: only show subset of results according to an isoform list**

```{r}
##intersection from mean expression
##input a list of isoform names for investigation.
sub.isoform.list<-AtRTD2$sub.isoforms
sub.isoform.list[1:10]
##assign the isoform name list to sub.isoform.list and set sub.isoform=TRUE
scores.mean2int.filtered.subset<-score.filter(
  scores = scores.mean2int,prob.cutoff = 0.5,dist.cutoff = 1,
  t.points.cutoff = 2,pval.cutoff = 0.01, cor.cutoff = 0.5,
  data.exp = NULL,mapping = NULL,sub.isoform.list = sub.isoform.list,
  sub.isoform = T,max.ratio = F,x.value.limit = c(9,17) 
)
```

<br>
**Example 3: only show results of the most abundant transcript within a gene**

```{r}
scores.mean2int.filtered.maxratio<-score.filter(
  scores = scores.mean2int,prob.cutoff = 0.5,dist.cutoff = 1,
  t.points.cutoff = 2,pval.cutoff = 0.01, cor.cutoff = 0,
  data.exp = data.exp,mapping = mapping,sub.isoform.list = NULL,
  sub.isoform = F,max.ratio = T,x.value.limit = c(9,17) 
)
```

##Make plots

###Error bar plot

```{r}
plotTSIS(data2plot = data.exp,scores = scores.mean2int.filtered,
         iso1 = 'AT5G60930_P2',iso2 = ' AT5G60930_P3',gene.name = NULL,
         y.lab = 'Expression',make.plotly = F,
         t.start = 1,t.end = 26,nrep = 9,prob.cutoff = 0.5,
         x.lower.boundary = 9,x.upper.boundary = 17,
         show.region = T,show.scores = T,
         line.width =0.5,point.size = 3,
         error.type = 'stderr',show.errorbar = T,errorbar.size = 0.5,
         errorbar.width = 0.2,spline = F,spline.df = NULL,ribbon.plot = F)
```

<br>
<h2 id="fig9"> </h2>
```{r,results='asis',echo=F,eval=TRUE}
if(figure.type=='html')
  cat('<img src="fig/figures_009.png" width="900px" height="400px" />')
if(figure.type=='word')
  cat('![](fig/figures_009.png)')
```
![](https://github.com/wyguo/TSIS/blob/master/vignettes/fig/figures_009.png)

###Ribbon plot

```{r}
plotTSIS(data2plot = data.exp,scores = scores.mean2int.filtered,
         iso1 = 'AT5G60930_P2',iso2 = ' AT5G60930_P3',gene.name = NULL,
         y.lab = 'Expression',make.plotly = F,
         t.start = 1,t.end = 26,nrep = 9,prob.cutoff = 0.5,
         x.lower.boundary = 9,x.upper.boundary = 17,
         show.region = T,show.scores = T,
         line.width =0.5,point.size = 3,
         error.type = 'stderr',show.errorbar = T,errorbar.size = 0.5,
         errorbar.width = 0.2,spline = F,spline.df = NULL,ribbon.plot = T)
```

<br>
<h2 id="fig10"> </h2>
```{r,results='asis',echo=F,eval=TRUE}
if(figure.type=='html')
  cat('<img src="fig/figures_010.png" width="900px" height="400px" />')
if(figure.type=='word')
  cat('![](fig/figures_010.png)')
```
![](https://github.com/wyguo/TSIS/blob/master/vignettes/fig/figures_010.png)

#References
Chang, W., et al. 2016. shiny: Web Application Framework for R. https://CRAN.R-project.org/package=shiny

Hastie, T.J. and Tibshirani, R.J. Generalized additive models.  Chapter 7 of Statistical Models in S eds. Wadsworth & Brooks/Cole 1990.

Sebestyen, E., Zawisza, M. and Eyras, E. Detection of recurrent alternative splicing switches in tumor samples reveals novel signatures of cancer. Nucleic Acids Res 2015;43(3):1345-1356.

Zhang, R., et al. AtRTD2: A Reference Transcript Dataset for accurate quantification of alternative splicing and expression changes in Arabidopsis thaliana RNA-seq data. bioRxiv 2016.

#Session Information
```{r session, echo=FALSE,eval=T}
sessionInfo()
```
