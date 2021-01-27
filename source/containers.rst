Docker / Singularity Containers
###############################

Containers are software environments which make it possible to distrubute  applications along with all the dependencies needed and run these on all computers which have the container engine installed. **Docker** is a well know container platform. For security reasons the Docker engine is not available on Omics. **Singularity** is an alternative to Docker which is better suited for running in a multi-user plaform such as Crunchomics. Singularity is able to deal with Docker containers and many run without problems. 

Singularity Hub
***************

.. code-block:: bash

	singularity pull singularity-images.sif shub://vsoch/singularity-images
	singularity run lolcow.sif 
	singularity run lolcow.sif 

Docker Hub
**********

.. code-block:: bash

	singularity build megahit docker://vout/megahit
	cat /etc/*release
	#CentOS Linux release 7.8.2003 (Core)
	singularity shell megahit
	#DISTRIB_ID=Ubuntu
	#DISTRIB_RELEASE=18.04
	#DISTRIB_CODENAME=bionic
	exit
	rm -rf ~/assembly
	singularity run megahit -v
	singularity run megahit -1 ERR486840_1.fastq.gz -2 ERR486840_2.fastq.gz -o ./assembly
	awk '/^>/ {print}' assembly/final.contigs.fa | wc -l
	#22 
	
	#Run this in a SLURM batch job:
	sbatch batch_megahit.txt 

	#!/bin/bash
	#SBATCH --job-name=assembly_gen.        # Job name

	echo "Running megahit" 

	cd # go to home

	rm -rf assembly

	singularity run megahit -t 8 -1 ERR486840_1.fastq.gz -2 ERR486840_2.fastq.gz -o ./assembly 
	awk '/^>/ {print}' assembly/final.contigs.fa | wc -l





