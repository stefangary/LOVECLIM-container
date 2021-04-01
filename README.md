# LOVECLIM-container
Docker file and associated documentation for building and running a LOVECLIM container.

# Building the Docker container

Built the Docker container locally using the standard command line with the dockerfile:
```bash
sudo docker build -t stefanfgary/loveclim .
```

It ran without any difficulties - this is using the Dockerfile provided by Pierre-Yves.

# Transferring the image

Saved the docker image:
```bash
sudo docker save a907915a0d63 | gzip -1c > loveclim.tar.gz
```

and then copied the 10GB compressed image to the Bowdion HPC grid.

# Upacking and converting to Singularity

First decompress the image, comes out to 26GB:
```bash
gunzip -c loveclim.tar.gz > loveclim.tar
```

By default, Singularity will write images to `$HOME/.singularity`.  This
will not work for the HPC grid since the image is more than the 10GB
allowed storage in `$HOME`.  Instead, define a new location to
store Singularity images with the environment variable:
```bash
export SINGULARITY_CACHEDIR="/mnt/research/sgary/.singularity"
```

and then create a singularity image from this tarball:
```bash
# No need to load singularity as a module - it is present in the base shell
# This command line is based on being in the same directory as the
# image archive file.  Absolute path example: docker-archive:///mnt/path_name
singularity build --sandbox loveclim docker-archive://loveclim.tar
```

# Running the Singularity container

One could probably automate the binding process with a .yml file
similiar to [singularity-compose-simple](https://github.com/singularityhub/singularity-compose-simple).
As far as I can see, based on the docs in singularity compose and
this [blog post (Turner-Trauring, 2021)](https://pythonspeed.com/articles/containers-filesystem-data-processing), 
it looks like Singularity does not have something akin to Docker 
volumes and only bindings are used.

For my purposes, I don't really need the container/service to stay up
in a persistent way - rather, I want it to be created, take input,
run model, write output, and that's it.  (+ repeat with other inputs).
In this case, setting up a service is not needed.

Right now, I'm working interactively for testing:
```bash
singularity shell --bind /mnt/research/sgary/LOVECLIM_latest/RUN:/LOVECLIM/RUN2,/mnt/research/sgary/data:/data loveclim
```

Then, within the container's shell,
```bash
cd /LOVECLIM/RUN2/V1.3/expdir/
./newexp example.param
cd example
./launch_r1
```

Code starts to compile but there is an error in linking -ludunits2.
**Solution:** the udunits library is installed at /opt/UDUNITS 
(in the Dockerfile) and there is no libudunits2.a library,
only libudunits.a, so the error makes sense.

In `LOVECLIM_latest/RUN/V1.3/expdir/ref/make.macros`, I removed 
the 2 at the end of -ludunits2 since there are no libs by that name.
I also changed `-L${UDULOVEDIR}/lib64` to just `-L${UDULOVEDIR}/lib` 
since lib64 was not created when building the `opeapi` container. 
Note that $UDULOVEDIR is an env variable set to /opt/UDUNITS at 
the very end of the Dockerfile.

# Verify output

Downloaded the LOVECLIM V1.2 [sample output](https://www.elic.ucl.ac.be/modx/assets/docs/LOVECLIM/OLD/LOVE_output.tar.gz).
The output files in my run using `example.params` have similar structure 
and variables as in the sample output.  Also, I verified that the large
scale ocean properties are reasonable (MOC, T, S, SSH, SST, etc.)

# Git submodules

At first, I'm just testing with some local repos to see what kind of control
I have on pinning versions.  I created 2 sample repositories: `stefangary/d1`
and `stefangary/d2`.