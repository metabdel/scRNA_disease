Load Data
=========

### Load necessary libraries

    library(tidyverse)

### Load GTEx dataset

The GTEx data (Version 8) was downloaded from
[here](https://gtexportal.org)

    file="../../Data/GTEx/GTEx_Analysis_2017-06-05_v8_RNASeQCv1.1.9_gene_median_tpm.gct"
    exp <- read_tsv(file,skip=2) %>% rename(gene_id=Name)

### Collapse expression per tissue

The gtex\_v8\_tissues.tsv contains the tissues to keep (N&gt;100), as
well as info about which organ each tissue belongs to. We want to keep
the different Brain tissues and not collapsed at the organ level for the
Brain. So we add the different Brain tissues to the organ level column.

    tissues <- read_tsv("../../Data/GTEx/v8/gtex_v8_tissues.tsv") %>% 
      rename(Lvl5=SMTSD,Tissue=SMTS) %>% 
      mutate(Tissue2=ifelse(grepl("Brain",Tissue),Lvl5,Tissue)) %>% 
      select(-Tissue) %>% 
      rename(Tissue=Tissue2)

    ## Parsed with column specification:
    ## cols(
    ##   SMTS = col_character(),
    ##   SMTSD = col_character(),
    ##   SMAFRZE = col_character(),
    ##   Nsample = col_double(),
    ##   drop = col_logical()
    ## )

We tidy the GTEx gene expression data and join with the tissue
information file

    exp <- gather(exp,key = Lvl5,value=Expr,-gene_id,-Description) %>% as.tibble()

    ## Warning: `as.tibble()` is deprecated, use `as_tibble()` (but mind the new semantics).
    ## This warning is displayed once per session.

    exp <- inner_join(exp,tissues)

    ## Joining, by = "Lvl5"

We then drop tissues with less than 100 samples, testis (gene expression
outlier), non natural tissues (e.g. EBV-transformed lymphocytes) and
then take the average expression for the different tissues that belong
to the same organ.

    exp_per_tissue <- exp %>% 
      filter(drop==FALSE) %>% 
      group_by(Tissue,gene_id,Description) %>% 
      summarise(Expr=mean(Expr))

Tidy the data

    exp <- exp_per_tissue %>% spread(Tissue,Expr) %>% ungroup()

### Scale to 1M TPM

    exp_scaled <- apply(exp[-c(1,2)],2,function(x) x*1e6/sum(x))
    exp <- cbind(exp[c(1,2)],exp_scaled) %>% as.tibble()

Only keep genes with a unique name

    exp <- exp %>% add_count(gene_id) %>% 
      filter(n==1) %>%
      select(-n) %>%
      mutate(gene_id=gsub("\\..+","",gene_id)) %>%
      gather(key = Lvl5,value=Expr,-gene_id,-Description) %>% 
      as.tibble()

### Load gene coordinates

Load gene coordinates and extend upstream and downstream by 100kb.

File downloaded from MAGMA website
(<a href="https://ctg.cncr.nl/software/magma" class="uri">https://ctg.cncr.nl/software/magma</a>).

Filtered to remove extended MHC (chr6, 25Mb to 34Mb).

    gene_coordinates <- 
      read_tsv("../../Data/NCBI/NCBI37.3.gene.loc.extendedMHCexcluded",
               col_names = FALSE,col_types = 'cciicc') %>%
      mutate(start=ifelse(X3-100000<0,0,X3-100000),end=X4+100000) %>%
      select(X2,start,end,1) %>% 
      rename(chr="X2", ENTREZ="X1") %>% 
      mutate(chr=paste0("chr",chr))

Filter genes
============

Get table for ENTREZ and ENSEMBL gene names.

    entrez_ensembl <- AnnotationDbi::toTable(org.Hs.eg.db::org.Hs.egENSEMBL)

Only keep genes with a unique entrez and ensembl id.

    entrez_ensembl_unique_genes_entrez <- entrez_ensembl %>% count(gene_id) %>% filter(n==1)
    entrez_ensembl_unique_genes_ens <- entrez_ensembl %>% count(ensembl_id) %>% filter(n==1)
    entrez_ensembl <- filter(entrez_ensembl,gene_id%in%entrez_ensembl_unique_genes_entrez$gene_id & ensembl_id %in% entrez_ensembl_unique_genes_ens$ensembl_id)
    colnames(entrez_ensembl)[1] <- "ENTREZ"
    colnames(entrez_ensembl)[2] <- "Gene"
    gene_coordinates <- inner_join(entrez_ensembl,gene_coordinates) %>% as.tibble()

    exp_lvl5 <- exp %>% rename(Expr_sum_mean=Expr,Gene=gene_id)

### Only keep MAGMA genes

Only keep protein coding genes present in the MAGMA gene file.

    exp_lvl5 <- inner_join(exp_lvl5,gene_coordinates)

### Write dictonary for cell type names

    dic_lvl5 <- exp_lvl5 %>% select(Lvl5) %>% unique() %>% mutate(makenames=make.names(Lvl5))
    write_tsv(dic_lvl5,"../../Data/GTEx/dictionary_cell_type_names.collapsed.txt")

QC
==

### Remove not expressed genes

    not_expressed <- exp_lvl5 %>% group_by(Gene) %>% 
      summarise(total_sum=sum(Expr_sum_mean)) %>% 
      filter(total_sum==0) %>% 
      select(Gene) %>% 
      unique() 

    exp_lvl5 <- filter(exp_lvl5,!Gene%in%not_expressed$Gene)

Specificity Calculation
=======================

The specifitiy is defined as the proportion of total expression
performed by the cell type of interest (x/sum(x)).

    exp_lvl5 <- exp_lvl5 %>% group_by(Gene) %>% 
      mutate(specificity=Expr_sum_mean/sum(Expr_sum_mean)) %>% ungroup()

### Get number of genes

Get number of genes that represent 10% of the dataset

    n_genes <- length(unique(exp_lvl5$ENTREZ))
    n_genes_to_keep <- (n_genes * 0.1) %>% round()

Save expression profile for other processing
============================================

    save(exp_lvl5,file = "expression.ready.Rdata")

### Functions

#### Get MAGMA input top10%

    magma_top10 <- function(d,Cell_type){
      d_spe <- d %>% group_by_(Cell_type) %>% top_n(.,n_genes_to_keep,specificity) 
      d_spe %>% do(write_group_magma(.,Cell_type))
    }

    write_group_magma  = function(df,Cell_type) {
      df <- select(df,Lvl5,ENTREZ)
      df_name <- make.names(unique(df[1]))
      colnames(df)[2] <- df_name  
      dir.create(paste0("MAGMA/"), showWarnings = FALSE)
      select(df,2) %>% t() %>% as.data.frame() %>% rownames_to_column("Cat") %>%
      write_tsv("MAGMA/top10.txt",append=T)
    return(df)
    }

#### Get LDSC input top 10%

    write_group  = function(df,Cell_type) {
      df <- select(df,Lvl5,chr,start,end,ENTREZ)
      dir.create(paste0("LDSC/Bed"), showWarnings = FALSE,recursive = TRUE)
      write_tsv(df[-1],paste0("LDSC/Bed/",make.names(unique(df[1])),".bed"),col_names = F)
    return(df)
    }

    ldsc_bedfile <- function(d,Cell_type){
      d_spe <- d %>% group_by_(Cell_type) %>% top_n(.,n_genes_to_keep,specificity) 
      d_spe %>% do(write_group(.,Cell_type))
    }

### Write MAGMA/LDSC input files

Filter out genes with expression below 1 TPM.

    exp_lvl5 %>% filter(Expr_sum_mean>1) %>% magma_top10("Lvl5")

    ## Warning: group_by_() is deprecated. 
    ## Please use group_by() instead
    ## 
    ## The 'programming' vignette or the tidyeval book can help you
    ## to program with group_by() : https://tidyeval.tidyverse.org
    ## This warning is displayed once per session.

    exp_lvl5 %>% filter(Expr_sum_mean>1) %>% ldsc_bedfile("Lvl5")
