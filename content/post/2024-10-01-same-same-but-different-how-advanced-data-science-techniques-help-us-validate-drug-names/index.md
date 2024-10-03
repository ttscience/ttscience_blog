---
title: 'Same-Same, but Different: How Advanced Data Science Techniques Help Us Validate
  Drug Names'
authors: ["kamil-pytlak"]
date: '2024-10-01'
categories:
  - machine learning
  - R
  - statistics
  - text analysis
tags:
  - data visualization
  - drug names
  - eCRF
  - data validation
  - levenshtein distance
  - NLP
  - t-SNE
slug: same-same-but-different-how-advanced-data-science-techniques-help-us-validate-drug-names
ShowToc: yes
TocOpen: yes
---

# Introduction

## Let's imagine...

... that we need to clean up our client's database for an upcoming interim analysis. We're looking at a lot of data, like patient demographics, patient outcomes (vital signs, physical exam, laboratory parameters), adverse events, and the medications they're taking. Once we load the data in R, it could look like this:


``` r
set.seed(7)

patient_ids <- sample(1:100, 20, replace = TRUE)

# Create a vector of medication names (with some intentional typos)
medications <- c("Aspirin", "Ibuprofin", "Paracetemol", "Metformn", "Lisinopril", 
                 "Omeprazole", "Metprolol", "Warferin", "Amoxcillin", "Simvastatin")

# Introduce typos by duplicating medication names with variations
medications_typo <- c(medications, "Aspirn", "Ibuprofine", "Paraceteml", "Metformin", 
                      "Lisinorpil", "Omeprazo", "Metoprolol", "Warfarin", "Amoxcilin", "Simavastatin")

# Randomly select medications (including those with typos)
con_meds <- sample(medications_typo, length(patient_ids), replace = TRUE)

con_meds_df <- data.frame(
  patient_id = patient_ids,
  con_med = medications_typo
)

print(con_meds_df)
```

```
##    patient_id      con_med
## 1          42      Aspirin
## 2          83    Ibuprofin
## 3          31  Paracetemol
## 4          92     Metformn
## 5          66   Lisinopril
## 6          15   Omeprazole
## 7          90    Metprolol
## 8           8     Warferin
## 9          67   Amoxcillin
## 10         88  Simvastatin
## 11         40       Aspirn
## 12         22   Ibuprofine
## 13         47   Paraceteml
## 14          8    Metformin
## 15         59   Lisinorpil
## 16         90     Omeprazo
## 17         12   Metoprolol
## 18         51     Warfarin
## 19         20    Amoxcilin
## 20         40 Simavastatin
```

The first column shows the patient ID, and the second column shows the drug they're taking. Some medications are repeated for different patients, which is normal. But some medications have been entered incorrectly, like Aspirin - Aspirn, Ibuprofin - Ibuprofine, and Omeprazole - Omeprazo. Let's look at the uniqueness of drug names:


``` r
unique(con_meds_df$con_med) |> sort()
```

```
##  [1] "Amoxcilin"    "Amoxcillin"   "Aspirin"      "Aspirn"       "Ibuprofin"   
##  [6] "Ibuprofine"   "Lisinopril"   "Lisinorpil"   "Metformin"    "Metformn"    
## [11] "Metoprolol"   "Metprolol"    "Omeprazo"     "Omeprazole"   "Paraceteml"  
## [16] "Paracetemol"  "Simavastatin" "Simvastatin"  "Warfarin"     "Warferin"
```
Do you see what's going on here? In a system that's not quite fully automated, drug names are entered by researchers. These are various researchers who don't have time to check which drug name is officially recognized as a model. Even if they do, it's easy to make a typo, even in the simplest of cases. 


## t-SNE + Distance Metrics = Visualization of Similarity Between Texts

To tackle the problem of detecting variations in drug names, we can use text distance metrics like Levenshtein distance, which measures the edits needed to transform one string into another. By calculating these distances, we can identify subtle differences. However, a distance matrix can be difficult to interpret. This is where t-SNE (t-distributed Stochastic Neighbor Embedding) is useful. It converts high-dimensional distance matrices into a 2D visualization, clustering similar items together. For example, typos like “Ibuprofin” and “Ibuprofine” will appear close together, making it easier to spot inconsistencies in drug names.

This approach transforms the complex task of detecting textual variations into a visual pattern-recognition challenge. Drug names with multiple typos or spellings cluster together; for instance, “Omeprazole” and its variations will form distinct groups, while names like “Warfarin” and “Warferin” will cluster closely. This visualization reveals patterns that are hard to discern in tables, such as recurring typos for specific drugs or misspellings used by different researchers.

Moreover, t-SNE can highlight complex patterns, like regional spelling differences (e.g., “Amoxicillin” vs. “Amoxycillin”). While traditional text-matching tools may miss these nuances, t-SNE effectively shows which names are similar based on their edit distances. When combined with interactive visualization tools like Plotly or ggplot, users can hover over data points to see original names, facilitating the exploration of clusters and enabling direct corrections. This method not only streamlines data cleaning but is crucial for maintaining drug name accuracy in clinical research, where precision is vital.


## Clustering - That Is, Group Similar Into Similar

The graphics can be tricky to interpret, especially when we have a lot of drugs. That's why we use an algorithmic approach based on clustering "similar into similar" in our workflow. One clustering technique we find really effective is DBSCAN (Density-Based Spatial Clustering of Applications with Noise). This algorithm is great at identifying clusters of different shapes and sizes, which helps us find meaningful patterns in the complex landscape of drug names. By looking at how dense the data is, DBSCAN groups similar drug names together while also showing us any outliers that might be typos. For example, it can group entries like "Ibuprofin" and "Ibuprofine," showing us any potential inconsistencies that could affect clinical trial results. This powerful approach not only makes the data more reliable but also makes it easier to clean up, so researchers can focus on getting good results in their clinical studies.


# Case Study: Validation of Drug Names for a Pharmaceutical Gig

## Overview of the Project and its Objectives

We did some detailed validation of eCRF data for one of our company's clients. This included things like demographics, lab results, and even concomitant medications (con-meds). 

> Concomitant drugs refer to the simultaneous or successive use of two or more drugs in order to achieve the purpose of treatment, and the main result is to enhance the drug efficacy or to decrease the drug side effects. (ScienceDirect, https://www.sciencedirect.com/topics/pharmacology-toxicology-and-pharmaceutical-science/concomitant-drug)

In our eCRF system, the researchers entered the drug names by hand. Unfortunately, text and humans don't always work well together. As you probably know, the popular acetylsalicylic acid can be written in a few different ways, including aspirin, salicylic acid, ASA, and... 2-acetoxybenzoic acid.

Have you already figured out what problems this can cause? Computers are pretty simple — to them, "aspirin" and "Aspirin" are two different text strings. So, when we're counting the proportions of patients taking different drugs at different stages of treatment, we'll get two separate proportions for aspirin and Aspirin, even though they're the same drugs.

So what solutions do we have?
- See what unique drugs [`unique(con_meds)`] our database contains, **find errors and just correct them**.
- See what unique drugs [`unique(con_meds)`] our database contains, **calculate the distance between their names, and then visualize them**.

If our database has a relatively small number of con-meds, the first option is probably the easiest. However, the more drugs patients are taking, the more complex it becomes. This is because different drugs can have similar names, and sometimes patients have different characteristics that affect their drug regimen. To quickly and effectively identify duplicates, ambiguities, and errors in drug names, we need a more efficient method. **This is where Levenshtein  and t-SNE come in handy.**


## Let's Do it in R: How to Quickly and Effectively Check the Differences Between Drug Names?

To begin with, we will simulate sample patients and, in particular, the medications they take. To do this, we will use the Anatomical Therapeutic Chemical (ATC) classification system. To scrap this dictionary, I used a script from user fabkury's [atcd](https://github.com/fabkury/atcd) repository on GitHub. The scraped data is current as of 05/09/2024. We also load libraries for data manipulation, visualization, t-SNE and DBSCAN construction.

> In the Anatomical Therapeutic Chemical (ATC) classification system, the active substances are divided into different groups according to the organ or system on which they act and their therapeutic, pharmacological and chemical properties. Drugs are classified in groups at five different levels. (WHO, https://www.who.int/tools/atc-ddd-toolkit/atc-classification)


``` r
library(dplyr)
library(stringdist )
library(Rtsne)
library(dbscan)
library(ggrepel)
library(ggplot2)
```



``` r
atc_df <- read.csv("WHO ATC-DDD 2024-09-05.csv")

head(atc_df, 10)
```

```
##    atc_code                        atc_name ddd  uom adm_r            note
## 1         A ALIMENTARY TRACT AND METABOLISM  NA <NA>  <NA>            <NA>
## 2       A01     STOMATOLOGICAL PREPARATIONS  NA <NA>  <NA>            <NA>
## 3      A01A     STOMATOLOGICAL PREPARATIONS  NA <NA>  <NA>            <NA>
## 4     A01AA      Caries prophylactic agents  NA <NA>  <NA>            <NA>
## 5   A01AA01                 sodium fluoride 1.1   mg     O 0.5 mg fluoride
## 6   A01AA02      sodium monofluorophosphate  NA <NA>  <NA>            <NA>
## 7   A01AA03                         olaflur 1.1   mg     O            <NA>
## 8   A01AA04               stannous fluoride  NA <NA>  <NA>            <NA>
## 9   A01AA30                    combinations  NA <NA>  <NA>            <NA>
## 10  A01AA51   sodium fluoride, combinations  NA <NA>  <NA>            <NA>
```

Next, we will extract only drug names (length `atc_code` equal to 7) and draw 100 drug names with possible repetitions.


``` r
set.seed(7)
drug_names <- atc_df |>
  filter(nchar(atc_code) == 7) |>
  slice_sample(n = 100, replace = TRUE) |>
  pull(atc_name)

unique(drug_names) |> sort()
```

```
##   [1] "acetylsalicylic acid and corticosteroids"      
##   [2] "aluminium preparations"                        
##   [3] "aminophylline"                                 
##   [4] "amphotericin B"                                
##   [5] "antazoline"                                    
##   [6] "artesunate and amodiaquine"                    
##   [7] "azacitidine"                                   
##   [8] "benazepril and amlodipine"                     
##   [9] "benzocaine"                                    
##  [10] "benzoyl peroxide"                              
##  [11] "betaine hydrochloride"                         
##  [12] "betamethasone"                                 
##  [13] "betaxolol, combinations"                       
##  [14] "bexagliflozin"                                 
##  [15] "biperiden"                                     
##  [16] "bupivacaine and meloxicam"                     
##  [17] "buspirone"                                     
##  [18] "calcium lactate"                               
##  [19] "calcium lactate gluconate"                     
##  [20] "captopril"                                     
##  [21] "carumonam"                                     
##  [22] "casopitant"                                    
##  [23] "cefapirin"                                     
##  [24] "ceftibuten"                                    
##  [25] "chymopapain"                                   
##  [26] "clotiazepam"                                   
##  [27] "cyanocobalamin"                                
##  [28] "desonide and antiseptics"                      
##  [29] "dexamethasone and antiinfectives"              
##  [30] "difluprednate"                                 
##  [31] "digitalis leaves"                              
##  [32] "diisopromine"                                  
##  [33] "eosin"                                         
##  [34] "epinastine"                                    
##  [35] "eplontersen"                                   
##  [36] "eptifibatide"                                  
##  [37] "ferric acetyl transferrin"                     
##  [38] "fluciclovine (18F)"                            
##  [39] "flumetasone"                                   
##  [40] "fluorouracil, combinations"                    
##  [41] "flutrimazole"                                  
##  [42] "folic acid"                                    
##  [43] "fostemsavir"                                   
##  [44] "gatifloxacin"                                  
##  [45] "gefarnate, combinations with psycholeptics"    
##  [46] "histapyrrodine, combinations"                  
##  [47] "Hyperici herba"                                
##  [48] "idrocilamide"                                  
##  [49] "indometacin, combinations"                     
##  [50] "iodine iofetamine (123I)"                      
##  [51] "isoprenaline"                                  
##  [52] "istradefylline"                                
##  [53] "kanamycin"                                     
##  [54] "lactulose"                                     
##  [55] "levodopa"                                      
##  [56] "levonorgestrel"                                
##  [57] "lincomycin"                                    
##  [58] "magnesium carbonate"                           
##  [59] "mecasermin"                                    
##  [60] "megestrol and estrogen"                        
##  [61] "menadione"                                     
##  [62] "methaqualone"                                  
##  [63] "micafungin"                                    
##  [64] "moexipril and diuretics"                       
##  [65] "narcobarbital"                                 
##  [66] "nebivolol and amlodipine"                      
##  [67] "nimetazepam"                                   
##  [68] "odevixibat"                                    
##  [69] "pegloticase"                                   
##  [70] "perphenazine"                                  
##  [71] "pethidine"                                     
##  [72] "phenylephrine"                                 
##  [73] "pipotiazine"                                   
##  [74] "pirprofen"                                     
##  [75] "plerixafor"                                    
##  [76] "potassium citrate"                             
##  [77] "prazosin"                                      
##  [78] "prednisone"                                    
##  [79] "remoxipride"                                   
##  [80] "reteplase"                                     
##  [81] "rifamycin"                                     
##  [82] "rivastigmine"                                  
##  [83] "roxithromycin"                                 
##  [84] "salsalate"                                     
##  [85] "sorbitol"                                      
##  [86] "streptokinase"                                 
##  [87] "succinimide"                                   
##  [88] "taurolidine"                                   
##  [89] "technetium (99mTc) pertechnetate"              
##  [90] "teneligliptin"                                 
##  [91] "theophylline, combinations excl. psycholeptics"
##  [92] "ticarcillin"                                   
##  [93] "tiemonium iodide and analgesics"               
##  [94] "timolol, thiazides and other diuretics"        
##  [95] "tolperisone"                                   
##  [96] "tramadol"                                      
##  [97] "tretoquinol"                                   
##  [98] "trypsin, combinations"                         
##  [99] "ursodoxicoltaurine"                            
## [100] "zidovudine"
```

We're looking to add a bit of confusion to our drug names, so we've created a function called `introduce_variation`. It takes a name and returns a new version with a duplicate, deleted, or rearranged character.


``` r
introduce_variation <- function(name) {
  # Randomly choose a type of modification to introduce a typo
  modification <- sample(c("duplicate", "remove", "swap"), 1)
  name_chars <- unlist(strsplit(name, ""))
  
  if (modification == "duplicate") {
    # Duplicate a random character
    duplicate_pos <- sample(1:length(name_chars), 1)
    name_chars <- append(name_chars, name_chars[duplicate_pos], after = duplicate_pos)
  } else if (modification == "remove") {
    # Remove a random character
    remove_pos <- sample(1:length(name_chars), 1)
    name_chars <- name_chars[-remove_pos]
  } else if (modification == "swap") {
    # Swap two adjacent characters
    swap_pos <- sample(1:(length(name_chars) - 1), 1)
    temp <- name_chars[swap_pos]
    name_chars[swap_pos] <- name_chars[swap_pos + 1]
    name_chars[swap_pos + 1] <- temp
  }
    
  return(paste(name_chars, collapse = ""))
}
```

Each drug name in `drug_names` has a 30% chance of generating a variation, so we'll end up with about 130 drug names instead of 100. We'll also add some drug names from another language (Polish).


``` r
set.seed(7)
sample_drug_names_with_typos <- sapply(drug_names, function(name) {
  if (runif(1) <= 0.3) {
    introduce_variation(name)
  } else {
    name
  }
})

complete_drug_names <- c(drug_names, sample_drug_names_with_typos)

# Add additional drug names in Polish
complete_drug_names <- c(complete_drug_names, c("Kwas acetylosalicylowy i kortykosteroidy", # Acetylsalicylic acid and corticosteroids
                                                "Węglan magnezu", # Magnesium carbonate
                                                "Kwas foliowy" # Folic acid
))

unique(complete_drug_names) |> sort() |> head(10)
```

```
##  [1] "acetylsalicylic acid and corticosteroids"
##  [2] "aluminium preparations"                  
##  [3] "aminophylline"                           
##  [4] "amphotericin B"                          
##  [5] "antazoline"                              
##  [6] "artesunate and amdoiaquine"              
##  [7] "artesunate and amodiaquine"              
##  [8] "azacitidine"                             
##  [9] "benazepril and amlodipine"               
## [10] "benazepril nd amlodipine"
```
And now for the most interesting part: we will visualize the names of the drugs on the chart. This is a two-step process:

1. Constructing a $ n \times n $ edit distance matrix between drug names using the Levenshtein  distance metric.
2. Using the t-SNE method to reduce the dimensions from $ n $ to 2 and visualize the embedding vectors.


``` r
unique_drug_names <- unique(complete_drug_names) |> sort()
```

When it comes to dimensionality reduction methods like t-SNE, it's important to think about a few key hyperparameters that can make a big difference in the resulting vector embeddings and, ultimately, the final visualization. In this case, I'm going to focus on `perplexity`, since it has the biggest impact here.

> A second feature of t-SNE is a tuneable parameter, “perplexity,” which says (loosely) how to balance attention between local and global aspects of your data. The parameter is, in a sense, a guess about the number of close neighbors each point has. The perplexity value has a complex effect on the resulting pictures. The original paper says, “The performance of SNE is fairly robust to changes in the perplexity, and typical values are between 5 and 50.” But the story is more nuanced than that. Getting the most from t-SNE may mean analyzing multiple plots with different perplexities. (Distill, https://distill.pub/2016/misread-tsne/)

In order to select the optimal choice of perplexity, I decided to use the method of its optimization using a modified Bayesian Schwarz information criterion associated with the KL divergence metric. The goal is to minimize `\(S(Perplexity)\)`. You can read more about this optimization technique here: [Automatic Selection of t-SNE Perplexity](https://arxiv.org/abs/1708.03229).


``` r
optimize_perplexity <-
  function(data,
           min_perp = 2,
           max_perp = 20,
           step = 1) {
    final_perp <- 0
    min_kl <- Inf
    n <- nrow(data)
    
    for (perp in seq(min_perp, max_perp, step)) {
      tsne_res <- Rtsne(data, is_distance = TRUE, perplexity = perp)
      
      kl_divergence <- 2 * min(tsne_res$itercosts) + log(n) * (perp / n)
      
      cat("Perplexity:", perp, "| KL Divergence:", kl_divergence, "\n")
      
      if (kl_divergence < min_kl) {
        min_kl <- kl_divergence
        final_perp <- perp
      }
      
      cat("Best Perplexity So Far:", final_perp, "| Best KL Divergence:", min_kl, "\n\n")
    }
    
    return(final_perp)
  }
```

We also want to figure out which drugs are in the same group. Each group should have drugs that are similar to each other in terms of Levenshtein distance. We'll use the DBSCAN algorithm for this. It doesn't require us to specify the number of clusters upfront. It estimates them based on things like `epsilon`. In other words, it's the furthest distance between two samples that determines if they're considered neighbours. For `minPts` I set the value to 2 (can you guess why?). For `epsilon`, on the other hand, I chose 15, which is due to the construction (offstage) of the k-distance plot


``` r
construct_tsne_with_dbscan <- function(data, max_perp = 20, eps = 15, minPts = 2) {
  # Step 1: Create a Levenshtein distance matrix
  distance_matrix <- stringdistmatrix(data, data, method = "lv")
  rownames(distance_matrix) <- data
  colnames(distance_matrix) <- data
  
  # Step 2: Convert the distance matrix into a format suitable for t-SNE
  distance_mtx <- as.dist(distance_matrix) |> as.matrix()
  
  # Step 3: Optimize the "perplexity" hyperparameter and run t-SNE on the distance matrix
  set.seed(7)
  
  best_perplexity <- optimize_perplexity(distance_mtx, max_perp = max_perp)

  tsne_result <- Rtsne(distance_mtx,
                       is_distance = TRUE,
                       perplexity = best_perplexity)

  # Step 4: Create a data frame with t-SNE results and drug labels
  tsne_df <- data.frame(Dim1 = tsne_result$Y[, 1],
                        Dim2 = tsne_result$Y[, 2],
                        Label = data)

  # Step 5: Perform OPTICS clustering on the t-SNE coordinates
  # Use the t-SNE coordinates for density-based clustering
  dbscan_result <- dbscan(distance_mtx, eps = eps, minPts = minPts)
  
  # Step 6: Add the DBSCAN cluster labels to the data frame
  tsne_df$Cluster <- as.factor(dbscan_result$cluster)
  
  return(tsne_df)
}
```

You'll find the t-SNE result in the graph below. This visualization shows how drug names are grouped together in a two-dimensional space based on how similar they are in terms of their text. Each point shows a drug name, and its position is based on the pairwise Levenshtein distances from the original names. Also, the color of the point shows which cluster it's in. You can see how the points are spread out and close together, which shows clusters of similar names. This could mean that there are some names that are similar but not the same, or that different naming conventions are used. Adding text labels makes it easier to understand what each point represents, so you can quickly identify specific drugs.


``` r
tsne_df <- construct_tsne_with_dbscan(unique_drug_names)
```

```
## Perplexity: 2 | KL Divergence: 1.365567 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 3 | KL Divergence: 1.467266 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 4 | KL Divergence: 1.605605 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 5 | KL Divergence: 1.612329 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 6 | KL Divergence: 1.841238 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 7 | KL Divergence: 1.664839 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 8 | KL Divergence: 1.691646 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 9 | KL Divergence: 1.789319 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 10 | KL Divergence: 1.751659 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 11 | KL Divergence: 1.799687 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 12 | KL Divergence: 1.731439 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 13 | KL Divergence: 1.812709 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 14 | KL Divergence: 1.706414 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 15 | KL Divergence: 1.795107 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 16 | KL Divergence: 1.728451 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 17 | KL Divergence: 1.791968 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 18 | KL Divergence: 1.796536 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 19 | KL Divergence: 1.730976 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567 
## 
## Perplexity: 20 | KL Divergence: 1.85299 
## Best Perplexity So Far: 2 | Best KL Divergence: 1.365567
```

``` r
tsne_df |>
    ggplot(aes(x = Dim1, y = Dim2, label = Label)) +
    geom_point(aes(color = Cluster), size = 1) +
  geom_text_repel(aes(label = Label), max.overlaps = 10, size = 2) +
    labs(x = "Dimension 1", y = "Dimension 2") +
    guides(color = "none") +
    theme_bw()
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-11-1.png" width="672" />

This graphic is pretty tricky to make sense of because there are so many data points and drug names that are all jumbled together. So, let's take a look at the DBSCAN clustering result instead:


``` r
clusters_list <- tsne_df |>
    group_by(Cluster) |>
    summarize(Drugs = paste(Label, collapse = " | ")) |>
    arrange(Cluster)
  
  # Print each cluster and the associated drug names
  for (i in 1:nrow(clusters_list)) {
    cat("Cluster", clusters_list$Cluster[i], ":\n")
    cat(clusters_list$Drugs[i], "\n\n")
  }
```

```
## Cluster 1 :
## acetylsalicylic acid and corticosteroids | aluminium preparations | aminophylline | amphotericin B | betaine hydrochloride | bexagliflozin | bupivacaine and meloxicam | calcium lactate | captopril | carumonam | casopitant | ceftibuten | chymopapain | clotiazepam | cyanocobalamin | desonide and antiseptics | dexamethasone and antiinfectives | diisopromine | epinastine | eptifibatide | ferric acetyl transferrin | fluciclovine (18F) | fluorouracil, combinations | flutrimazole | folic acid | gatifloxacin | gefarnate, combinations with psycholeptics | histapyrrodine, combinations | idrocilamide | iodine iofetamine (123I) | istradefylline | Kwas acetylosalicylowy i kortykosteroidy | Kwas foliowy | lactulose | levodopa | levonorgestrel | magnesium carbonate | methaqualone | moexipril and diuretics | narcobarbital | nimetazepam | pegloticase | phenylephrine | pipotiazine | plerixafor | reteplase | rivastigmine | roxithromycin | salsalate | sorbitol | succinimide | teneligliptin | theophylline, combinations excl. psycholeptics | tiemonium iodide and analgesics | timolol, thiazides and other diuretics | tolperisone | tramadol | tretoquinol | trypsin, combinations | ursodoxicoltaurine | Węglan magnezu | zidovudine 
## 
## Cluster 2 :
## antazoline | azacitidine | benzocaine | taurolidine 
## 
## Cluster 3 :
## artesunate and amdoiaquine | artesunate and amodiaquine 
## 
## Cluster 4 :
## benazepril and amlodipine | benazepril nd amlodipine 
## 
## Cluster 5 :
## benzoly peroxide | benzoyl peroxide 
## 
## Cluster 6 :
## betametahsone | betamethasone 
## 
## Cluster 7 :
## betaxolol, combinations | betaxolol, cominations 
## 
## Cluster 8 :
## biperiden | pirprofen 
## 
## Cluster 9 :
## buspirone | pethidine | prednisone 
## 
## Cluster 10 :
## calcium lactate gluconate | calcium lactate guconate 
## 
## Cluster 11 :
## cefapirin | eosin | mecasermin | prazosin 
## 
## Cluster 12 :
## dfluprednate | difluprednate 
## 
## Cluster 13 :
## digitalis leaves | digitalis lleaves 
## 
## Cluster 14 :
## eplonteresn | eplontersen 
## 
## Cluster 15 :
## flumetasone | fulmetasone 
## 
## Cluster 16 :
## fostemsavir | fostemsvir 
## 
## Cluster 17 :
## Hyerici herba | Hyperici herba 
## 
## Cluster 18 :
## indometacin, combinations | indometacinn, combinations 
## 
## Cluster 19 :
## isoprenaline | perphenazine 
## 
## Cluster 20 :
## kanamycin | lincomycin | lnicomycin | rifamycin 
## 
## Cluster 21 :
## megestrol and estorgen | megestrol and estrogen 
## 
## Cluster 22 :
## menadione | menadionee 
## 
## Cluster 23 :
## micafungin | ticarcillin 
## 
## Cluster 24 :
## nebivolol ad amlodipine | nebivolol and amlodipine 
## 
## Cluster 25 :
## odevixibat | oedvixibat 
## 
## Cluster 26 :
## potassium citrate | pottassium citrate 
## 
## Cluster 27 :
## remoxiipride | remoxipride 
## 
## Cluster 28 :
## srteptokinase | streptokinase 
## 
## Cluster 29 :
## technetium (99mTc) perechnetate | technetium (99mTc) pertechnetate
```

For instance, cluster 1 includes a wide variety of drugs, such as acetylsalicylic acid and corticosteroids, aminophylline, bupivacaine, and meloxicam.

Clusters like cluster 5, which includes benzoyl peroxide and its misspelled variant benzoly peroxide, show how useful Levenshtein distance can be for identifying typographical errors. Similarly, cluster 6 brings together different spellings of betamethasone.

This clustering analysis helps us understand the relationships among drug names and the potential issues that similar or identical names can cause. This isn't perfect, though. We still need to check if there are any other similar duplicates (take a look at the first cluster). Also, the Levenshtein metric is designed for editing, so it's not ideal for texts in different languages (see "Kwas foliowy" and "folic acid"). But it can definitely streamline our work and automate the process for future reports, and it provides some interesting insights!


# Final Thoughts: Con-meds data in EDC - how to make it clean and meaty?

It's really important to make sure that the data on the drugs people are taking at the same time as the ones being tested (concomitant medications, or "con-meds") is accurate and complete in the systems we use to capture data from clinical trials. To make this happen, we need to automate data entry tasks. Instead of relying on researchers to manually enter drug data, integrating the electronic case report form (eCRF) with a comprehensive drug dictionary, like the WHO Anatomical Therapeutic Chemical (ATC) classification, can make the process a lot more efficient. This integration would make it simple for users to enter or select drug names from a standard list, which would help to avoid errors caused by spelling variations or outdated names. By using these automated solutions, we can make sure that our con-meds data is cleaner and more reliable, which will improve the overall quality of clinical research.