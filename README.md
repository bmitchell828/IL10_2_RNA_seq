# IL10_2_RNA_seq

Analysis of RNA-seq data from IL10 phase 2. WT and IL10 knockout mice were placed on control or zinc supplemented (25 mM) water for 4 weeks before sacrifice. 

# Workflow 

Prior to importing count data to R studio, demultiplexed sequence data for  3’RNAseq project OMICS4, plate Aydemir_OMICS4, were imported to galaxy (usegalaxy.org) from BioHPCin fastq format. Quality control was conducted with FastQC (Andrews, S. (n.d.). FastQC A Quality Control tool for High Throughput Sequence Data. http://www.bioinformatics.babraham.ac.uk/projects/fastqc/) and summarized with MultiQC (Ewels, P., Magnusson, M., Lundin, S., & KÃ¤ller, M. (2016). MultiQC: summarize analysis results for multiple tools and samples in a single report. Bioinformatics, 32(19), 3047â3048. https://doi.org/10.1093/bioinformatics/btw354). Transcript abundance was quantified using salmon (Patro, R., Duggal, G., Love, M. I., Irizarry, R. A., & Kingsford, C. (2017). Salmon provides fast and bias-aware quantification of transcript expression. Nature Methods, 14(4), 417â419. https://doi.org/10.1038/nmeth.4197) with reference genome Mus musculus (assembly GRCm39). 