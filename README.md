# micropacker-for-containers

Kernel assisted microcontainerization PoC.

Micropacker-for-containers is a PoC that will help you microcontainerize a normal Docker container in a faster and more effective way than using a "trial and error" [approach](https://hackernoon.com/how-to-build-a-tiny-httpd-container-ae622c37db39#c81d). If you are interested in learning first what a microcontainer is, take a look at a general introduction [here](https://blogs.oracle.com/developers/the-microcontainer-manifesto).

There is also a small chance you are here because you already know what microcontainers are, but you don't want to use a "tiny" base image like the ones provided by Google in the [distroless](https://github.com/GoogleContainerTools/distroless) project and rather build your own.

This project follows a KISS approach, trying to do only a few things and heavily leveraging other open-source utilities to build the final microcontainer image.

## General overview

The basic steps:

1. Put your kernel in "listening" mode with `sysdig`;

2. Run all your tests against your target container to trigger its functionalities;

3. Export the information captured by the kernel in a human-readable format with the `microdump.lua` helper;

4. Compile and then run `micropacker` in a newly created target container instance, using the file generated at the previous step as input.

5. The output will be a minimal rootfs tar archive that can be referenced in a new Dockerfile to build a microcontainer image.

## Why micropacker-for-containers

The are basically three ways an executable can reference functions declared in a library:

* Static linking
* Dynamic linking
* Dynamic loading (look at `dlopen` [man page](https://linux.die.net/man/3/dlopen))

Dynamic loading is the real challenge for microcontainers, as you would need to disassemble your target binary, as well as all its libraries, to find every possible way the `dlopen` function can be called. Just as an example, see the [error](https://hackernoon.com/how-to-build-a-tiny-httpd-container-ae622c37db39#c81d) generated by a missing, dynamically loaded `libuuid` library in a microcontainer.

The good news is that not only the glibc `dlopen` call, but also virtually every file opened by the target binary can be intercepted by listening to syscalls. Instead of reinventing the wheel (or a system call table rootkit!), [sysdig](https://sysdig.com/opensource/) seems a good solution to listen to kernel events.

You are not forced to use `sysdig` provided that you give `micropacker` a compatible file in input, you can also write this input file by hand should you wish so. This project just makes your life easier providing a [chisel](https://github.com/draios/sysdig/wiki/Chisels-Overview) in the repository; chisels are small Lua scripts that allow you to extend core `sysdig` functionality. The `microdump.lua` chisel is used to generate a human readable file that is exactly what `micropacker` expects in input.

## Getting Started

For this README, we will use the development version of project [Admiral](https://hub.docker.com/r/vmware/admiral/), an open-source Java application based on Photon OS that you can download from Docker Hub by pulling the `vmware/admiral:dev` image.

### Prerequisites

#### On the target container

Unless you want to rebuild images, the only prerequisite is that the container has a `/bin/sh`.

#### On the host

Apart having a functioning Docker setup, satisfying the prerequisites on a Ubuntu/Debian system shouldn't be different from:

```console
user@host:~$ sudo apt-get install sysdig make
user@host:~$ sudo cp microdump.lua /usr/share/sysdig/chisels/
```

On a Photon OS system instead:

```console
root@host:~# tdnf install sysdig make
root@host:~# cp microdump.lua /usr/share/sysdig/chisels/
```

#### Static build of micropacker.go on the host

It is possible to have an executable that will run in every possible Linux container by statically linking Golang code against `musl` instead of `glibc`.
See this guide on [statically compiling Golang with musl](https://dominik.honnef.co/posts/2015/06/statically_compiled_go_programs__always__even_with_cgo__using_musl/) if you want to know more about this.

We have a Docker-based build that runs with a Makefile; the last step is optional and allows you to check that the final binary isn't using any dynamically linked library:

```console
user@host:~$ sudo make
user@host:~$ ldd micropacker/micropacker
	not a dynamic executable
```

### Put sysdig in listening mode on the host

Pull your target image and then ask `sysdig` to perform a raw capture of everything that happens when the container is started:

```console
user@host:~$ sudo docker pull vmware/admiral:dev
user@host:~$ sudo sysdig -qw kdump.cap container.image=vmware/admiral:dev &
```

Start the container as you would normally do:

```console
user@host:~$ sudo docker run -d -p 8282:8282 vmware/admiral:dev
```

### Run __ALL__ the tests you have

This is the key part: __the approach is as good as your QA__.

The kernel will see as many syscalls as deep your functional testing is. With a browser or any client of your application, trigger every functionality/feature, ideally with automated tests.

### Create a rootfs.tar archive from the container

Stop the `sysdig` capture and use `microdump.lua` to get your "needed files" list.

```console
user@host:~$ sudo kill [sysdig PID]
user@host:~$ sudo docker stop [cid]
user@host:~$ sudo sysdig -r kdump.cap -c microdump | sort | uniq > files.txt
```

Now create a new folder on your host, say `foo`, and after moving the statically compiled `micropacker` executable and the `files.txt` file there, run the following command:

```console
user@host:~$ sudo docker run -dit -v [path to folder foo]:/foobar --entrypoint=/bin/sh -u=0 vmware/admiral:dev
```

We've overridden the `ENTRYPOINT` tag and explicitly asked to run as root. Notice that the folder inside the container is named `foobar`, any non-existing folder name is fine (don't be tempted to use `/tmp`).

Execute `micropacker` and copy the output back to the host:

```console
user@host:~$ sudo docker exec -it [cid] /foobar/micropacker -i=/foobar/files.txt -o=/foobar/rootfs.tar -d=true
user@host:~$ sudo docker cp [cid]:/foobar/rootfs.tar rootfs.tar
```

### Build your micro image

Now that you have the `rootfs.tar` archive on the host, a microcontainer is just a matter of creating an appropriate Dockerfile.

Create a Dockerfile that contains the needed `ENV`, `VOLUME`, `WORKDIR`, `EXPOSE` tags as the original one... basically everything except the `FROM`, `RUN`, `COPY` and `ADD` tags. Do not forget Dockerfile tags that are set by parent Dockerfiles!

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
ENV PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
WORKDIR $ADMIRAL_ROOT
EXPOSE $ADMIRAL_PORT
VOLUME $ADMIRAL_STORAGE_PATH

ADD rootfs.tar /
ENTRYPOINT ["/entrypoint.sh"]
```

Now build the Dockerfile, start the microcontainer and then (try to!) explore with a `docker exec` the microcontainerized Admiral.

## FAQ

### I have a container I want to minimize, but I don't have a service listening on any port

That shouldn't be an issue, at the Q&A step trigger all the tests with a `docker exec` command instead of using a browser or a client.

### My image is _too_ minimal, I want to preserve some debuggability

Identify what commands are important for you (`ls`, `cat`, ...) and trigger them with a `docker exec` before stopping the kernel monitoring. Do not forget that if you use those commands against files that exist in the base image, those will be included too. Either use them against files that would be included anyway or modify the `files.txt` file before packing.

### Can this be automated?

Yes! The main problem of the legacy version was that an automation script should have known in advance the distribution of the target container to start the installation of the Python runtime libraries.

### Why don't you use strace?

Mostly for simplicity and perceived elegance. We have found some good reasons to avoid strace [in this blog post](http://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html) too.


### What's next for this project?

We are looking into providing more functionalities while maintaining the flexibility and the original KISS spirit of the project.

## License

The project is licensed under the BSD 2-clause license, see the [LICENSE.txt](LICENSE.txt) file for the full text.
Release binaries are not currently being considered, as they require additional work from a legal/licensing standpoint.
