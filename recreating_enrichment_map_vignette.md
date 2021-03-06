# Recreating EnrichmentMap tutorials with RCy3
Julia Gustavsen  
May 24, 2016  
# Purpose: 

* Recreating tutorials from [Bader lab](http://www.baderlab.org/Software/EnrichmentMap/Tutorial) using Rcy3 and cytoscape.
* Create and test out functions using RCy3 that make Enrichment Map easy to use in R. 

# Enrichment Map
## Functional enrichment analysis

Many scientists use perform experiments to determine which biological pathways are expressed more in certain diseases or conditions. Using processed sequence data from RNAseq experiments, different treatments and/or samples can be visualized for genes that are more or less enriched compared to the baseline. This informs which genes are important for regulating processes related to the a disease or a treatment or another condition. Visualizing which genes are statistically more highly expressed under certain conditions can be helpful for interpretation and further analysis of the data. Based on which genes are enriched, it can also be determined which pathways are present in the specific disease or treatment. 

The functional enrichment analysis is done outside of this vignette. We will use already processed data and make a network from these data in Cytoscape using the package RCy3.

## Reproducible Functional enrichment analysis

The basic workflow is using R scripts with [RCy3](https://github.com/tmuetze/Bioconductor_RCy3_the_new_RCytoscape) to build [EnrichmentMaps](http://www.baderlab.org/Software/EnrichmentMap) that can be visualized and analysed in [Cytoscape](http://www.cytoscape.org/). The benefit of using R scripts is that it is easier to reproduce your analysis or to repeat it with different data. 

# Install done?

-  To proceed please follow instructions in [installation vignette](Install_vignette.html) if you do not already have RCy3 and Cytoscape installed. 

RCy3 (stands for R to Cytoscape 3, there is also a RCy that was used with Cytoscape 2 see [here](https://www.bioconductor.org/packages/release/bioc/html/RCytoscape.html). The RCy3 package (actively developed by Tanja Muetze, Georgi Kolishovski, Paul Shannon) uses the [CyREST api](https://github.com/idekerlab/cyREST/wiki) to allow communication between R and Cytoscape. CyREST now comes with all installations of Cytoscape. It uses the API (application programming interface) from Cytoscape to send and receive information from R. This means that we can send data from R to Cytoscape and also receive information about the graphs that you have made in Cytoscape in R. This is useful for reproducibility, but also if you are analysing networks in ways that are not yet supported by plugins in Cytoscape. 

# GSEA processed data
So what we will do today is to use data already processed in Gene Set Enrichment Analysis (GSEA which "determines whether an *a priori* defined set of genes shows statistically  significant, concordant differences between two biological states"). 

We will use this processed data to make an Enrichment Map in Cytoscape from R. 

## Load the appropriate libraries

```r
library(RCy3)
library(httr)
library(RJSONIO)
```

## Important note:

* Make sure Cytoscape is open before running the code below!

## Load functions for creating Enrichment map


```r
source("./functions_to_add_to_RCy3/working_with_EM.R")
```

Create the connection to Cytoscape

```r
# first, delete existing windows to save memory:
deleteAllWindows(CytoscapeConnection())
cy <- CytoscapeConnection ()
```

Examine the commands that are available in Enrichment Map.

```r
getEnrichmentMapCommandsNames(cy, "build")
```

```
##  [1] "analysisType"         "classDataset1"        "classDataset2"       
##  [4] "coeffecients"         "enrichments2Dataset1" "enrichments2Dataset2"
##  [7] "enrichmentsDataset1"  "enrichmentsDataset2"  "expressionDataset1"  
## [10] "expressionDataset2"   "gmtFile"              "phenotype1Dataset1"  
## [13] "phenotype1Dataset2"   "phenotype2Dataset1"   "phenotype2Dataset2"  
## [16] "pvalue"               "qvalue"               "ranksDataset1"       
## [19] "ranksDataset2"        "similaritycutoff"
```

```r
getEnrichmentMapCommandsNames(cy, "gseabuild")
```

```
## [1] "combinedconstant" "edbdir"           "edbdir2"         
## [4] "expressionfile"   "expressionfile2"  "overlap"         
## [7] "pvalue"           "qvalue"           "similaritymetric"
```

## Description of files that can be used

See the Bader lab website for full explanation:<https://github.com/BaderLab/EnrichmentMapApp/blob/EM_Cyto3_port/EnrichmentMapPlugin/doc/EM_wiki_manual.txt>

- **"gmtFile"**:  Tab-separated file where the header has name, description and samples and each row is one gene in the geneset.
- **"analysisType"** Analysis type can be "generic", "GSEA", or "DAVID/BiNGO/Great""
- **"classDataset1"** Classes (optional) for dataset1
- **"classDataset2"** Classes (optional) for dataset2 
- **"coeffecients"** Similarity Coefficient type (typo in name, but you must type it that way). Can be "OVERLAP","JACCARD", or "COMBINED" (both OVERLAP and JACCARD)
- **"similaritycutoff"** Similarity Cutoff for the coefficient chosen, 0.0 is least similar and 1.0 is most similar. 
- **"enrichments2Dataset1"** Tab-separated File containing gene names and their p-values from enrichment condition 2 from dataset 1
- **"enrichments2Dataset2"** Tab-separated File containing gene names and their p-values from enrichment condition 2 from dataset 2
-  **"enrichmentsDataset1"** Tab-separated File containing gene names and their p-values from enrichment condition 1 from dataset 1 
-  **"enrichmentsDataset2"** Tab-separated File containing gene names and their p-values from enrichment condition 1 from dataset 2
- **"expressionDataset1"** Expression (optional) tab-separated file containing gene name, description and expression values. For dataset1
- **"expressionDataset2"** Expression (optional) tab-separated file containing gene name, description and expression values. For dataset2
- **"phenotype1Dataset1"** Default is "Up" for Phenotype1. Can change to a descriptive word for enrichment 1 
- **"phenotype2Dataset1"** Default is "Down" for Phenotype2. Can change to a descriptive word for enrichment 2 
- **"phenotype1Dataset2"**  same as above but for Dataset2
- **"phenotype2Dataset2"** same as above but for Dataset2
- **"pvalue"** P-value Cutoff for enrichment data to be used.       
- **"qvalue"** False discovery rate (Q-value) cutoff. Used with multiple comparisons. 
- **"ranksDataset1"** Ranks (optional) tab-separated file containing genes and their rank (from GSEA) for Dataset1
-  **"ranksDataset2"** Ranks (optional) tab-separated file containing genes and their rank (from GSEA) for Dataset2

## Send data to the cytoscape network

So first we read in the data from the supplied files and set the parameters.

**Note on file paths**: 
You cannot use relative paths with EnrichmentMap from R. The filenames need to be given as their absolute paths.


```r
path_to_file <- "/home/julia_g/windows_school/gsoc/EM-tutorials-docker/notebooks/data/"

enr_file <- paste0(path_to_file,
                   "gprofiler_results_mesenonly_ordered_computedinR.txt")
exp_file <- paste0(path_to_file,
                   "MesenchymalvsImmunoreactive_expression.txt")
```

## Set the parameters for use in the Enrichment Map.

```r
em_params <- list(analysisType = "generic",
                  enrichmentsDataset1 = enr_file,
                  pvalue = "1.0",
                  qvalue = "0.00001",
                  expressionDataset1 = exp_file, 
                  similaritycutoff = "0.25",
                  coeffecients = "JACCARD")
```

No graph details are returned, this is just setting the parameters that will be sent to Cytoscape via RCy3. 

Now build the enrichment map

```r
EM_1 <- setEnrichmentMapProperties(cy,
                                   "build",
                                   em_params)
```

```
## [1] "Successfully built the EnrichmentMap."
## [1] "Cytoscape window EM1_Enrichment Map successfully connected to R session."
```

These parameters can also be set in Cytoscape, but we are setting them here via script. The function that we run also attaches the window created in Cytoscape to our R session, so that we are able to manipulate the stylistic aspects of our network by using "EM_1".

## Save Enrichment map network image


```r
fitContent(EM_1)
Sys.sleep(10)
saveImage(EM_1,
          "EM_1",
          "png",
         h=2000)  
```

![](./EM_1.png)

## Change the layout of the network

Can change any of the visual properties of "EM_1" now. For demonstration let's change the layout. 

```r
layoutNetwork(EM_1,
              'kamada-kawai')
Sys.sleep(5) 
## save network visualization
saveImage(EM_1,
          "EM_1_kamada-kawai",
          "png",
          h = 2000)
```
![](./EM_1_kamada-kawai.png)

We can also save our network file to use at a later date or to send to collaborators.


```r
saveNetwork(EM_1, "EM_1") ## Creates "EM_1.cys" which can be reopened in Cytoscape
```

In our R session we are connected to the graph in Cytoscape and can manipulate the graph's visual properties, but we do not have the graph information in R. 


```r
EM_1@graph
```

```
## A graphNEL graph with directed edges
## Number of Nodes = 0 
## Number of Edges = 0
```

If we want the graph pulled in to R then we can set the argument "copy.graph.to.R" to `TRUE` in `setEnrichmentMapProperties()`. This pulls in our EM analysis graph in Cytoscape and stores it as EM_1_2 in R.  


```r
EM_1_2 <- setEnrichmentMapProperties(cy,
                                     "build",
                                     em_params,
                                     copy.graph.to.R = TRUE)
```

```
## [1] "Successfully built the EnrichmentMap."
## [1] "Cytoscape windowEM2_Enrichment Map successfully connected to R session and graph copied to R."
```

Now if we have a look at the graph part of the stored object we will see we have retrieved the information from the graph. 

```r
EM_1_2@graph
```

```
## A graphNEL graph with directed edges
## Number of Nodes = 297 
## Number of Edges = 1793
```

```r
print(noa.names(getGraph(EM_1_2))) # retrieves node attributes from the graph
```

```
##  [1] "name"                    "EM2_GS_DESCR"           
##  [3] "EM2_Formatted_name"      "EM2_Name"               
##  [5] "EM2_GS_Source"           "EM2_GS_Type"            
##  [7] "EM2_pvalue_dataset1"     "EM2_Colouring_dataset1" 
##  [9] "EM2_fdr_qvalue_dataset1" "EM2_gs_size_dataset1"
```

```r
fitContent(EM_1_2)
Sys.sleep(10)
saveImage(EM_1_2,
          "EM_1_2",
          "png",
          h = 2000)
```

![](./EM_1_2.png)

# Summarizing Enrichment Results with Enrichment Maps

## Recreating [Protocol 4 - Summarize Enrichment Results with Enrichment Maps](https://github.com/BaderLab/EM-tutorials-docker/blob/master/notebooks/Protocol%204%20-%20Summarize%20Enrichment%20Results%20with%20Enrichment%20Maps.ipynb)

### Option 1: Load enrichment results from g:Profiler

Load in the datafiles

```r
path_to_file <- "/home/julia_g/windows_school/gsoc/EM-tutorials-docker/notebooks/data/"

enr_file <-  paste0(path_to_file,
                    "gprofiler_results_mesenonly_ordered.txt")

exp_file <- paste0(path_to_file,
                   "MesenchymalvsImmunoreactive_expression.txt") # this one works. 
#expression_RNA_seq <- paste0(path_to_file,
#                             "MesenchymalvsImmunoreactive_RNSseq_expression_ed.txt") ## not working
# ranks_file <- paste0(path_to_file,
#                      "MesenchymalvsImmunoreactive_RNA_seq_ranks.rnk") ## does not work
ranks_file <- paste0(path_to_file,
                     "MesenchymalvsImmunoreactive_edger_ranks.rnk")

classes_file <- paste0(path_to_file,
                       "MesenchymalvsImmunoreactive_RNAseq_classes.cls")
```


```r
em_params <- list(analysisType = "generic",
                  enrichmentsDataset1 = enr_file,
                  pvalue = "1.0",
                  qvalue = "0.0001",
                  expressionDataset1 = exp_file, 
                  ranksDataset1 = ranks_file, ## not working from RNA seq data
                  classDataset1 = classes_file,
                  phenotype1Dataset1 = "Mesenchymal", # shows up as positive, red, nodes. 
                  phenotype2Dataset1 = "Immunoreactive",
                  similaritycutoff = "0.25",
                  coeffecients = "JACCARD")

EM_ex_4 <- setEnrichmentMapProperties(cy,
                                      "build",
                                      em_params)
```

```
## [1] "Successfully built the EnrichmentMap."
## [1] "Cytoscape window EM3_Enrichment Map successfully connected to R session."
```



```r
fitContent(EM_ex_4)
Sys.sleep(10)
saveImage(EM_ex_4,
          "EM_ex_4",
          "png",
          h = 2000)
```

![](./EM_ex_4.png)


## Helpful references:

### Reference for the API

This site describes the cyREST API: http://idekerlab.github.io/cyREST/
