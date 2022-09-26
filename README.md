# docker-to-vms
----------------------

# Why you should not follow "Best practices for writing Dockerfiles".

[Docker's Best practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

[Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)

There are two claims from the `Best practices` I will put to the test and see if it stands up to scrutiny.


- "Multi-stage builds are useful to anyone who has struggled to optimize Dockerfiles while keeping them easy to read and maintain."

- "Multi-stage builds allow you to drastically reduce the size of your final image, without struggling to reduce the number of intermediate layers and files"

They say `drastically reduce the size of your final image`. Which means `image size optimisation`.

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

What the best practice should be is to build your own image if you require more than one `FROM` in your `Dockerfile`.

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

Address the discrepancies between docker containers and linux virtual machines.

	USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
	root           1  0.0  0.1 168900 12308 ?        Ss   Sep17   1:54 /sbin/init sp
	root           2  0.0  0.0      0     0 ?        S    Sep17   0:00 [kthreadd]
	
	USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
	root           1  0.0  0.0   4248  3092 pts/0    Ss   09:38   0:00 /bin/bash
	root        7062  0.0  0.0  20252  4824 ?        S    10:22   0:00 /lib/systemd/
	root       10152  0.0  0.0   5900  2852 pts/0    R+   13:25   0:00 ps aux

- Linux VMs runs its init daemon as a process with PID 1.
- Docker containers is shell or directly user-defined executable as a PID 1.

For example if you are building a bootable ubuntu image or a virtual machine image you need the kernel in the `boot` folder

	$HOME/focal/chroot/boot/vmlinuz*
	$HOME/focal/chroot/boot/initrd*

Since it is a docker container optimization it is not needed and other unused packages.

Important to learn this skill of knowing what you only need in a docker container.
Without docker installation and with unused packages lets compare the custom focal image with the official ubuntu:20.04 image.

	REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
	focal        latest    e9e5ca8f9c58   11 minutes ago   113MB
	ubuntu       20.04     a0ce5a295b63   2 weeks ago      72.8MB

As you can see our custom focal image contains about 40MB of installed packages compared to the official ubuntu:20.04 image.

Once you are done configuring your environment `exit` and unmount if you needed it.

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

You can also start by looking at what is inside the docker official images for example 

[official ubuntu:20.04 image](https://github.com/docker-library/buildpack-deps/blob/master/ubuntu/focal/Dockerfile)

[official gcc10 image](https://github.com/docker-library/gcc/blob/master/10/Dockerfile)

You can see that the `gcc` base image is actually a debian buster image

[official debian buster](https://github.com/docker-library/buildpack-deps/blob/master/debian/buster/Dockerfile)

And this image has quite a few packages which are not needed for the `hello` program even when building it.

This really shows why the `Best practice should be build your own image` and choose your own packages.

You should apply the same logic with your custom images using the steps above for the base.

# Docker layers

This is where I will get a lot of you screaming your love for docker and justification for `Docker's Best Practices`.

	Using default tag: latest
	latest: Pulling from library/gcc
	23858da423a6: Pull complete 
	326f452ade5c: Pull complete 
	a42821cd14fb: Pull complete 
	8471b75885ef: Pull complete 
	8ffa7aaef404: Pull complete 
	0dbd3d90c419: Pull complete 
	c8360ea64db4: Pull complete 
	65bba72ff1de: Pull complete 
	a615a380ba22: Pull complete 
	Digest: sha256:4f8717c532f9c07d6258e3d17faf0df97ffe0c18628d7769c1e25ca20c237a1e
	Status: Downloaded newer image for gcc:latest
	docker.io/library/gcc:latest

A quick recap of what we are talking about here. Each layer represents an instruction (i.e `RUN` or `COPY`) in the image’s Dockerfile and the docker engine will pull/push them accordingly.

Lets take our custom focal image above which is 112MB and push it to dockerhub

	The push refers to repository [docker.io/*****/focal]
	0a56bfe65fa4: Pushed 
	latest: digest: sha256:c4266a3768216451ab05a21b343740b62db219888bcf829db504e3b32b5b1bd9 size: 528

From dockerhub

	*****/focal:latest
	DIGEST:sha256:c4266a3768216451ab05a21b343740b62db219888bcf829db504e3b32b5b1bd9
	OS/ARCH
	linux/amd64
	COMPRESSED SIZE 
	56.48 MB
	LAST PUSHED
	4 minutes ago by *****

Can you notice any thing? Yes, our custom focal image has been compressed to 56.48MB

Yes, there is an advantage in docker layering of your `Dockerfile` but if there are many layers which you don't need then that advantage is wasted.
Taking the example of the `gcc` Debian buster base image above, there are quite a few packages which are unused by our simple `hello` example and it is common to find that in complex projects. Learn the skills to build your own images and you can then use `FROM` your own image to take advantage of docker's layers.


# DevOps

The issues of `Image size & Memory optimization` is a glimpse of why it is not enough to only know your DevOps automation tools or docker tricks or complex pipelines.

A successful DevOps Engineer has to know very well or take the time to understand what is to be automated. The simple `hello` program above is a small illustration.

When dealing with complex applications, it is a very good idea to understand how they are build, configured/run locally so that you are successful at automating them and know what you only need inside your docker images.

One skill a DevOps Engineer should definately have or learn is that of Build tools or systems for common programming languages.
Choose a language and learn the build tools very well as they teach you the foundations of what libraries are needed to run the binaries inside your docker containers. 

With the `hello` example a better `optimization` could be a `static library`

`gcc -o hello hello.c -static`

If this `hello` program was for example targeted for an embedded platform and glibc is too large, it makes sense to use a smaller static library.
This could reduce the image size to 2MB or less.

# Multi-platform images

I started with the claim that if you have more than one `From` in your `Dockerfile` you have a problem. The steps above demonstrate why but there is an exception when it will make sense.

	FROM multiarch/qemu-user-static:x86_64-aarch64 as qemu
	FROM arm64v8/ubuntu
	COPY --from=qemu /usr/bin/qemu-aarch64-static /usr/bin

A brief recap of what you see in this `Dockerfile`. What we want to achieve here is run `arm64v8` platform using qemu emulator.
This allow us to build/run other supported platforms other than amd64 which I use for the ubuntu development environment for all the steps above.
The context here is very different from `multi-stage builds for docker image size optimisation`.
You will still worry about what's in this image `arm64v8/ubuntu` and if there are packages you don't need if `image size optimisation` was your concern.
Lets look inside the images in our example `Dockerfile`

- [multiarch/qemu-user-static](https://github.com/multiarch/qemu-user-static/blob/master/containers/latest/Dockerfile)

As you can see the base image is `FROM busybox` which itself we know from above is about 1.5MB and the rest are tiny scripts to configure the hosts `binfmt_misc` for the target platform.
If we really want to be pure and stick to one `FROM` in our `Dockerfile` the `binfmt_misc` scripts can be easily implemented before running a different platform.

For our custom images you will bootstrap your base for the arm64 platform with 

	sudo debootstrap --arch=arm64 --foreign --variant=minbase focal $HOME/arm64/chroot
	
You will need additional pre-requisites:

	binfmt-support
	qemu-user-static
	
Otherwise you get this error when you try to chroot into the arm64 environment:

	chroot: failed to run command ‘/bin/bash’: Exec format error
	
`Exec format error` is a platform issue which you resolve with this:

	sudo cp /usr/bin/qemu-arm-static $HOME/arm64/chroot/usr/bin/

You can then access the arm64 environment:

	sudo chroot $HOME/arm64/chroot
	
And if you have trouble installing packages because of internet connection you can also copy host files from this folder

	/run/systemd/resolve/* to $HOME/arm64/chroot/etc/
	
- [Qemu](https://github.com/multiarch/qemu-user-static)
- [Buildx](https://docs.docker.com/build/building/multi-platform/)
- [binfmt_misc](https://github.com/multiarch/qemu-user-static/blob/master/containers/latest/register.sh)

# Other Dockerfile pitfalls

Generally avoid scripting a `Dockerfile` for the following reasons using this example:

[Bash script Dockerfile](https://github.com/multiarch/qemu-user-static/blob/master/run.sh)

	for file in ${releases_dir}*
	do
	    if [[ $file =~ qemu-(.+)-static ]]; then
		to_arch=${BASH_REMATCH[1]}
		if [ "$from_arch" != "$to_arch" ]; then
		    work_dir="${out_dir}/${from_arch}_qemu-${to_arch}"
		    mkdir -p "${work_dir}"
		    cp -p "${releases_dir}qemu-${to_arch}-static" ${work_dir}
		    cp -p "${work_dir}/qemu-${to_arch}-static" "${out_dir}/latest/"
		    cat > ${work_dir}/Dockerfile -<<EOF
	FROM scratch
	COPY qemu-${to_arch}-static /usr/bin/
	EOF
		    docker build -t ${DOCKER_REPO}:$from_arch-$to_arch-${TAG_VER} ${work_dir}
		    for target in  "${DOCKER_REPO}:$from_arch-$to_arch" \
			"${DOCKER_REPO}:$to_arch-${TAG_VER}" \
			"${DOCKER_REPO}:$to_arch" ; do
			docker tag ${DOCKER_REPO}:$from_arch-$to_arch-${TAG_VER} ${target}
		    done
		    rm -rf "${work_dir}"
		fi
	    fi
	done

	docker build -t ${DOCKER_REPO}:${TAG_VER} ${out_dir}/latest
	docker tag ${DOCKER_REPO}:${TAG_VER} ${DOCKER_REPO}:latest
	docker build -t ${DOCKER_REPO}:register ${out_dir}/register

What the above code snipet wants to achieve is to auto generate the target platform `Dockerfile` when the `$from_arch` platform does not match the `$to_arch` platform and build a tagged version ensuring the target `binfmt_misc` is registered. There are issues with this approach:

- Verification
- Maintainabilty of code

This code snipet can simply be achieved with a variable or `ENV` set to `$to_arch`

	cat > ${work_dir}/Dockerfile -<<EOF
	FROM scratch
	COPY qemu-${to_arch}-static /usr/bin/
	EOF

For verification, your `Dockerfile` has to be independent of the script verifying it. This is true for both `Static and Dynamic verification`.

Our example `Dockerfile` above is a simple one but if this was several lines of code which most in complex projects are this will be a nightmare for maintainabilty of code and difficult to understand.

Generally try to keep to these rules:

- Dockerfile
- Run lint or implement any static verification
- Scan security vulnerabilites and dependencies especially if you don't follow the one `FROM` rule. Another reason you should build your own images.
- Dynamic verification to ensure requirements are met.
- It is also important to document `exceptions and failures` because this is the basis of a learning process for troubleshooting.

It is worth mentioning that our example `Dockerfile` taken from the `qemu-user-static` github repository is now a `[DEPRECATED]` feature

	The concept of "compatible images" are deprecated because multiarch/qemu-user-static can build 
	and run standard multi-architecture container images without the multiarch compatible images now.

In other words they found a better way of implementation.

# Useful docker commands

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

[Issue#15721 docker.io](https://github.com/docker/docker.github.io/issues/15721)
