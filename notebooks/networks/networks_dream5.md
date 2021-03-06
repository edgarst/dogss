Network reconstruction with dogss: DREAM5
================
Edgar Steiger
2018

-   [ROC/PR analysis](#rocpr-analysis)
-   [different groupings](#different-groupings)

**work in progress**

This document shows and explains how to use the dogss package and how to reproduce Figures 14 and 15 from our paper [Sparse-Group Bayesian Feature Selection Using Expectation Propagation for Signal Recovery and Network Reconstruction](https://arxiv.org/abs/1809.09367).

First we need to load some packages that are required for comparisons and plotting (please install if not available on your machine):

``` r
library(dogss) # our method for sparse-group Bayesian feature selection with EP
library(glmnet) # standard lasso
library(gglasso) # group lasso
library(SGL) # sparse-group lasso
library(MBSGS) # Bayesian feature selection with Gibbs sampling

library(ggplot2) # for nice plots
library(ggthemes) # for even nicer plots
library(grid); library(gridExtra) # to arrange plots pleasantly

library(reshape2) # to melt data into "tidy" long-format

library(DescTools) # for area computations (AUROC, AUPR)

library("foreach")
library("doParallel")
library(parallel)
```

Furthermore we need to load three R files with additional code:

``` r
source("../auxiliary_rfunctions/my_cvSGL.R") # proper cross validation for SGL package

source("../auxiliary_rfunctions/my_theme.R") # functions to adjust ggplots
```

Finally, we provide all of the results on our simulated data to reconstruct the plots from the publication. If you wish to re-do all of the simulations/calculations, change the following parameter to `TRUE` (only do this if you have access to multiple cores):

``` r
selfcompute <- FALSE
B <- 100 # number of simulations
ncores <- 50 # number of cores used for parallelization
```

ROC/PR analysis
---------------

``` r
nobs_train <- c(100, 300, 500)

geneexpr <- as.matrix(read.table(file="/project/dogss/datasets/dream5/net3_expression_data.tsv", header=TRUE))
tfs <- read.table(file="/project/dogss/datasets/dream5/net3_transcription_factors.tsv", colClasses="character")$V1
truth <- read.table(file="/project/dogss/datasets/dream5/DREAM5_NetworkInference_GoldStandard_Network3", header=FALSE, colClasses=c("character", "character", "integer"))
truth <- truth[truth$V3==1, c(1,2)]
load("/project/dogss/datasets/dream5/G_firstrun.RData")
load("/project/dogss/datasets/dream5/wisdomG.RData")

#####
# net3_expression_data <- read.delim("/run/user/1000/gvfs/sftp:host=geniux.molgen.mpg.de,user=steige_e/project/dogss/datasets/dream5/net3_expression_data.tsv")
# geneexpr <- net3_expression_data
# net3_transcription_factors <- read.table("/run/user/1000/gvfs/sftp:host=geniux.molgen.mpg.de,user=steige_e/project/dogss/datasets/dream5/net3_transcription_factors.tsv", quote="\"", comment.char="", stringsAsFactors=FALSE)
# tfs <- net3_transcription_factors$V1
# DREAM5_NetworkInference_GoldStandard_Network3 <- read.delim("/run/user/1000/gvfs/sftp:host=geniux.molgen.mpg.de,user=steige_e/project/dogss/datasets/dream5/DREAM5_NetworkInference_GoldStandard_Network3", header=FALSE, stringsAsFactors=FALSE)
# truth <- DREAM5_NetworkInference_GoldStandard_Network3[DREAM5_NetworkInference_GoldStandard_Network3$V3==1, c(1,2)]
# load("/run/user/1000/gvfs/sftp:host=geniux.molgen.mpg.de,user=steige_e/project/dogss/datasets/dream5/G_firstrun.RData")
# load("/run/user/1000/gvfs/sftp:host=geniux.molgen.mpg.de,user=steige_e/project/dogss/datasets/dream5/wisdomG.RData")

geneexpr_center <- scale(geneexpr, scale=FALSE)
geneexpr_scale <- scale(geneexpr)

geneexpr_center_full <- geneexpr_center
geneexpr_scale_full <- geneexpr_scale

myG <- as.integer(Gold)
names(myG) <- tfs
genes <- colnames(geneexpr)
wisdomG <- moduleG[tfs]
names(wisdomG) <- tfs
myrandomG <- sample(myG, length(myG))
names(myrandomG) <- tfs
myclusterG <- kmeans(t(geneexpr_scale_full[, tfs]), centers=length(unique(myG)))$cluster
wisdomrandomG <- sample(wisdomG, length(wisdomG))
names(wisdomrandomG) <- tfs
wisdomclusterG <- kmeans(t(geneexpr_scale_full[, tfs]), centers=length(unique(wisdomG)))$cluster

Gs <- cbind(myG, myclusterG, myrandomG, wisdomG, wisdomclusterG, wisdomrandomG)

nfeats <- dim(geneexpr)[2]
nobs <- dim(geneexpr)[1]
ntfs <- length(tfs)
neffs <- dim(truth)[1]
truth_matrix <- matrix(0, nrow=nfeats, ncol=nfeats, dimnames=list(genes1=colnames(geneexpr), genes2=colnames(geneexpr)))
for (i in 1:neffs) {
  truth_matrix[truth[i, 1], truth[i, 2]] <- 1
  truth_matrix[truth[i, 2], truth[i, 1]] <- 1
}
neffs <- sum(truth_matrix)/2

for (j in seq(nobs_train)) {
  cat("\n nobs: ", nobs_train[j], "\n")
  for (run in 1:10) {
    cat("\n run: ", run, "\n")
    train <- sample(seq(nobs), nobs_train[j])
    test <- seq(nobs)[-train]
    
    geneexpr_center_train <- geneexpr_center_full[train, ]
    geneexpr_center_test <- geneexpr_center_full[test, ]
    geneexpr_scale_train <- geneexpr_scale_full[train, ]
    geneexpr_scale_test <- geneexpr_scale_full[test, ]
    save(train, test, file=paste0("/project/dogss/tests_thesis/03_dream5/train_test_run_", run, "_nobs_", nobs_train[j], "_dream5.RData"))
    
    ncl <- detectCores()
    ncl <- min(ncl, ncores)
    myCluster <- makeCluster(ncl, outfile="/project/dogss/tests_thesis/dream5.txt")
    registerDoParallel(myCluster)
    results_dream5 <- foreach (g=genes, .packages=c("dogss2", "SGL", "glmnet", "gglasso", "Matrix")) %dopar% { # "MSGS"
      cat("\n", g, "\n")
      Y <- as.vector(geneexpr_center_train[, g])
      scores <- vector("list", 6)
      results <- vector("list", 6)
      
      for (grouping in 1:6) {
        if (!(g %in% tfs)) {
          index <- Gs[, grouping]
          x <- geneexpr_scale_train[, tfs]
        } else {
          index <- Gs[!(tfs %in% g), grouping]
          x <- geneexpr_scale_train[, tfs][, !(tfs %in% g)]
        }
        
        groups <- unique(index)
        ngroups <- length(groups)
        nG <- ngroups
        nngroups <- sapply(groups, function(g) sum(index==g))
        
        while (max(groups) != ngroups) { # this is for gglasso which needs consecutively numbered groups :-/
          del <- max(groups)
          index[index==del] <- which.min(1:nG %in% groups) # this is superweird :D
          groups <- unique(index)
          ngroups <- length(groups)
          nngroups <- sapply(groups, function(g) sum(index==g))
        }
        
        res_dogss <- cv_dogss(x, Y, G=index, tol=1e-03)
        res_ssep <- cv_dogss(x, Y, G=NULL, tol=1e-03)
        res_sgl <- my_cvSGL(data=list(x=x, y=Y), index=index, standardize=FALSE, thresh=1e-03, maxit=1000)
        res_gglasso <- cv.gglasso(x=x, y=Y, group=index, nfolds=10, intercept=FALSE, eps=1e-03, maxit=1000)
        res_lasso <- cv.glmnet(x, Y, standardize=FALSE, intercept=FALSE, thresh=1e-03, maxit=1000)
        
        res_sgl$lambda <- res_sgl$fit$lambdas
        
        if (is.na(res_gglasso$lambda.1se)) {
          whichmin <- which.min(res_gglasso$cvm)
          res_gglasso$lambda.1se <- res_gglasso$lambda[which.max(res_gglasso$cvm<=res_gglasso$cvm[whichmin]+res_gglasso$cvsd[whichmin])]
        }
        
        new_res_gglasso <- list(lambda=res_gglasso$lambda, gglasso.fit=list(beta=as(res_gglasso$gglasso.fit$beta, "dgCMatrix")), lambda.1se=res_gglasso$lambda.1se)
        new_res_lasso <- list(lambda=res_lasso$lambda, glmnet.fit=list(beta=res_lasso$glmnet.fit$beta), lambda.1se=res_lasso$lambda.1se)
        new_res_sgl <- list(lambda=res_sgl$lambda, beta=res_sgl$beta, sgl.fit=list(beta=as(res_sgl$fit$beta, "dgCMatrix")), lambda.1se=res_sgl$lambda_1se)
        
        res_gglasso <- new_res_gglasso
        res_lasso <- new_res_lasso
        res_sgl <- new_res_sgl
        
        res_sgl$mylambda <- apply(res_sgl$sgl.fit$beta, 1, function(b) res_sgl$lambda[which(b!=0)[1]])
        res_gglasso$mylambda <- apply(res_gglasso$gglasso.fit$beta, 1, function(b) res_gglasso$lambda[which(b!=0)][1])
        res_lasso$mylambda <- apply(res_lasso$glmnet.fit$beta, 1, function(b) res_lasso$lambda[which(b!=0)[1]])
        
        scores[[grouping]] <- cbind(dogss=res_dogss$p_final, ssep=res_ssep$p_final, sgl=res_sgl$mylambda, gglasso=res_gglasso$mylambda, lasso=res_lasso$mylambda) # ...$areascore # ...$p_final # ,
        row.names(scores[[grouping]]) <- names(index)
        results[[grouping]] <- list(res_dogss=res_dogss, res_ssep=res_ssep, res_sgl=res_sgl, res_gglasso=res_gglasso, res_lasso=res_lasso)
      }
      scoresresults <- list(scores=scores, results=results)
      save(scoresresults, file=paste0("/project/dogss/tests_thesis/03_dream5/scoresresults_gene_", g, "_run_", run, "_nobs_", nobs_train[j], ".RData"))
      scoresresults
    }
    stopCluster(myCluster)
    save.image(paste0("/project/dogss/tests_thesis/03_dream5/finalresults_run_", run, "_nobs_", nobs_train[j], ".RData"))
  }
}
```

``` r
  load("/project/dogss/tests_thesis/03_dream5_analysis/average_dream5_results_300_1.RData")
# FPR <- FPR[, 1:4]
# TPR <- TPR[, 1:4]
# PREC <- PREC[, 1:4]
colnames(FPR) <- methods #[1:4]
colnames(TPR) <- methods #[1:4]
colnames(PREC) <- methods #[1:4]
N <- dim(FPR)[1]
mygrid <- c(1:2000, 2:1450*1000+1)
myranks <- c(0:2000, 2000+1000*1:18)
data <- data.frame(method=melt(FPR[mygrid, ])[, 2], fpr=melt(FPR[mygrid, ])[, 3], tpr=melt(TPR[mygrid, ])[, 3], precision=melt(PREC[mygrid, ])[, 3])
data <- rbind(data, data.frame(method="random", fpr=c(0,0.25,1), tpr=c(0,0.25,1), precision=c(0.0014, 0.0014, 0.0014)))
data$method <- factor(data$method, levels = c(methods, "random"), ordered = TRUE)

colnames(PREDERROR) <- methods
rownames(PREDERROR) <- myranks
data_pred <- melt(PREDERROR)
data_pred$method <- factor(data_pred$method, levels = methods, ordered = TRUE)

Ed_palette_full <- Ed_palette
Ed_palette <- Ed_palette[c(1:5, 7)]

myROC <- ggplot(data, aes(x=fpr, y=tpr, color=method)) +
  geom_line(aes(linetype=method), size=1) +
  theme_Ed() +
  scale_colour_Ed() +
  scale_linetype_manual(values=c(rep("solid", 5), "dashed")) +
  # geom_segment(aes(x=0, y=0, xend=1, yend=1, colour="random"), linetype="dashed", colour=Ed_palette[7]) +
  labs(title= "ROC curve", x = "FPR", y = "TPR")

myPR <- ggplot(data, aes(x=tpr, y=precision, color=method)) +
  geom_line(aes(linetype=method), size=1) +
  theme_Ed() +
  scale_colour_Ed() +
  scale_linetype_manual(values=c(rep("solid", 5), "dashed")) +
  # geom_segment(aes(x=0, y=2055/N, xend=1, yend=2055/N, colour="random"), linetype="dashed", colour=Ed_palette[7]) +
  ylim(0,1) +
  labs(title= "Precision-Recall curve", x = "TPR", y = "Precision")

myPR_zoom <- ggplot(data, aes(x=tpr, y=precision, color=method)) +
  geom_line(aes(linetype=method), size=1) +
  theme_Ed() +
  scale_colour_Ed() +
  scale_linetype_manual(values=c(rep("solid", 5), "dashed")) +
  # geom_segment(aes(x=0, y=2055/N, xend=0.25, yend=2055/N, colour="random"), linetype="dashed", colour=Ed_palette[5]) +
  ylim(0, 0.5) +
  xlim(0, 0.25) +
  labs(title= "PR curve: zoomed", x = "TPR", y = "Precision")

myPred <- ggplot(data_pred, aes(x=rank, y=value, color=method)) +
  geom_line(size=1) +
  theme_Ed() +
  scale_colour_Ed() +
  ylim(0, max(PREDERROR)) +
  labs(title= "Prediction on test set", x = "number of edges", y = "Prediction error")

grid_arrange_shared_legend(myROC, myPR, myPR_zoom, myPred, ncol = 2, nrow = 2) #
Ed_palette <- Ed_palette_full
```

different groupings
-------------------

``` r
myranks <- c(0:2000, 2000+1000*1:18)
methods <- c("dogss", "ssep", "sgl", "gglasso", "lasso") #
mynobs_train <- 300 # c(100, 300, 500)
groupings <- 1:4 # 1:6

for (run in 3:10) {
  cat("run", run, "\n")
  #for (mynobs in seq(mynobs_train)) {
    cat("nobs", mynobs_train, "\n")
    # load(paste0("/project/dogss/tests_thesis/03_dream5/nobs_100/train_test_run_", run, "_nobs_100_dream5.RData"))
    load(paste0("/project/dogss/tests_thesis/03_dream5/nobs_", mynobs_train, "/finalresults_run_", run, "_nobs_", mynobs_train, ".RData"))
    for (gr in groupings) {
      cat("grouping", gr, "\n")
      allscores <- matrix(0, nrow=nfeats*ntfs-ntfs, ncol=5)
      pairs <- matrix("", nrow=nfeats*ntfs-ntfs, ncol=2)
      mia <- 1
      mie <- 0
      betabeta <- vector("list")
      for (g in 1:length(genes)) {
        mie <- mie+dim(results_dream5[[g]]$scores[[gr]])[1]
        pairs[mia:mie, ] <- cbind(rep(genes[g], dim(results_dream5[[g]]$scores[[gr]])[1]), as.character(rownames(results_dream5[[g]]$scores[[gr]])))
        allscores[mia:mie, ] <- results_dream5[[g]]$scores[[gr]]
        mia <- mia+dim(results_dream5[[g]]$scores[[gr]])[1]
        betas <- matrix(0, ncol=5, nrow=length(tfs), dimnames=list(tfs=tfs, method=methods))
        betas[!(tfs %in% genes[g]), "dogss"] <- results_dream5[[g]]$results[[gr]]$res_dogss$m_final
        betas[!(tfs %in% genes[g]), "ssep"] <- results_dream5[[g]]$results[[gr]]$res_ssep$m_final
        betas[!(tfs %in% genes[g]), "sgl"] <- results_dream5[[g]]$results[[gr]]$res_sgl$beta
        betas[!(tfs %in% genes[g]), "gglasso"] <- results_dream5[[g]]$results[[gr]]$res_gglasso$gglasso.fit$beta[, which.max(results_dream5[[g]]$results[[gr]]$res_gglasso$lambda==results_dream5[[g]]$results[[gr]]$res_gglasso$lambda.1se)]
        betas[!(tfs %in% genes[g]), "lasso"] <- results_dream5[[g]]$results[[gr]]$res_lasso$glmnet.fit$beta[, which.max(results_dream5[[g]]$results[[gr]]$res_lasso$lambda==results_dream5[[g]]$results[[gr]]$res_lasso$lambda.1se)]
        betabeta[[genes[g]]] <- betas
      }
      
      save(allscores, run, gr, file=paste0("/project/dogss/tests_thesis/03_dream5_analysis/allscores_", run, "_", mynobs_train, "_", gr, ".RData"))
      
      K <- (nfeats-ntfs)*ntfs+(ntfs-1)*ntfs/2
      P <- sum(truth_matrix)/2
      N <- K-P
      
      ranks <- apply(allscores, 2, function(r) rank(1-r, ties.method="min"))
      colnames(ranks) <- methods
      recov_matrices <- list()
      for (m in methods) recov_matrices[[m]] <- matrix(0, nrow=dim(truth_matrix)[1], ncol=dim(truth_matrix)[2], dimnames=list(from=rownames(truth_matrix), to=colnames(truth_matrix)))
      if (gr==1) {
        for (m in methods) {
          for (j in seq(dim(pairs)[1])) {
            if (recov_matrices[[m]][pairs[j,2], pairs[j,1]]==0 | recov_matrices[[m]][pairs[j,2], pairs[j,1]] > ranks[j, m]) {
              recov_matrices[[m]][pairs[j,2], pairs[j,1]] <- ranks[j, m]
              recov_matrices[[m]][pairs[j,1], pairs[j,2]] <- ranks[j, m]
            }
          }
        }
      } else {
        for (m in methods[c(1,3,4)]) {
          for (j in seq(dim(pairs)[1])) {
            if (recov_matrices[[m]][pairs[j,2], pairs[j,1]]==0 | recov_matrices[[m]][pairs[j,2], pairs[j,1]] > ranks[j, m]) {
              recov_matrices[[m]][pairs[j,2], pairs[j,1]] <- ranks[j, m]
              recov_matrices[[m]][pairs[j,1], pairs[j,2]] <- ranks[j, m]
            }
          }
        }
      }
      
      save(ranks, run, gr, file=paste0("/project/dogss/tests_thesis/03_dream5_analysis/ranks_", run, "_", mynobs_train, "_", gr, ".RData"))
      
      M <- length(methods)
      
      tpr <- matrix(0, K, M)
      fpr <- matrix(0, K, M)
      precision <- matrix(0, K, M)
      
      logicalK <- matrix(FALSE, nrow=nfeats, ncol=nfeats)
      logicalK[1:ntfs, (ntfs+1):nfeats] <- TRUE
      logicalK[1:ntfs, 1:ntfs][upper.tri(logicalK[1:ntfs, 1:ntfs])] <- TRUE
      
      for (m in 1:M) {
        truth_ordered <- truth_matrix[logicalK][order(recov_matrices[[m]][logicalK])]
        tpr[, m] <- cumsum(truth_ordered)/P
        fpr[, m] <- cumsum(!truth_ordered)/N
        precision[, m] <- tpr[, m]*P/(tpr[, m]*P + fpr[, m]*N)
      }
      AUROC <- sapply(1:length(methods), function(m) AUC(c(0,fpr[, m],1), c(0,tpr[, m],1)))
      AUPR <- sapply(1:length(methods), function(m) AUC(c(0,tpr[, m],1), c(1,precision[, m],0)))
      
      save(AUROC, AUPR, tpr, fpr, precision, run, gr, file=paste0("/project/dogss/tests_thesis/03_dream5_analysis/aucs_", run, "_", mynobs_train, "_", gr, ".RData"))
      
      # prediction
      Y <- geneexpr_center_test[, !(colnames(geneexpr_center_test)%in%tfs)]
      sumY2 <- sum(Y^2)
      X <- geneexpr_scale_test[, tfs]
      BETA <- vector("list")
      RANKS <- vector("list")
      prederror <- matrix(0, nrow=length(myranks), ncol=5, dimnames=list(rank=myranks, method=methods))
      if (gr==1) {
        for (m in methods) {
          cat(m,"\n")
          BETA[[m]] <- do.call(cbind, lapply(betabeta, function(b) b[, m]))
          BETA[[m]] <- BETA[[m]][, !(colnames(geneexpr_center_test)%in%tfs)]
          RANKS[[m]] <- matrix(max(ranks)+1, nrow=ntfs, ncol=nfeats)
          RANKS[[m]][-c(0:(ntfs-1)*(ntfs+1)+1)] <- ranks[, m]
          RANKS[[m]] <- RANKS[[m]][, !(colnames(geneexpr_center_test)%in%tfs)]
          
          cl <- makeCluster(no_cores)
          clusterExport(cl, c("Y", "X", "RANKS", "myranks", "BETA", "m", "sumY2"))
          prederror[, m] <- parSapply(cl, seq(myranks), function(r) sum((Y-X%*%ifelse(RANKS[[m]]>myranks[r], 0, BETA[[m]]))^2)/sumY2)
          stopCluster(cl)
        }
      }
      save(prederror, run, gr, file=paste0("/project/dogss/tests_thesis/03_dream5_analysis/prederror_", run, "_", mynobs_train, "_", gr, ".RData"))
      
    }
 # }
}
##### average the curves
#for (mynobs in seq(mynobs_train)) {
  cat("averages nobs", mynobs_train, "\n")
  for (gr in groupings) {
    cat("averages grouping", gr, "\n")
    load(file=paste0("/project/dogss/tests_thesis/03_dream5_analysis/aucs_", 1, "_", mynobs_train, "_", gr, ".RData"))
    load(file=paste0("/project/dogss/tests_thesis/03_dream5_analysis/prederror_", 1, "_", mynobs_train, "_", gr, ".RData"))
    TPR <- tpr
    FPR <- fpr
    PREC <- precision
    PREDERROR <- prederror
    allAUROC <- matrix(0, nrow=10, ncol=5)
    allAUPR <- matrix(0, nrow=10, ncol=5)
    allAUROC[1, ] <- AUROC
    allAUPR[1, ] <- AUPR
    for (run in 2:10) {
      load(file=paste0("/project/dogss/tests_thesis/03_dream5_analysis/aucs_", run, "_", mynobs_train, "_", gr, ".RData"))
      load(file=paste0("/project/dogss/tests_thesis/03_dream5_analysis/prederror_", run, "_", mynobs_train, "_", gr, ".RData"))
      allAUROC[run, ] <- AUROC
      allAUPR[run, ] <- AUPR
      TPR <- TPR + tpr
      FPR <- FPR + fpr
      PREC <- PREC + precision
      PREDERROR <- PREDERROR + prederror
    }
    TPR <- TPR/10
    FPR <- FPR/10
    PREC <- PREC/10
    PREDERROR <- PREDERROR/10
    meaned_auroc <- sapply(1:length(methods), function(m) AUC(FPR[, m], TPR[, m]))
    meaned_aupr <- sapply(1:length(methods), function(m) AUC(TPR[, m], PREC[, m]))
    save(TPR, FPR, PREC, PREDERROR, allAUPR, allAUROC, meaned_aupr, meaned_auroc, methods, file="/project/dogss/tests_thesis/03_dream5_analysis/average_dream5_results_", mynobs_train, "_", gr, ".RData")
  }
#}
cat("done\n")
```

``` r
  load("/project/dogss/testing/withpred_dream5_withpred_diffgroups_whole_final.RData")
  old_FPR <- FPR[, 1]
  old_TPR <- TPR[, 1]
  old_PREC <- PREC[, 1]
  load("~/Programme/dogss/testing/dream5_withpred_diffgroups_whole_averaged_10.RData")
  FPR <- cbind(old_FPR, FPR)
  TPR <- cbind(old_TPR, TPR)
  PREC <- cbind(old_PREC, PREC)
  methods <- c("co-binding", "unif. random", "perm. groups", "kmeans")
  colnames(FPR) <- methods
  colnames(TPR) <- methods
  colnames(PREC) <- methods
  N <- dim(FPR)[1]
  mygrid <- c(1:2000, 2:1500*1000+1)
  data <- data.frame(method=melt(FPR[mygrid, ])[, 2], fpr=melt(FPR[mygrid, ])[, 3], tpr=melt(TPR[mygrid, ])[, 3], precision=melt(PREC[mygrid, ])[, 3])

  myROC <- ggplot(data, aes(x=fpr, y=tpr, color=method)) +
    geom_line(size=1.3) +
    theme_Ed() +
    scale_colour_Ed() +
    geom_segment(aes(x=0, y=0, xend=1, yend=1, colour="random"), linetype="dashed", colour=Ed_palette[5]) +
    labs(title= "ROC curve", x = "FPR", y = "TPR")

  myPR_zoom <- ggplot(data, aes(x=tpr, y=precision, color=method)) +
    geom_line(size=1.3) +
    theme_Ed() +
    scale_colour_Ed() +
    geom_segment(aes(x=0, y=2066/N, xend=0.25, yend=2066/N, colour="random"), linetype="dashed", colour=Ed_palette[5]) +
    ylim(0, 0.5) +
    xlim(0, 0.25) +
    labs(title= "PR curve: zoomed", x = "TPR", y = "Precision")

  grid_arrange_shared_legend(myROC, myPR_zoom, ncol = 2, nrow = 1)
```
