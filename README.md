# docker-to-vms
----------------------

# Why you should not follow "Best practices for writing Dockerfiles".

[Docker's Best practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

They recommend that you:

- Use multi-stage builds

        `# syntax=docker/dockerfile:1
        FROM golang:1.16-alpine AS build

        # Install tools required for project
        # Run `docker build --no-cache .` to update dependencies
        RUN apk add --no-cache git
        RUN go get github.com/golang/dep/cmd/dep

        # List project dependencies with Gopkg.toml and Gopkg.lock
        # These layers are only re-built when Gopkg files are updated
        COPY Gopkg.lock Gopkg.toml /go/src/project/
        WORKDIR /go/src/project/
        # Install library dependencies
        RUN dep ensure -vendor-only

        # Copy the entire project and build it
        # This layer is rebuilt when a file changes in the project directory
        COPY . /go/src/project/
        RUN go build -o /bin/project

        # This results in a single layer image
        FROM scratch
        COPY --from=build /bin/project /bin/project
        ENTRYPOINT ["/bin/project"]
        CMD ["--help"] `

If you are already using more than one `FROM` in your `Dockerfile` you have a problem.
Yes this maybe faster in building your image but there are issues with this approach.

- Dependencies you add with each `FROM`

- Security vulnerabilities you add with each `FROM`

- Maintainability of code. With each `FROM` you add, the code become difficult to maintain and understand.

What the best practice should be is to build your own image from if you require more than one `FROM` in your `Dockerfile`.

In the following example, lets say I want to build a docker Ubuntu focal image.

Your `Dockerfile` will start with

`FROM ubuntu:20.04`

or any other Debian base image. If you seriously need to be pulling from another `FROM` then instead of following  the recommended approach above build your own.

Assuming you are on ubuntu development platform:

# 1. Prerequisites

	sudo apt-get install \
	    binutils \
	    debootstrap \
	    squashfs-tools \
	    xorriso \
	    grub-pc-bin \
	    grub-efi-amd64-bin \
	    mtools

# 2. Bootstrap base  (any variant of debian you need)  

	 sudo debootstrap \
	   --arch=amd64 \
	   --variant=minbase \
	   focal \
	   $HOME/focal/chroot \
	   http://us.archive.ubuntu.com/ubuntu/

# 3. Mount points (optional)

	sudo mount --bind /dev $HOME/focal/chroot/dev \
	&& sudo mount --bind /run $HOME/focal/chroot/run 

# 4. Access focal environment

	sudo chroot $HOME/focal/chroot

Now that you are inside the environment, you can install whatever you want like using `RUN` in your `Dockerfile`.

For example install docker if you want docker in docker:

	cat <<EOF > /etc/apt/sources.list
	deb http://us.archive.ubuntu.com/ubuntu/ focal main restricted universe multiverse
	deb-src http://us.archive.ubuntu.com/ubuntu/ focal main restricted universe multiverse

	deb http://us.archive.ubuntu.com/ubuntu/ focal-security main restricted universe multiverse
	deb-src http://us.archive.ubuntu.com/ubuntu/ focal-security main restricted universe multiverse

	deb http://us.archive.ubuntu.com/ubuntu/ focal-updates main restricted universe multiverse
	deb-src http://us.archive.ubuntu.com/ubuntu/ focal-updates main restricted universe multiverse
	EOF

	apt-get update	&& \
	apt-get install gpg curl -y && \
	mkdir -p /etc/apt/keyrings && chmod -R 0755 /etc/apt/keyrings \
	    && curl -fsSL "https://download.docker.com/linux/ubuntu/gpg" \
	    | gpg --dearmor --yes -o /etc/apt/keyrings/docker.gpg \
	    && chmod a+r /etc/apt/keyrings/docker.gpg \
	    && echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu focal stable" > /etc/apt/sources.list.d/docker.list
	    	    
	DEBIAN_FRONTEND=noninteractive apt-get update \
	    && apt-get install -y -qq --no-install-recommends \
	    docker-ce=5:20.10.17~3-0~ubuntu-focal \
	    docker-ce-cli=5:20.10.17~3-0~ubuntu-focal \
	    containerd.io=1.6.8-1 docker-compose-plugin=2.6.0~ubuntu-focal \
	    docker-scan-plugin=0.17.0~ubuntu-focal \
	    && docker --version

Once you are done provisioning your environment `exit` and unmount if you needed it.

# 5. Docker image

	sudo tar -C $HOME/focal/chroot -c . | docker import - focal

# 6. Test

	docker run focal cat /etc/lsb-release && docker --version

- output:

	DISTRIB_ID=Ubuntu

	DISTRIB_RELEASE=20.04

	DISTRIB_CODENAME=focal

	DISTRIB_DESCRIPTION="Ubuntu 20.04 LTS"

	Docker version 20.10.17, build 100c701
	

# 7. Make changes

	docker commit --change='CMD ["/bin/bash"]' [containerID] [new image]

So that you can do this for example
	
	docker run -it [new image]
	
Or you can now safely add `FROM` your own image in a `Dockerfile`

		FROM focal
		EXPOSE 2222
		CMD ["bin/bash"]	

# 8. Image size & Memory optimization

It is true that the recommended `Dockerfile multi-stage builds` will produce "image size reductions" for your `FROM` final or `run` stage for example:

		FROM gcc AS build
		COPY hello.c .
		RUN gcc -o hello hello.c
		FROM ubuntu
		COPY --from=build hello .
		CMD ["./hello"]

Your final image does not have the gcc image. And even further reduction if you pulled with

		FROM busybox:glibc
		COPY --from=build hello .
		CMD ["./hello"]

In this particular example your final image is about 5MB instead of 1GB with the `gcc` image.

Where this argument falls apart is when you point out that you could equally `build` with the gcc image and with `build` artifacts `deploy` with `busybox:glibc` image using separate stages which are not within a single `Dockerfile` with a cleanup of the `build` image. You will still arrive at the same output of 5MB. Is it really `image size optimization` as it is portrayed in `Dockerfile's Best practices`?

The `True` sense of `optimization` will mean reducing the `gcc` image size itself from 1GB to 5MB.

The trick here is to know what you want inside the docker image.
In this specific example it comes down to build tools for the `gcc` image and the C shared runtime library for the `busybox:glibc` image. One is needed to build the `hello` program and the later to run it. Since you don't need build tools to run `hello` a smaller image with just what is needed to run `hello` is pulled.

You should apply the same logic with your custom images using the steps above for the base.

# DevOps

The issues of `Image size & Memory optimization` is a glimpse of why it is not enough to only know your DevOps automation tools or docker tricks or complex pipelines.

A successful DevOps Engineer has to know very well or take the time to understand what is to be automated. The simple `hello` program above is a small illustration.

When dealing with complex applications, it is a very good idea to understand how they are build, configured/run locally so that you are successful at automating them and know what you only need inside your docker images.

One skill a DevOps Engineer should definately have or learn is that of Build tools or systems for common programming languages.
Choose a language and learn the build tools very well as they are the foundations of what libraries are needed to run the binaries inside your docker containers. 

# Other useful docker commands

You can start by extracting any docker container filesystem

		ContainerID=$(docker run -d busybox /bin/true)
		docker export -o $HOME/busybox.tar ${ContainerID}

Change it locally and back to docker image with `import`

		tar -xf busybox.tar -C $HOME/busybox
		...
		tar -C $HOME/busybox -c . | docker import - busybox

# Conclusion

With the steps above you can navigate from `Docker images to Debian bootable images and virtual machines`.

The recommended multi-stage docker builds are meant to solve docker build issues.

They are not meant to solve your problem.	

If you consider your `Dockerfile` as Configuration as Code, the steps above can easily be scripted and stored in a Version Control system just like with your `Dockerfile`.

You are also in control of what you add in your image and avoid the `FROM` vunerabilities.
