    # https://bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.html
    library(tidyverse)

    ## ── Attaching packages ─────────────────────────────────────────────────────────────────── tidyverse 1.2.1 ──

    ## ✔ ggplot2 3.1.0       ✔ purrr   0.3.1  
    ## ✔ tibble  2.0.1       ✔ dplyr   0.8.0.1
    ## ✔ tidyr   0.8.3       ✔ stringr 1.4.0  
    ## ✔ readr   1.3.1       ✔ forcats 0.4.0

    ## ── Conflicts ────────────────────────────────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ dplyr::filter() masks stats::filter()
    ## ✖ dplyr::lag()    masks stats::lag()

Data Wrangling
==============

countData and geneinfo
----------------------

    # import count data, set rows at entreziz
    countData <- read.table("../results/kallistocounts.txt", 
                            sep = ",", row.names = NULL)

    ## set row names as entrezid for deseq2
    row.names(countData) <- countData$entrezid

    # replace dashes with periods (to make deseq happy)
    colnames(countData) <- gsub('-', '.', colnames(countData))

    # create df with gene info and remove this from countData
    geneinfo <- countData %>%
      select(row.names, Name, geneid, entrezid)
    head(geneinfo)

    ##                row.names     Name geneid       entrezid
    ## NP_001001127.1    408082    EDNRB 408082 NP_001001127.1
    ## NP_001001129.1    408183  CYP26A1 408183 NP_001001129.1
    ## NP_001001189.1    374073    CFDP1 374073 NP_001001189.1
    ## NP_001001194.1    407777    AvBD7 407777 NP_001001194.1
    ## NP_001001195.1    407779     KRT5 407779 NP_001001195.1
    ## NP_001001201.1    408034 HSD11B1L 408034 NP_001001201.1

    countData <- countData %>%
      select (-c(row.names, Name, geneid, entrezid))

    # because DESeq require integers
    countData <- round(countData)

    # print tolal num of genes and samples
    dim(countData)

    ## [1] 14937   987

    # make a tibble for the sole purpose of previewing this huge df
    countDataTibble <- as.tbl(countData)
    print(countDataTibble, n_extra = 0)

    ## # A tibble: 14,937 x 987
    ##    L.Blu13_male_go… L.Blu13_male_hy… L.Blu13_male_pi… L.G107_male_gon…
    ##               <dbl>            <dbl>            <dbl>            <dbl>
    ##  1                6               37               46                7
    ##  2                1                0               47                2
    ##  3              408               72              197              728
    ##  4                1                0                0                1
    ##  5                0                0                0                2
    ##  6               65                3               30              147
    ##  7               12                1                1                0
    ##  8               38               21                1               97
    ##  9                2              142               12                0
    ## 10                3                7                4                4
    ## # … with 14,927 more rows, and 983 more variables

    #rm(countDataTibble)

wrangle colData
---------------

    colData <- read.table(file.path( "../metadata/kallistosamples.txt"), header = F, stringsAsFactors = F)

    # use strsplit to cut the filename into meaningful columns
    colData$bird <- sapply(strsplit(as.character(colData$V1),'\\_'), "[", 1)
    colData$sex <- sapply(strsplit(as.character(colData$V1),'\\_'), "[", 2)
    colData$tissue <- sapply(strsplit(as.character(colData$V1),'\\_'), "[", 3)

    # to create columns with variable names in the filename 
    colData$temp <- sapply(strsplit(as.character(colData$V1),'\\_'), "[", 4)
    colData$treatmentTemp <- sapply(strsplit(as.character(colData$temp),'\\.'), "[", 1)
    colData$NYNO <- sapply(strsplit(as.character(colData$temp),'\\.'), "[", 2)

    # relabel incorrectly named variables 
    colData$treatment <- ifelse(grepl("extend-hatch", colData$treatmentTemp), "extend",
                                ifelse(grepl("inc-prolong", colData$treatmentTemp), "prolong",
                                       ifelse(grepl("m.hatch", colData$treatmentTemp), "m.n2",
                                       colData$treatmentTemp)))

    # drop no-longer-needed cols
    colData <- colData %>% dplyr::select(-c(temp, treatmentTemp, NYNO))

    # replace dashes with periods and make rowname, (to make deseq happy)
    colData$bird <- gsub("-", ".", colData$bird)
    colData$treatment <- gsub("-", ".", colData$treatment)
    colData$V1 <- gsub("-", ".", colData$V1)
    row.names(colData) <- colData$V1

    # make variables factors and specify level
    cols <- c("sex", "treatment", "tissue")
    colData[cols] <- lapply(colData[cols], factor) 
    colData$tissue <- factor(colData$tissue, 
                             levels = c("hypothalamus", "pituitary", "gonad"))

    # create a combinatorial variable
    colData$group <- paste(colData$sex, colData$tissue, colData$treatment, sep=".")

    # add variable to distinguish characterization study and manipulation
    colData$study <- ifelse(grepl("m.|extend|prolong", colData$treatment), "manipulation", "charcterization")

    colData$study <- factor(colData$study, 
                             levels = c("charcterization", "manipulation"))

    ## view colData
    str(colData)

    ## 'data.frame':    987 obs. of  7 variables:
    ##  $ V1       : chr  "L.Blu13_male_gonad_control.NYNO" "L.Blu13_male_hypothalamus_control.NYNO" "L.Blu13_male_pituitary_control.NYNO" "L.G107_male_gonad_control" ...
    ##  $ bird     : chr  "L.Blu13" "L.Blu13" "L.Blu13" "L.G107" ...
    ##  $ sex      : Factor w/ 2 levels "female","male": 2 2 2 2 2 2 1 1 1 2 ...
    ##  $ tissue   : Factor w/ 3 levels "hypothalamus",..: 3 1 2 3 1 2 3 1 2 3 ...
    ##  $ treatment: Factor w/ 16 levels "bldg","control",..: 2 2 2 2 2 2 2 2 2 2 ...
    ##  $ group    : chr  "male.gonad.control" "male.hypothalamus.control" "male.pituitary.control" "male.gonad.control" ...
    ##  $ study    : Factor w/ 2 levels "charcterization",..: 1 1 1 1 1 1 1 1 1 1 ...

    head(colData, 5)

    ##                                                                            V1
    ## L.Blu13_male_gonad_control.NYNO               L.Blu13_male_gonad_control.NYNO
    ## L.Blu13_male_hypothalamus_control.NYNO L.Blu13_male_hypothalamus_control.NYNO
    ## L.Blu13_male_pituitary_control.NYNO       L.Blu13_male_pituitary_control.NYNO
    ## L.G107_male_gonad_control                           L.G107_male_gonad_control
    ## L.G107_male_hypothalamus_control             L.G107_male_hypothalamus_control
    ##                                           bird  sex       tissue treatment
    ## L.Blu13_male_gonad_control.NYNO        L.Blu13 male        gonad   control
    ## L.Blu13_male_hypothalamus_control.NYNO L.Blu13 male hypothalamus   control
    ## L.Blu13_male_pituitary_control.NYNO    L.Blu13 male    pituitary   control
    ## L.G107_male_gonad_control               L.G107 male        gonad   control
    ## L.G107_male_hypothalamus_control        L.G107 male hypothalamus   control
    ##                                                            group
    ## L.Blu13_male_gonad_control.NYNO               male.gonad.control
    ## L.Blu13_male_hypothalamus_control.NYNO male.hypothalamus.control
    ## L.Blu13_male_pituitary_control.NYNO       male.pituitary.control
    ## L.G107_male_gonad_control                     male.gonad.control
    ## L.G107_male_hypothalamus_control       male.hypothalamus.control
    ##                                                  study
    ## L.Blu13_male_gonad_control.NYNO        charcterization
    ## L.Blu13_male_hypothalamus_control.NYNO charcterization
    ## L.Blu13_male_pituitary_control.NYNO    charcterization
    ## L.G107_male_gonad_control              charcterization
    ## L.G107_male_hypothalamus_control       charcterization

    # check that rownames and colnames match for DESeq
    ncol(countData) == nrow(colData)

    ## [1] TRUE

    colData %>% select(sex, tissue, treatment, study)  %>%  summary()

    ##      sex               tissue        treatment               study    
    ##  female:497   hypothalamus:327   control  : 73   charcterization:576  
    ##  male  :490   pituitary   :330   inc.d9   : 71   manipulation   :411  
    ##               gonad       :330   inc.d17  : 66                        
    ##                                  n9       : 66                        
    ##                                  m.inc.d17: 63                        
    ##                                  bldg     : 60                        
    ##                                  (Other)  :588

save bird and sample and count data
-----------------------------------

    samples <- colData

    birds <- colData %>%
      select(bird, sex, treatment) %>%
      distinct()

subset for manipulation study
-----------------------------

    # create a new dfs for just the manipulation study
    colData_manipluation <- colData %>%
      dplyr::filter(grepl('m.|extend|prolong', treatment)) %>%
      droplevels()
    row.names(colData_manipluation) <- colData_manipluation$V1

    # print sample sizes
    colData_manipluation %>% select(sex,treatment, tissue)  %>%  summary()

    ##      sex          treatment           tissue   
    ##  female:208   extend   :60   hypothalamus:138  
    ##  male  :203   m.inc.d17:63   pituitary   :137  
    ##               m.inc.d3 :60   gonad       :136  
    ##               m.inc.d8 :60                     
    ##               m.inc.d9 :49                     
    ##               m.n2     :59                     
    ##               prolong  :60

    # create list with filenames and select countData columns matching the file names
    savecols <- as.character(colData_manipluation$V1) 
    savecols <- as.vector(savecols) 
    countData_manipluation <- countData %>% dplyr::select(one_of(savecols)) 

    # confirm that row and col lenghts are equal
    ncol(countData_manipluation) == nrow(colData_manipluation)

    ## [1] TRUE

subset for characterization study
---------------------------------

    # filter out data related to the manipulation study
    colData_characterization <- colData %>%
      dplyr::filter(!grepl('m.|extend|prolong', treatment)) %>%
      droplevels()
    row.names(colData_characterization) <- colData_characterization$V1

    # print sample sizes
    colData_characterization %>% select(sex,treatment, tissue)  %>%  summary()

    ##      sex        treatment            tissue   
    ##  female:289   control: 73   hypothalamus:189  
    ##  male  :287   inc.d9 : 71   pituitary   :193  
    ##               inc.d17: 66   gonad       :194  
    ##               n9     : 66                     
    ##               bldg   : 60                     
    ##               hatch  : 60                     
    ##               (Other):180

    savecols <- as.character(colData_characterization$V1) 
    savecols <- as.vector(savecols) 
    countData_characterization <- countData %>% dplyr::select(one_of(savecols)) 

    # check that row and col lenghts are equal
    ncol(countData_characterization) == nrow(colData_characterization)

    ## [1] TRUE

save files for downstream use
-----------------------------

    write.csv(countData, "../results/00_counts.csv")
    write.csv(samples, "../metadata/00_samples.csv")
    write.csv(birds, "../metadata/00_birds.csv")

    write.csv(colData_manipluation, "../metadata/00_colData_manipluation.csv", row.names = TRUE) 
    write.csv(countData_manipluation, "../results/00_countData_manipluation.csv", row.names = TRUE) 
    write.csv(colData_characterization, "../metadata/00_colData_characterization.csv", row.names = TRUE) 
    write.csv(countData_characterization, "../results/00_countData_characterization.csv", row.names = TRUE) 

    write.csv(geneinfo, "../metadata/00_geneinfo.csv", row.names = TRUE)
