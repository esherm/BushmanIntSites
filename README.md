                     Eric's Integration Site Calling Code

This code is designed to take fastq files that are produced by the MiSeq and return integration sites, multihits, and chimeras.  RData and fasta files are used for data persistance in place of a database, thus allowing massive parallelization and the LSF job submission system on the PMACS HPC: http://www.med.upenn.edu/hpc/hardware-physical-environment.html

------------------------------------------------------------------------------

                                    USAGE
                                    
Analysis is started by having the user create the following directory structure:

Primary analysis directory
  |-- sampleInfo.csv
  |-- PMACS_kickoff.R
  |-- Data
      |-- Undetermined_S0_L001_I1_001.fastq.gz
      |-- Undetermined_S0_L001_R1_001.fastq.gz
      |-- Undetermined_S0_L001_R2_001.fastq.gz
      
      
sampleInfo.csv contains sample metadata as described in intSiteValidation/sampleInfo.csv.

PMACS_kickoff.R contains additional metadata parameters that must be set by the user if the user desires to process multiple integration site runs in parallel.

Data/Undetermined... are the fastq files returned by the MiSeq

After creating the directory structure, the following command is issued from within the primary analysis directory:

bsub -n1 -q normal -J "BushmanKickoff_analysis" -o logs/kickoffOutput.txt Rscript PMACS_kickoff.R

The rest of the processing is fully automated and shouldn't take more than 4 hours to process 1.5e7 raw reads.

Temporary files and logs are currently stored by default.  Theses files can be automatically removed post-run by adjusting the 'cleanup' variable in PMACS_kickoff.R.
                                
------------------------------------------------------------------------------

                                    INPUT
                                  
Metadata parameters required in sampleInfo.csv and processingParams.csv.  They can be provided in any order, although I recommend that processingParams contains dry-side analysis values and sampleInfo contains wet-side sample metadata.  Required fields are:

qualityThreshold - ASCII equivalent of minimum quality score for use in sliding-window based read quality trimming

badQualityBases - minimum number of bases below the qualityThreshold that must be observed within qualitySlidingWindow in order for the read to be trimmed

qualitySlidingWindow - the window size used for sliding-window based read quality trimming

primer - 3' sequence of the PCR2 amplification primer

ltrBit - LTR sequence 3' of the PCR2 amplification primer landing site but 5' of the host/provirus junction

largeLTRFrag - reverse compliment of 34nt of LTR sequence immediately upstream of the host/provirus junction - this sequence is used for trimming of vector sequence read-through from short fragments

linkerSequence - unique linker sequence including N's for primerID (if included in the linker/primer scheme ).  This code can process traditional non-primerID linkers as well.

linkerCommon - reverse compliment of the linker common sequence - this sequence is used for trimming of linker sequence read-through from short fragments

mingDNA - minimum number of genomic DNA left after read trimming in order for the read to be passed to the aligner

alias - unique identifier for each barcode/linker pair

vectorSeq - absolute or relative path to the fasta file containing vector sequence to be used for removal of the internal fragment

minPctIdent - minimum percent identity of alignments to be considered as candidate alignments

maxAlignStart - maximum distance that the fist basepair of an alignment can be downstream from the host/provirus junction in order to be considered as a candidate alignment

maxFragLength - maximum fragment size allowed during pairing of read1 and read2 alignments

bcSeq - 12bp error-correcting golay code - as of this time, no other barcode formats are supported

                                    
------------------------------------------------------------------------------

                                    OUTPUT
                                    
INTEGRATION SITES:
This code returns integration sites in two formats.  allSites.RData is a GRanges object that contains a single record for each Illumina read.  sites.final.RData contains a list of dereplicated integration sites along with a tally of how many reads were seen for each site (irregardless of sonic breakpoint).

MULTIHITS:
Multihits are stored in multihitData.RData which is a GRangesList.  The first item in this list is a GRanges object of all multihits, and the second item is a GRanges object of dereplicated multihits, and the third item is a GRanges object of multihit clusters.

CHIMERAS:
Chimeras are stored in chimeraData.RData which is a list that contains some basic chimera frequency statistics and a GRangesList object.  Each GRanges object contains two records, one for the read1 alignment and another for the read2 alignment

PRIMER_IDs:
PrimerIDs (if present in linker sequences) are stored in primerIDData.RData.  This file is a list containing a DNAStringSet and a BStringSet containing the quality scores of the primerIDs.

STATS:
Processing statistics are returned in the stats.RData file.  This file contains a single data.frame named 'stats'.  Detail of the specific columns provided in this dataframe will be added later.

------------------------------------------------------------------------------

                                DEPENDENCIES

This code is highly dependent on Bioconductor packages for processing DNA data and collapsing/expanding alignments.

The sessionInfo() from an environment successfully running this code is included for reference below until specific versioning requirements can be identified.

other attached packages:
 [1] ShortRead_1.24.0        GenomicAlignments_1.2.1 Rsamtools_1.18.2       
 [4] BiocParallel_1.0.0      hiReadsProcessor_0.1.5  xlsx_0.5.7             
 [7] xlsxjars_0.6.1          rJava_0.9-6             hiAnnotator_1.0        
[10] BSgenome_1.34.0         Biostrings_2.34.1       XVector_0.6.0          
[13] plyr_1.8.1              RMySQL_0.9-3            DBI_0.3.1              
[16] rtracklayer_1.26.2      iterators_1.0.7         foreach_1.4.2          
[19] GenomicRanges_1.18.3    GenomeInfoDb_1.2.3      IRanges_2.0.1          
[22] S4Vectors_0.4.0         BiocGenerics_0.12.1     sonicLength_1.4.4   

------------------------------------------------------------------------------

                                CODE STRUCTURE

- Primary read trimming and integration site logic is contained in pairedIntSites.R.
- Files with the "_PMACS" suffix are short R files used to easily allow branching and condensation of parallel processing threads.
- Barcode error correcting logic is performed in golay.py as wrapped by processGolay.py and alignment parameters are contained in BLATsamples.sh.
- All core code files are currently located in the PMACS filesystem at /home/aubreyba/EAS/PMACS_scripts and filepaths are hardcoded into the various R scripts - this will likely be changed

The code structure leaves much room for improvement.

------------------------------------------------------------------------------

                                    TESTS

A sample dataset is included for verification of integration site calling accuracy.  Transfer the intSiteValidation folder to an LSF envrionment and execute the code as described in USAGE.  Upon successful execution , the directory should resemble intSiteValidationOUTPUT.tar.gz.  Temporary files are included in this compressed directory to help users understand the step-by-step specifics of the code.  Note that this subset of data contains samples with some borderline cases.  For example, clone7 samples should all fail, and many of the clone1-4 samples should return no multihits or chimeras.  The current implementation of the code handles these gracefully.