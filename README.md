![logo](/img/logo.png)

# strandedLibraryGrade
How well did your strand-specific RNA-seq library prep work?

Get a score between 0 and 1.




# Detailed Usage

     
	 STRANDED LIBRARY GRADE

                                   [mate2]->
          Forward Strand ----------------------------------------------->
                         <---------------------------------------------- Reverse Strand	 
		                                <-[mate1]


	strandedLibraryGrade 0.0.1

	Usage:

	If you already have the RNA-seq alignments in BAM format:
		strandedLibraryGrade -b /path/to/alignments.bam [-u] 

	If you have transcript FASTA and paired reads produced by dUTP method of RNA-seq:
		strandedLibraryGrade -t /path/to/transcripts.fasta -1 /path/to/R1.fastq.gz -2 /path/to/R2.fastq.gz [-m] [-O prefix] [-P threads] [ -S skip ] [-N nreads ] [-X maxins ]

	If you have transcript FASTA and unpaired reads produced by dUTP method of RNA-seq:
		strandedLibraryGrade -t /path/to/transcripts.fasta -U /path/to/R1.fastq.gz [-m] [-O prefix] [-P threads] [ -S skip ] [-N nreads ] [-X maxins ]

	If you have a config text file specifying file locations of many BAMs or Fastqs (as explained below). All need to be same type (paired or unpaired).
		strandedLibraryGrade -f /path/to/samples.txt [-u] [-m] [-O prefix] [-P threads] [ -S skip ] [-N nreads ] [-X maxins ]
	

	Options:

	-b	BAMMODE : Input BAM. Alignments should be produced in a stranded-naive way. 
			  Thus, where relevant, do not tell alignmer the strandedness and/or orientation you expect. 
			  Can be SAM format as well. SAMtools will aut-detect.
			  Not compatible with -t/-1/-2/-U.
	-u	BAMMODE: BAM was produced by mapping single-end unpaired reads. Default: false (assumed paired). Also may need to use this for FOFNMODE when using file full of single-end BAMs.
	-t	FULLMODE: Bowtie2-indexed transcript (query) sequences. Also need -1/-2 or -U. Not compatible with -b.
			  To make, do: 
				bowtie2-build transcripts.fasta transcripts
			  Provide /Path/to/bt2_index_prefix.
			  In above example that would be /Path/to/transcripts
			  NOTE: also needed in FOFNMODE for fastq files.
	-1	FULLMODE: Mate1 reads from paired reads. Provide: /path/to/R1.fastq.gz. Not compatible with -b and -U.
	-2	FULLMODE: Mate1 reads from paired reads. Provide: /path/to/R1.fastq.gz. Not compatible with -b and -U.
	-U	FULLMODE: Mate1 reads from paired reads. Provide: /path/to/R1.fastq.gz. Not compatible with -b, -1, and -2.
	-m	Optional. Provide /Path/to/desired/bowtie2. Default: looks in environment.
	-O	Output prefix. Outputs are PREFIX.transcript-summaries.txt and PREFIX.final-summary.txt. Default prefix: strandedLibraryGradeOutput.
	-P	Number of parallel threads. Only applies to bowtie2 step in FULLMODE. Default = 2.
	-S	Skip this many reads before processing. Currently only applies to bowtie2 step in FULLMODE. Default = 0.
	-N	Only analyze this many reads. Currently only applies to bowtie2 step in FULLMODE. Default = all (1000000000).
	-X	Max insert size (i.e. max fragment size). Currently only applies to bowtie2 step in FULLMODE with PAIRED reads. Default = 600.
	-f	FOFNMODE: Process multiple samples serially.
			Specify /path/to/samples.txt, a tab-delimited text file with 2-3 columns:
			column1 = output prefix
			column2 = R1 fastq (can be gz) or BAM. Note: if BAM of single-end reads, specify -u. If FASTQ, need to specify -t.
			column3 = R2 fastq. Optional column for FULLMODE. Note: make sure to specify -t.
	-v	Verbose. Say stuff. Only does stuff during FOFN mode at the moment.

	Dependencies:
	- Bowtie2
	- BEDtools
	- SAMtools
	- Awk

	Pipeline notes:
	  FULL-MODE:
	  Internally, Bowtie2 is used in an --end-to-end --very-fast* mode. 
		  * similar to --very-fast but with tweaks to increase speed further.
	  Moreover, bowtie2 is given --no-unal --no-contain --no-mixed --no-discordant.
	  This is done since SAMtools is used anyway to filter for only confidently-mapped concordant read pairs ( -f3 -F4 -q 30).
	  Overall, the goal is to evaluate the quality of the library prep.
	  Things like low MAPQ or discordance can arise due to things like algorithmic choices and reference quality.
	  Hence, we use only high confidence concordant alignments to judge the strandedness of the library.


	Use cases:
	- Created RNA-seq library through the dUTP method (most stranded library prep kits).
		- Expect the strandedness to be ISR as determined by Salmon.
			- I = inward-facing.
			- S = strand-specific.
			- R = mate1 maps to reverse strand (i.e. the reverse complement of the transcript sequence).

	Citation:
	- This script was developed to evaluate the quality of strand-specific library preps (from NEB and Illumina) for studying DNA Puffs in Bradysia coprophila.
	- It is free to use, further develop, criticize, etc.
	- Feel free to cite this github repo if you find it helpful in your own research.



	Output:
		- transcript-summaries.txt
			- For each transcript, it outputs a 4-column line: 
				1. sequence name, 
				2. total number of confidently-mapped concordant read pairs mapping to it, 
				3. number of those pairs in proper strandedness: mate1 on rev strand. 
				4. proportion pairs with mate1 on rev strand. That is col3/col2.
				- This proportion is the strand-specificity score for that particular transcript.
				- It ranges from 0 to 1, 0 being the worst, 1 being perfect.
				- >90% of transcripts should have scores >0.9
					- some transcripts with very few reads mapping to them are often the culprits for having low scores.
			- The last line is the global library score. It is named "all".
				- Columns 2, 3, and 4 are analogous to above only using confidently-mapped concordant read pairs mapping to all transcripts.
		- final-summary.txt
			- This is a 2-column tabular text file: 
				1. metric name
				2. metric score
			- Example:
				-	3535634				## line1: number confidently-mapped concordant pairs with mate1 mapped to reverse strand.
				+	12980				## line2: number confidently-mapped concordant pairs with mate1 mapped to forward strand.   
				pairs	3548614				## line3: total number of confidently-mapped concordant pairs encountered.
				strandbias	0.996342		## line4: line1/line3. should be same as column4 in last line (column1=all) of transcript-summaries.txt.
				perfectBiasTranscripts	9389		## line5: number of transcripts encountered with perfect strand-specificity scores of 1.
				biasedTranscripts	9726		## line6: number of transcripts encountered with strand-specificity scores exceeding 0.9.
				nTranscripts	9993			## line7: total number of transcripts encountered.
				perfectBiasScore	0.939558	## line8: line5/line7. Proportion of transcripts encountered with perfect strand-specificity scores of 1.
				transBiasScore	0.973281		## line9: line6/line7. Proportion of transcripts encountered with strand-specificity scores exceeding 0.9.
			- Brief comments on scores:
				- strandbias:		This is the strandedLibraryGrade. It is a reflection of the strand-specificity of the reads and library itself.
				- perfectBiasScore:	This is a joint-reflection of the library and the quality of the reference transcripts.
				- transBiasScore:	Same as perfectBiasScore, but more lenient.
			- Further info on the scores:
				- All three scores are jointly affected by 
					(i) experimental factors: factors upstream of analysis such as the strand-specificity of the library and sequencing, 
					(ii) introduced factors: algorithmic choices, and reference quality.
				- The strandbias gives all condidently-mapped concordant read pairs equal weights regardless of which transcript they map to.
					- Therefore, strandbias is weighted much more heavily towards experimental factors. 
				- The perfectBiasScore and transBiasScore give all transcripts equal weights. 
					- Thus each transcript strand-specificity score is treated with equal confidence regardless of how few or many read pairs map to it.
						- And regardless of how low or high quality the reference transcript sequence is...
					- This weights the summary scores more heavily towards introduced factors.
					- Indeed, the transcripts with low strand-specificity scores may be suspect that should be flagged for manual annotation review and curation.
						- They may also reflect regions of the genome with genes on both strands.
					- Note that these scores can be interpreted as quantiles or percentiles over the transcripts.
						- perfectBiasScore : the highest quantile one could ask for that would still return 1.
						- transBiasScore   : the highest quantile one could ask for that would still return >0.9.
		


