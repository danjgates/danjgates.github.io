---
title: '4.)Pipeline 0-to-Cloud (Intro to Docker)'
date: 2024-01-29
permalink: /posts/2024/01/Docker/
tags:
  - Pipeline
  - Nextflow
  - Cloud
  - Docker
---

### In unit 4 of 0 to Cloud I'll be taking us through an introduction to containerized software, specifically in Docker

# Unit 4:

## Docker


Now that we have the code we want to run in a stable, flexible pipeline set up we've satisfied the one of the three components required for our completed cloud pipeline.
Code alone, as we all know, is only part of the battle and that code is no good if the software that is required to run the code isn't installed.
That's where containers come in.
Here I'll be walking through how to get our important piece of software into a framework that lets us ensure that it will always be there when we try to run our pipeline.

The way we ensure that not just our code but also the required software is portable is through using containerization in Docker.
Docker, solely in and of itself, is a huge sprawling behemoth of a topic; one worthy of far more words than I'll be able to dedicate to it here.
As such, if you haven't encountered containerization or Docker before it may be wise to do some background reading into how it works.
It's really quite handy!

For our purposes I will be using a Dockerfile as a sort of script for how to set up a virtual machine the way that we want it set up.
This Dockerfile method is not required and the fundamental unit of Docker is an image like we will make with our Docker build command, but I personally like having the Dockerfile around as sort of receipt for what all went into making the image.
For our purposes I will be instructing docker to build an image off one of the more recent, stable ubuntu linux versions so the first line of my Dockerfile is:

```
FROM ubuntu:22.04
```

I am going to be installing the NCBI-datasets program using the software manager conda so I then make explicit that I want my machine set up so it will execute software installed where conda likes to install things and that looks like:

```
ENV PATH="/root/miniconda3/bin:${PATH}"
ARG PATH="/root/miniconda3/bin:${PATH}"
```

Now I'm going to do run the standard code that you run when you're working with a fresh Linux system to update it and upgrade it and that looks like this:

```
RUN apt-get update && apt-get -y upgrade
```

I need the program wget to fetch the conda installation files and I can install that with the native Linux installer using this:

```
RUN apt-get install -y wget && rm -rf /var/lib/apt/lists/*
```

Then I'm all set to give the instructions to install conda which look like this:

```
RUN wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh 
RUN mkdir /root/.conda 
RUN bash Miniconda3-latest-Linux-x86_64.sh -b 
RUN rm -f Miniconda3-latest-Linux-x86_64.sh 
RUN conda --version
```

I do some conda config stuff (this is not required but makes things a bit more stable as you start installing lots of bioinformatics stuff):

```
RUN conda config --add channels defaults
RUN conda config --add channels bioconda
RUN conda config --add channels conda-forge
```

And I install the aws cli, (this is not required to complete our immediate task but in the next module we'll begin setting up AWS and our virtual machine that we set up here needs to be able to contact AWS so we might as well add it in now)

```
RUN conda install conda-forge::awscli
```

Finally, we are going to install the ncbi-datasets-cli software (which is described here and the conda recipe for it is here) as so:

```
RUN conda install conda-forge::ncbi-datasets-cli
```

With the conda environment set up like this you can install just about any bioinformatics software you might need in a very similar manner to how we installed the ncbi-databases software.
This completes our Dockerfile.

Now we want to build our Dockerfile into an image of a machine that can run our software that we will then save to the repository we made in our docker hub.

For me that was `danjgates/pipeIntroDocker` which is what I want to tag my docker build with so my docker build command looks like:

```console
foo@bar:~$ docker buildx build --platform linux/amd64 -t danjgates/pipeIntroDocker .
```


Here I am using the docker command, the `buildx build` mode where I specify `--platform linux/amd64` because I'm actually building this on a different type of hardware (Apple M1 chip) than I plan on running (amd64) and I'm giving the build the tag that will eventually be the repo where it will live on docker hub `danjgates/pipeIntroDocker` and finally the period at the end of the command tells it to look for a file called Dockerfile in the current directory.
After I execute that command I can push the image to the docker hub repo where it can be accessed by our Nextflow pipeline.

```console
foo@bar:~$ docker push danjgates/ncbidatabases
```






