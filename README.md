# micropacker-for-containers

Kernel assisted microcontainerization PoC.

Micropacker-for-containers is a PoC that will help you micro-containerize a normal container in a faster and more effective way than using a "trial and error" [approach](https://hackernoon.com/how-to-build-a-tiny-httpd-container-ae622c37db39#c81d). If you are interested in learning first what a microcontainer is, take a look at a general introduction [here](https://blogs.oracle.com/developers/the-microcontainer-manifesto).

There is also the small chance that you are here because you already know what microcontainers are, but you don't want to use a "tiny" base image like the ones provided by Google in the [distroless](https://github.com/GoogleContainerTools/distroless) project but rather build your own.

This project follows a KISS approach, trying to do only a few things and heavily leveraging other open-source utilities to build the final microcontainer image.

## General overview

The basic steps:

1. Put your kernel in "listening" mode;

2. Run all your tests against your target container to trigger its functionalities;

3. Export the information captured by the kernel in a human-readable format with `sysdig`;

4. Compile and then copy `micropacker` to a newly created target container and execute it, using the file generated above as input.

## Why micropacker-for-containers

The are basically three ways an executable can reference functions declared in a library:

* Static linking
* Dynamic linking
* Dynamic loading (look at `dlopen` [man page](https://linux.die.net/man/3/dlopen))

With static linking all the library functions are already included in the final executable, while with dynamic linking instead you need use the output of something like `ldd` to gather the list of the linked libraries. Dynamic loading on the other hand poses an additional challenge, as you would need to disassemble your target binary, as well as all its shared libraries, and parse it to find every possible way `dlopen` can be called. Just as an example, see the [error](https://hackernoon.com/how-to-build-a-tiny-httpd-container-ae622c37db39#c81d) generated by a dynamically loaded (and missing) `libuuid` library.

The good news is that not only the glibc `dlopen` call, but also virtually every file opened by the target binary can be intercepted by listening to syscalls.

Instead of reinventing the wheel (or a system call table rootkit!), [sysdig](https://sysdig.com/opensource/) seems a good solution to listen to kernel events. You are not forced to use `sysdig` provided that you give `micropacker` a compatible file in input, you can also write the input file by hand should you wish so. Micropacker-for-containers just makes your life easier providing a [chisel](https://github.com/draios/sysdig/wiki/Chisels-Overview) in the repository; chisels are small Lua scripts that allow you to extend `sysdig` functionality. The `microdump.lua` script is used to generate a human readable file that is exactly what `micropacker` expects in input.

## Getting Started

The process of generating a microcontainer image is comprised of the following 6 stages:
* Host machine preparation;
* Micropacker compilation;
* Sysdig startup;
* Container tests execution;
* RootFS creation;
* Microcontainer image creation.

These stages are further described on the following sections - to make it easier to understand, kernel-assisted micropacking stages are explained by seeing them in action against a normal container. As a rule of thumb, the target containers should have at least a shell or a binary you can use to have the container wait for commands or sleep indefinitely.

For this README, we will use the development version of project [Admiral](https://hub.docker.com/r/vmware/admiral/), an open-source Java application based on Photon that you can download from Docker Hub by pulling `vmware/admiral:dev`.

### Prerequisites

#### On the host

On an Ubuntu/Debian system, satisfying prerequisites on the host shouldn't be different from:

```console
user@host:~$ sudo apt-get install sysdig
user@host:~$ sudo cp microdump.lua /usr/share/sysdig/chisels/
```

On a Photon OS system instead:

```console
root@host:~# tdnf install sysdig
root@host:~# cp microdump.lua /usr/share/sysdig/chisels/
```

#### Static build of micropacker.go

It is possible to have an executable that will run in every possible Linux container by statically linking Golang code against musl instead of glibc.
See this guide on [statically compiling Golang with musl](https://dominik.honnef.co/posts/2015/06/statically_compiled_go_programs__always__even_with_cgo__using_musl/) for more information.

We have a docker-based build that runs with a Makefile (requires sudo privileges) - the two latter steps are optional and allow you to check that the final binary isn't using dynamically linked libraries:

```console
user@host:~$ make
user@host:~$ ls -lh micropacker/micropacker
user@host:~$ ldd micropacker/micropacker
```

### Put sysdig in listening mode on the host

Ask `sysdig` to perform a raw capture of everything that happens on your target container:

```console
user@host:~$ sudo sysdig -qw kdump.cap container.image=vmware/admiral:dev &
```

Pull your target image and start it normally:

```console
user@host:~$ sudo docker pull vmware/admiral:dev
user@host:~$ sudo docker run -d -p 8282:8282 vmware/admiral:dev
```

### Run __ALL__ the tests you have

This is the key part: __the approach is as good as your QA__.

The kernel will see as many `dlopen` calls (actually it sees `open` syscalls) as deep your functional testing is. Trigger every functionality/feature of the product, possibly with automated tests as part your CI/CD pipeline.

### Create a rootfs.tar archive from the container

Stop the `sysdig` capture and use `microdump.lua` to get your "needed files" list.

```console
user@host:~$ sudo kill <sysdig PID>
user@host:~$ sudo docker stop <cid>
user@host:~$ sudo sysdig -r kdump.cap -c microdump | sort | uniq > files.txt
```

Copy the `micropacker` and `files.txt`, then jump inside a __new__ Admiral container, overriding the `ENTRYPOINT` tag and running as root:

```console
user@host:~$ sudo docker run -dit --entrypoint=/bin/sh -u=0 vmware/admiral:dev
user@host:~$ sudo docker cp files.txt [cid]:/tmp/
user@host:~$ sudo docker cp micropacker/micropacker [cid]:/tmp/
user@host:~$ sudo docker exec -it [cid] /bin/sh
```

Now run `micropacker` to get a tarball in debug mode:

```console
root@container:~# cd /tmp
root@container:~# ./micropacker -i=files.txt -o=rootfs.tar -d=true
```

### Build your micro image

Now that you have the `rootfs.tar` archive, a microcontainer is just a matter of creating an appropriate Dockerfile.  
Copy back the `rootfs.tar` file to your host and create a Dockerfile that contains the needed `ENV`, `VOLUME`, `WORKDIR`, `EXPOSE` tags as the original one... basically everything except the `FROM`, `RUN`, `COPY` and `ADD` tags. Do not forget Dockerfile tags that are set by parent Dockerfiles!

Example for Admiral:

```dockerfile
FROM "scratch"
ENV JAVA_HOME=/usr/lib/jvm/default-jvm \
  ADMIRAL_PORT=8282 \
  ADMIRAL_STORAGE_PATH=/var/admiral/ \
  USER_RESOURCES=/etc/xenon/user-resources/system-images/ \
  ADMIRAL_ROOT=/admiral \
  MOCK_MODE=false
ENV DIST_CONFIG_FILE_PATH=$ADMIRAL_ROOT/config/dist_configuration.properties
ENV CONFIG_FILE_PATH=$ADMIRAL_ROOT/config/configuration.properties
ENV LOG_CONFIG_FILE_PATH=$ADMIRAL_ROOT/config/logging.properties
ENV HOME=$ADMIRAL_ROOT
ENV TERM=xterm

WORKDIR $ADMIRAL_ROOT
EXPOSE $ADMIRAL_PORT
VOLUME $ADMIRAL_STORAGE_PATH

ADD rootfs.tar /
ENTRYPOINT ["/entrypoint.sh"]
```

In case there are environment variables set on the parent Dockerfiles, you can set them on your container with the `env` command once inside a container.

Now build the Dockerfile, start the microcontainer and then try to `docker exec` to the microcontainerized Admiral!

## FAQ

### I have a container I want to minimize, but I don't have a listening service

That shouldn't be an issue, trigger all the tests with a `docker exec` command.

### My image is _too_ minimal, I want to preserve some debuggability

As above, identify what commands are important for you (`ls`, `cat`, ...) and trigger them with a `docker exec` before stopping the kernel monitoring.

### Can this be automated?

Yes! The main problem of the legacy version was that an automation script should have known in advance the distribution of the target container to start the installation of the Python runtime libraries.

### What's next for this project?

We are investigating on how to remove the sysdig dependency and use other tracing mechanisms.

## License

The project is licensed under the BSD 2-clause license, see the [LICENSE.txt](LICENSE.txt) file for the full text.
Release binaries are not currently being considered, as they require additional work from a legal/licensing standpoint.
