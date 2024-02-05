
# Unit 2:

## Pipeline structure.

### To recap 

>In unit 1 I outlined what the core code of this vignette will consist of and gave a few lines of commands that I would enter into the command line of a linux system in order to run the code from start to finish.
Since our objectives here are not to run the code a single time on a local computer, this post will be walking through the first step to uplifting some code into and effective cloud environment. 
Specifically, we will be looking at adding a pipeline format that will allow our code to be run by the pipeline software nextflow.

### Lets begin

To begin with Nextflow we can start by breaking our code into the two important bits.
The first important bit, the part that reads in the tomato accession number and returns metadata will be the first step in our pipeline.
In a Nextflow this is what's called a <mark>script</mark> and scripts are delineated by (among other ways) three quotation marks on the line before and the line after the process block.
We only have one command, on a single line so our script will look like a line with three quotation marks, our code that we want to run, and then another line with three quotations.

>In Nextflow, scripts are organized within processes, which are sort of the fundamental unit of a nextflow script; like a paragraph in an essay.

To run our script we need to package it within a <mark>process</mark>.
Strictly speaking we need: 

1. a name for our process 
2. a curly open bracket 
3. a script to run
4. a curly closed bracket.

So our simplest version of our first command looks like:

```
process pullNCBI {
	"""
    datasets summary genome accession GCF_000188115.5  > tomato.metadata
    """
}
```

This is correct in terms of Nextflow syntax but not sufficient for our overall purpose because we want what is produced here to be passed to our next step.
To make this happen we add an <mark>output block</mark> to our process which looks like this:

```
process pullNCBI {
    output:
    path "tomato.metadata"

    """
    datasets summary genome accession GCF_000188115.5  > tomato.metadata
    """
}
```

### Your next process block

That then completes our first process block named pullNCBI.
Now we have a completed process block with an explicated output that we want to a second script to operate on so the next is to repeat the above process and take our next script and situate it within a process block like so:

```
process convertToUpper {
    """
    cat tomato.metadata | tr '[a-z]' '[A-Z]' > tomato.metadata.upper
    """
}
```

As with our earlier process, this is technically correct and would work if you ran it in isolation in a directory where a file named `tomato.metadata` lives but won't achieve our objective because we don't specify that `tomato.metadata` isn't a file that already exists in the place where it looks for files and is instead the output of the above process.
We fix this by supplying an input block.
The input block follows a similar format as the output block that we used earlier.
Here I will just define the input as the variable `x` and now our finished second pipeline step becomes:

```
process convertToUpper {

    input:
    path x

    """
    cat $x | tr '[a-z]' '[A-Z]' > tomato.info
    """
}
```

The final ingredient in this little pipeline jambalya we're making here is the <mark>workflow</mark> block.
The two process blocks contain all the requisite information to run the scripts within them but do not contain the information required to know what process, if any, should feed into any other process.

>Basically, the workflow block acts as a navigation for how we want Nextflow to run our otherwise independent processes.

In our case we want process pullNCBI to hand its output over as input for the process convertToUpper so our workflow block looks like:

```
workflow {
    pullNCBI  | convertToUpper 
}
```

### What did we do here?

So as a recap so far, we've taken our little coding task outlined in the first post and built this Nextflow script to run it:

```
process pullNCBI {
    output:
    path "tomato.metadata"

    """
    datasets summary genome accession GCF_000188115.5  > tomato.metadata
    """
}

process convertToUpper {

    input:
    path x

    """
    cat $x | tr '[a-z]' '[A-Z]' > tomato.info
    """
}

workflow {
    pullNCBI  | convertToUpper 
}
```

To do the actual running of this pipeline I saved the above code in a file called main.nf and, on a computer with a working -- executable version of the ncbi datasets and nextflow software, ran the following from the console:

```console
foo@bar:~$ nextflow run main.nf 
```


I have also saved the pipeline in a github [here](https://github.com/danjgates/pipeIntro).
Nextflow has very nice integration with github in that it will natively search for github repos without requiring a bunch of fancy arguments.
The only customization I do is because I have my github project set to main, and Nextflow's default looks for the branch "master" so I reconcile this with the -r flag ("-r main") but the command to run the Nextflow script from my github is:

```console
foo@bar:~$ nextflow run danjgates/pipeIntro -r main
```


Congratulations, you have just run your first pipeline (unless of course it didn't run or this wasn't your first pipeline)
On completion on my local machine this is what the output of the successful run looks like:

```
N E X T F L O W  ~  version 23.10.0
Launching \`main.nf` [hopeful_minsky] DSL2 - revision: 9687f40736
executor >  local (2)
[dd/15d10e] process > pullNCBI       [100%] 1 of 1 ✔
[af/e36cb4] process > convertToUpper [100%] 1 of 1 ✔
```


What does this all mean? 
The first line that shows the version is self explanatory.
The second shows the file that was just run and the third line tells you that Nextflow used the local executor (as opposed to a cluster like you might find in a high performance computing cluster or through a cloud system like AWS Batch that we'll explore later).
The fourth and the fifth lines show the processes that are being run in the order they are running.
Our script runs pretty fast but you should see these update in real time as Nextflow is running.

Another important piece of information from these lines is found in the brackets.
If you didn't notice, assuming you were running the Nextflow commands I gave above, there was a new directory created called `work`.
If you were to open this work directory after this run completed there would be two directories called `dd` and `af` and within each there would be another directory (`15d10e` would be in `dd` and `e36cb4` would be found in `af`).
It is here in these directories shown in the brackets that you would find generally useful things like the input and output files, run logs, and hidden executable scripts that are generally helpful for debugging.
The `af/e36cb4` folder is also where our final results and sure enough when I navigate there I see a nice file called `tomato.info` that contains a bunch of genome metadata that is, importantly, in all caps.

### This concludes our initial foray into the world of pipelines.

Stay tuned for more additions!
