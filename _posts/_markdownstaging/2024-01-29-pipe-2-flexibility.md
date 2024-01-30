---
title: '3.)Pipeline 0-to-Cloud (Pipeline flexibility)'
date: 2024-01-29
permalink: /posts/2024/01/Pipeline-flex/
tags:
  - Pipeline
  - Nextflow
  - Cloud
  - Docker
---

### In unit 3 of 0 to Cloud I'll be looking at some simple steps that can increase the flexibility of your pipelines 

# Unit 3:
## Pipeline flexibility

As you can probably see from the unit title above, the focus of this module will be on pipeline flexibility.
This is a general topic but I am going to treat it very specifically here and, admittedly, a major motivator for kicking this into its own module was because I didn't want to introduce general Nextflow and the scary code jargon that come with the flexibility achieved in parameter and variable definition all in one module.
So since you've now got your feet wet in Nextflow it's time to talk about the low hanging fruit in making your code flexible enough to handle a variety of pipeline tasks and that is parameter and variable definitions.

For this we're not actually going to start fully from scratch as I did quietly sneak in one variable in the previous module.
In the process convertToUpper the script that runs is `cat $x | tr '[a-z]' '[A-Z]' > tomato.info`.
That `$x` in that script, that's a variable and it is defined by the input statement `path x`.
So what happens is that our workflow defines that the convertToUpper process receives an input from the pullNCBI process, the input statement says we will give that input a variable name of `x` so that when the script calls `$x` the pipeline knows that that is the x that we have defined in the input statement.
We are going to expand on this now to make it so that the accession number in the pullNCBI process (GCF_000188115.5) does not need to be hardcoded so we gain the flexibility of changing it without having to make keystrokes/edits in the important script blocks of our processes. 

The way that I do this is to add this parameter to the first line of the script:

```
params.input = 'GCF_000188115.5'
```

I have now defined a parameter called input and given it a value of the accession number we want the ncbi datasets to query metadata from.
At this point in time I could modify the script in the first process to be the same except for changing the hardcoded GCF_000188115.5 to instead call my parameter with '${params.input}' and things should run similar to before.
I'm going to go a bit further and have the variable parameter input follow my objects as their name.
This way when I complete I will have a final file called GCF_000188115.5.info.
Why am I doing this?
The reason is that I am preparing to run this script on more than just the accession number for tomato and want to avoid having N parallel processes all returning a file unhelpfully called `tomato.info`.
It may seem a bit tedious to track the parameter through all the inputs and outputs in your pipeline, and by all means you can get by quite well without doing it, but I prefer to keep things orderly so if I have to dig around in intermediate files of a highly parallelized script I'll be working with something more informative than `foo` or `junk1` and `junk2`.

Ok so what does explicating the naming entail?
We can start by renaming the output of the first script to be `${params.input}.metadata` instead of tomat.metadata
So now our completed script for the pullNCBI process looks like:
```
datasets summary genome accession '${params.input}' > ${params.input}.metadata
```

This means that our output statement for the pullNCBI process is no longer valid so we then change that to:

```
    output:
    path "${params.input}.metadata"
```

That should wrap up the first process.
In the second process the input statement is still correct, as it already defined a variable that was read by the script.
The file name of what is output from the script does not need to be called tomat.info anymore and can be changed to `${params.input}.info` (I hope you are picking up the pattern here that within the curly brackets will be the accession and that I'm appending a postfix will read as .info in this case, this is a handy way of making sure you don't end up with a bunch of intermediate files and results files called GCF_000188115.5).
And since we've changed what we are calling the thing that gets output by the script we need to make sure the output matches that and to do that we change that to this:

```
    output:
    file("${params.input}.info")
```

Since we're already working on general pipeline hygiene and tidyness I'm also going to go ahead a specify that I don't want the output file to end up in some arbitrarily named folder for me to fish out later but instead I would like it to go just by itself into the work directory where I can find it easier.
To do that I'm going to define a publishDir (publish directory) in the convertToUpper process and that is going to look like this:
publishDir './work/', mode: 'move'
And that should tell the process that I want the output in the `./work` folder and that I want it moved there, not symlinked.

Altogether the code then looks like:

```
params.input = 'GCF_000188115.5'


process pullNCBI {
    output:
    path "${params.input}.metadata"

    """
    datasets summary genome accession '${params.input}' > ${params.input}.metadata
    """
}

process convertToUpper {
	publishDir './work/', mode: 'move'

    input:
    path x

    output:
    file("${params.input}.info")

    """
    cat $x | tr '[a-z]' '[A-Z]' > "${params.input}.info"
    """
}

workflow {
    pullNCBI | convertToUpper 
}
```

Notably nothing in the workflow changes, because the overall structure is still the same, we're just a bit more flexible with how we handle inputs.

Hopefully that can run for you like it did for me.

The final point related to flexibility that I must touch on is that parameters are really flexibility multipliers. 
The astute reader will notice that the current manifestation of the code requires you to go in and change that params.input on the first line by hand if you want a different accession.
That parameter doesn't need to be defined there, however, and you could instead define all your parameters in a nextflow.config file instead.
That would allow you to truly keep all the things you want the flexibility to alter out of your actual pipeline code, again reducing the edits and keystrokes that need to happen in important code.
And if we want to talk about even more flexibility, you can even call parameters from the command line when you run your nextflow script.
When I remove that first line defining the parameter params.input from the script completely and then run nextflow as:

```console
foo@bar:~$ nextflow run main.nf --input 'GCF_000226075.1'
```


it completes successfully because it knows that the --input flag means it should make a parameter called params.input and the value assigned to it should be the trailing text (GCF_000226075.1).
Pretty neat huh?!
Although I still favor parameters going in a nextflow.config file as once you hit more than 5 or so parameters that becomes a lot of work to enter at the command line every time you want your scripts to run and it's also nice having a document with all the important parameter values saved for your run so you don't have to go dig up old executed commands if you have questions later on.

# Some Parting Thoughts

Before we begin our next section I would like to highlight a few important high level points.
Cynically you could say that all we did could be completed by a bash script and you would be mostly correct.
In fact, you could probably argue that, at this point, putting the simple code into a pipeline is a net negative as there are no new benefits in terms of the results and now our code is abstracted by a whole new coding language which often makes debugging more difficult because you have a new source of errors and often simple bash errors are hidden because what Nextflow reports may not be as informative as receiving the error in bash.
Also running the pipeline like this requires at least one new piece of software (Nextflow, possibly Java) in order to execute. 
I think, in a microcosm, these are correct observations and an important part of wisdom in the world of bioinformatic pipelines is knowing the cost benefit math for applying these tools.

I do think there are some major high level advantages to getting a pipeline to this point in addition to the benefits of flexibility that I have been belaboring in the above post.
Where I see the biggest improvement gained through this specific pipeline intervention is synergy between Nextflow and github and also in checkpointing and error handling.
As I mentioned earlier Nextflow has native github integration which is a strong incentive to getting your code up on github and developing it from there.
The benefits there as compared to say a .sh bash script are numerous but include better stability, version control, and access to forking and branching so you don't end up with multiple exact copies or slightly modified versions of .sh executables as your projects expand or move through different phases (e.g. from development to production).   
Native execution of github repos from nextflow makes your code portable as well.
You don't need to send (and again propogate copies) of your .sh executables when your collaborators want to run your code or when you want to run your code from your home computer etc. 
The other benefit is that Nextflow has native checkpointing.
In our example, if I had made a typo in the script part of the convertToUpper process the first process would have completed, the run would have ended on error, but I would be able to resume the run after correcting the typo without having to rerun the first process again.
Again, this may seem trivial in a two process pipeline that completes nearly instantaneously but, in my experience, is another valuable source of flexibility as your pipelines become more complex.
