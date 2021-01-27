Using Slurm Indirectly
######################

Using Slurm indirectly
**********************

Instead of controlling job preparation and submission directly via the command line it is possible to access slurm through packages in (programming) environments. A number of options are presented. 

Slurm in R
=============

Using the rslurm package an R session running on the headnode (omics-h0) that can run jobs on the cluster. The functions ``slurm_call`` and ``slurm_apply`` can be used to submit computationally expensive jobs on the compute nodes. The help provided in the package about  the functions gives an example. To test demonstrate it on Crunchomics with some slight modifications 

.. code-block:: R

	require("rslurm")
	# Create a data frame of mean/sd values for normal distributions 
	pars <- data.frame(par_m = seq(-10, 10, length.out = 1000), 
                   par_sd = seq(0.1, 10, length.out = 1000))
  	#                 
	# Create a function to parallelize
	ftest <- function(par_m, par_sd) {
 	samp <- rnorm(2e6, par_m, par_sd)
 	c(s_m = mean(samp), s_sd = sd(samp))
	}
	#
	sjob1 <- slurm_apply(ftest, pars,slurm_options=list(mem="1G"))
	get_job_status(sjob1)
	# wait until get_job_status(sjob1)$completed 
	# the job will take about a minute if resources are available. 
	res <- get_slurm_out(sjob1, "table")
	all.equal(pars, res) # Confirm correct output
	cleanup_files(sjob1)

Some notes while running it on Crunchomics: 

*   While running jobs the package creates a map in the current directory which is used to communicate data to the R processes on the nodes called *slurm_[jobname]*. The working directory changes to that directory so relative file paths in the function will not work. 
*   The cleanup_files function can be used to remove this directory. 
*   By default the jobs will have 100MB of memory. Rather than giving up, the typical behaviour of R is to become very very slow if it does not have enough memory. Use ``slurm_options=list(mem="1G")`` in the slurm_call/slurm_apply to get enough(Change 1G to the needed amount) memory for your function. 
*   R has to run on the Slurm-cluster (Jobs can not be submitted from omics-app01)

Slurm in Python
===============

A package called simple_slurm is available in the madpy3 environment. Check `Simple Slurm pypi.org <https://pypi.org/project/simple-slurm/>`_ for package documentation and examples. 

Slurm and Snakemake
===================

Existing snakemake pipelines can be run on the cluster (almost) without change, using a parameter file (cluster.json) snakemake is told how jobs have to be submitted. Below is an example setup of a normal  snakefile which maps a number of samples to a genome. This demonstration can be copied and  run using the following commands: 

.. code-block:: bash

	cp -r /zfs/omics/software/doc/Crunchomics_intro/SnakeDemo .
	cd SnakeDemo
	ls
	# 3 files and an empty directory are copied 
	#
	# Snakemake has to be available use 
	# source /zfs/omics/software/v_envs/madpy3/bin/activate	
	# if which snakemake comes back without result 
	#
	batch snakemake.cmd


Check the snakefile using 

.. code-block:: bash

	cat Snakefile

Snakefile is a typical snakemake input file to trim and map some samples, but the process should work for any Snakefile.  The snakefile can be processed unchanged by snakemake utilizing the cluster using the a parameter file: 

.. code-block:: c

	#cluster.json
	{
    	"__default__" :
    	{
        	"time" : "00:30:00",
        	"n" : 1,
        	"c" : 1,
        	"memory"    : "2G",
        	"partition" : "all",
        	"output"    : "logs/slurm-%A_{rule}.{wildcards}.out",
        	"error"     : "logs/slurm-%A_{rule}.{wildcards}.err",
    	},
    	"align_to_database" :
    	{
        	"time" : "02:00:00",
        	"memory"    : "10G",
        	"c" : 8
    	},
    	"trim_read" :
    	{
        	"time" : "01:00:00",
        	"c" : 2
    	} 
	}


In the parameter file the options values can be specified for how snakemake has to configure the slurm jobs. In it the number of cores, the amount of memory etc. is given for the rules. If the name of the rule  is not given snakemake uses the values given in ``__default__``

Using the ``--cluster`` and ``--cluster-config`` parameters snakemake is able replace the job execution commands with slurm batch jobs. 

.. code-block:: bash

	snakemake -j 8 --cluster-config cluster.json --cluster "sbatch  -p {cluster.partition} -n {cluster.n}  -c {cluster.c} -t {cluster.time} --mem {cluster.memory} -o {cluster.output} "


The command above can be wrapped into an sbatch script (the file: snakemake.cmd)  In this way the snakemake program itself runs as a slurm job. Note that sbatch commands executed from within an sbatch job will run in a new and separate allocation independent of the allocation from which sbatch is run. While the snakemake program itself runs on a single core the batches it runs like the mapping tasks can use 8 cores.   

The mapping of parameters defined in the cluster.json file to the sbatch command. Snakemake takes care that jobs are submitted using the parameters in the cluster.json file. By default the parameters in ``__default__`` are used. For rules that match a clause in the cluster.json file for example ``align_to_database`` changes from the default can be included for example the the number of cores (8 instead of 1) and the time (2 hour intead of 30 minutes). 
As mentioned in the introduction snakefiles can be run **almost** unchanged. Some rules do very little actual work and it make little sense to setup and schedule a job for it for example creating a directory. These rules can be declared as ``localrules`` at the top of the snakefile. This means snakemake will execute them directly instead of submitting them as a slurm job. Also depending on if and how thread counts are dealt with in the snakefile tweaking can improve the utilization of the allocated resources.  

The snakemake parameter ``-j`` which normally limits the number of cores used by snakemake limits the number of concurrently submitted jobs regardless of the number of cores each job allocates. Setting -j to high values such as 999  as is done in some examples on the internet is usually not a good idea as it might claim the complete cluster. 

Using snakemake in combination with the (super fast NVMe SSD) local scratch partition for temporary storage adds an extra challenge. The main mode of operation of snakemake is checking the existence of files. However, a snakemake process running on the headnode can not access the scratch on the nodes. A possibility is to configure snakemake (in cluster.json) to run on a single node and to combine that with running snakemake itself using sbatch in the selected node. 

Debugging Slurm jobs
####################

If things do not run as expected there are a number of tools and tricks to find out why. 

*   Initially ``squeue`` can be used to get the state of the job. Is it in the queue, running or ended. A job which request resources which are not available is queued until the resources become available. If a jobs ask for 6 nodes it will be queued but never run as there are only 5 compute nodes. 
*   Check the output file of the batchjob (slurm_[jobid].out or the name given using --output parameter).  Use ``cat slurm_[jobid].out`` or ``tail slurm_[jobid].out`` to check what the job sent to its standard output. 
*   To see if the job runs as expected, an additional interactive job can be started on the node the job is running on using ``srun -w [nodeX] --pty bash -i``.  The commands ``top`` and ``htop`` are useful to see which programs are running and if the program is  using the allocated resources as expected.  
*   If it is not possible to allocate a job on the node an option is to directly ssh into node. This is only possible if you have an active job on the particular node. 
*   A common problem is lack of memory. Memory is also limited in by the slurm allocation. The default memory allocation is 100MB. While this is fine for some, it is not enough for many jobs. Some jobs simply give up with an error if they don't have enough memory. Others try to make the best of it. This might result in jobs, which are expected to complete in a couple of hours to run for days. A telltale sign of this happening is that memory usage shown by top is close to or equal to the allocation asked for. While setting up jobs keep in mind the compute nodes have 512GB each. That is  8G for each of the 64 cores on average. Don't allocate what you do not need: this might be used by others. It is not unreasonable to allocate 128GB of memory for a 16-core job.     

