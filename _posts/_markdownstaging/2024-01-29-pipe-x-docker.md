
## Intro to Docker

In the last blog we built a docker image from a Dockerfile and pushed it to our repo that we created on dockerhub.
Lets see in this edition if we can't get our containerized software talking to our pipeline.

I'm going to walk through a somewhat laborious demonstration of how to access our docker image, and how to poke around to see that it's working.
Most of this is not a necessary step in launching pipelines but I do think it's a good academic experiment to walk through every now and then as it helps me greatly when I need to debug my pipelines.
 
1. First I'm going to prune all my docker images and I'm going to stop and remove all my active containers. You can view active containers with `docker ps -a`

2. Then you can stop active containers with `docker stop $(docker ps -aq)`

3. Then remove stopped containers with `docker rm $(docker ps -aq)`

4. Now we'll move on to local images. You can view active images with: `docker images`

5. And finally prune images with: `docker system prune -a`

This will ensure I'm working from a fresh docker image that I pull from docker hub, not a version that I may have kicking around from development.
The stop, remove, and prune steps are not necessary but I think it's good to be exposed to them if you are a docker beginner because they are (at least for me) quite important when I'm developing pipelines.

For the sake of this task I like to tunnel into the docker image and poke around, it gives me a better idea of what the environment looks like than calling individual commands on the image from my terminal.

To do this we want to pull down the image from our repository:


```console
foo@bar:~$ docker pull danjgates/ncbidatabases
```

(Remember our above command to check for images? We can use that again here if we want to check on our pulled image)

Then we want to run the image:

```console
foo@bar:~$ docker run --name datasetsimage -d -it danjgates/ncbidatabases
```

(Remember our above command to check on running containers? We can use that again here if we want to see that it's running)

As a quick reminder, my current conda (base) environment that I'm working out of here does not have a working copy of ncbi datasets on it (my working version is in my conda datasets environment)
So when I go to my terminal and try to execute the datasets command I get this:

```
(base) danielgates@Daniels-Mac-mini nextflow % datasets --version
zsh: command not found: datasets
```


Now I want to tunnel into the image so I can poke around and verify that the software we want is still there. 
For that I use:

```console
foo@bar:~$ docker exec -it datasetsimage bash
```


There's not a lot you need to do here, the obvious one is to make sure that you can call the datasets command from the $HOME directory.

```
root@d64613987c2b:~# cd $HOME
root@d64613987c2b:~# datasets --version
datasets version: 16.2.0
```

This tells me that datasets can be called from `$HOME`.
It's also good just to check on your `$PATH` and make sure it is what we specified in the Dockerfile that the image was plausibly built upon.

```
root@d64613987c2b:~# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
root@d64613987c2b:~# which datasets
/usr/local/bin/datasets
```

This tells me that datasets is in ``/usr/local/bin/datasets` and that is in my `$PATH`.
Everything checks out.

As with above, these commands, especially the docker exec command and the poking around is not necessary, but again I think it's a good introduction to how to move in and out of images if you don't have much background in docker.

## Make the pipeline talk to docker


Great! Now how do we make our pipeline use our docker virtual machine instead of the local machine it's on?

The simplest way, since we have already pushed an image that we have verified is working onto docker hub is to use the `--with-docker` flag to point to our docker hub repo.


As a little illustration as to how this works I'll start by executing my nextflow pipeline out of my current location.
When I try to execute my script I get:

```
(base) danielgates@Daniels-Mac-mini nextflow % nextflow run main.nf --input 'GCF_000226075.1'             
N E X T F L O W  ~  version 23.10.0
Launching `main.nf` [compassionate_thompson] DSL2 - revision: ebf5336160
executor >  local (1)
[dd/cd991d] process > pullNCBI       [100%] 1 of 1, failed: 1 ✘
[-        ] process > convertToUpper -
ERROR ~ Error executing process > 'pullNCBI'

Caused by:
  Process `pullNCBI` terminated with an error exit status (127)

Command executed:

  datasets summary genome accession 'GCF_000226075.1' > GCF_000226075.1.metadata

Command exit status:
  127

Command output:
  (empty)

Command error:
  .command.sh: line 2: datasets: command not found

Work dir:
  /Users/danielgates/Desktop/nextflow/work/dd/cd991d88abdb367387c0784bc0be82

Tip: you can replicate the issue by changing to the process work dir and entering the command `bash .command.run`

 -- Check '.nextflow.log' file for details
```

This is not surprising because, as I established above, I do not have a working version of datasets where my terminal is currently at. 

But when I add the with docker command my nextflow script:

```
(base) danielgates@Daniels-Mac-mini nextflow % nextflow run main.nf --input 'GCF_000226075.1' -with-docker 'danjgates/ncbidatabases'
N E X T F L O W  ~  version 23.10.0
Launching `main.nf` [irreverent_hypatia] DSL2 - revision: ebf5336160
executor >  local (2)
[d8/fcb21b] process > pullNCBI       [100%] 1 of 1 ✔
[08/21dc38] process > convertToUpper [100%] 1 of 1 ✔
```

Huzzah!
The script runs to completion.

This concludes the meat and potatoes of the docker portion.

## Final thoughts from Dan (for now)

Now as a final refresher: why are we using docker again?

The final goal of this project is to run our code on a cloud machine.

The most effective way to take advantage of cloud resources and scalability imo is to assume you are starting with a completely fresh machine that has no software installed on it outside of what comes out of the box on a brand new linux operating system.
So what we did with docker is to make a virtual machine that we know has the software we need.
Thus, instead of trying to tell a fresh linux operating system what code it needs to install, and navigating all the crap that comes with dependency issues, version, etc. on the fly, every time we want to run our pipeline in the cloud we have instead made a virtual machine that we know works and all we have to do is tell our fresh cloud machine to hand over its hardware resources to the virtual machine.
This greatly simplifies the problem because all we do now is focus on conversing with the virtual machine (telling it what input files are where, what code to run, where to put things etc.)

As with the pipeline segments, I could also add another digression about costs and benefits of containerization.
Largely, however, the arguments are very similar to the pipeline cost benefit analyses, containers are another **thing** and as such require time and effort to set up, they are not guaranteed to work for your specific task, and they can be a brand new set of headaches (in my case I burned a full two days troubleshooting unhelpful bash errors before I figured out that developing my containers on an M1 chip meant they were incompatible with the architecture of the cloud hardware I was trying to run them on).
And, conversely, the benefits are quite similar.
When done right they make your whole workflow more stable, reliable, reproducible, portable, and they are crazy synergistic with a pipelines and the cloud and especially nice for those of us who don't have cluster admins willing to do custom software install for our projects (don't forget to tell your cluster admin you appreciate them today)
With that, I think we can wrap up Docker.
We now have two (code we want to run, and software to run said code) out of the three main components required to realize our objective of running pipelines in the cloud.
The last piece of the puzzle is to go out and find some cloud computing hardware to set all this on top of.



