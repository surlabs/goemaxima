goemaxima
=========
This project developed at the Mathematical Institute of Goettingen provides an alternative webservice for stack, which is a question type for ilias and moodle focused on math.
The image was originally developed to be used in a cluster/loadbalancing setup, but for most workloads it is viable to use it as a single container.

This implementation in golang strives to have faster startup and processing times than the corresponding java version by using fork instead of starting a new process for each request.
For some more information on how this works, [see the documentation](/doc/How_it_works.md).


Building a Docker Image
=======================

A docker image can be built by first building the web server with buildweb.sh, which will place the web server executable into `./bin/`.
In order for that to work, `go` needs to be installed.

After that, the docker container for a particular stackmaxima version can be built by invoking `buildimage.sh STACKMAXIMA_VERSION` with `STACKMAXIMA_VERSION` substitutet to be the stackmaxima version to base the container off.

Example:
```
$ ./buildweb.sh
$ ./buildimage.sh 2020061000
```

The image should then be available as `goemaxima:2020061000-dev`.

The supported stackmaxima versions can be seen by looking at the versions file of the root of this repository.

Using the Docker Image
======================

The prebuilt docker images are available from the docker hub at `mathinstitut/goemaxima:[version]-latest`.

They can be simply run with docker-compose in the top-directory of the repository (setting the `STACKMAXIMA_VERSION` to the desired value):
```
export STACKMAXIMA_VERSION=...
docker-compose up -d
```
The container should then be available on port 8080 (from outside the host too, keep this behind a firewall so it is not reachable from the general internet).

The port of the server inside the container is `8080` and the path that has to be input into stack is `http://[address:port]/goemaxima`.
Some older versions of the image accept only `http://[address:port]/maxima` and this url should work in newer versions as well.

If you do not wish to use the docker-compose configuration, you can also run the image with
```
$ docker run --restart=always --tmpfs /tmp -p $address:$port:8080 $imagename
```
where `$address:$port` is the ip and port you want to make the service available on and `$imagename` is the name of the docker image you chose (e.g. `mathinstitut/goemaxima:2020120600-latest`).
Use `0.0.0.0` as address to listen to all addresses.

Note that this program prefers to quit on errors it can not recover from, so setting `restart=always` is strongly recommended.

The memory usage of the container is around 50MB when idle and can quickly spike to 250MB on heavy use.
The process startup time for a single request is a few milliseconds and negligible in comparison to processing the CAS request itself.

For advanced users, one can also deploy the image in kubernetes by using the helm chart provided in the `helmmaxima` directory.

Environment Variables
=====================
A few internal options can be set from environment variables.

* `GOEMAXIMA_DEBUG` activates some debug logs with `GOEMAXIMA_DEBUG=1` (or more with `GOEMAXIMA_DEBUG=2`).

* `GOEMAXIMA_EXTRA_PACKAGES` is a colon-separated list of maxima packages that gets loaded at startup at the server.
   The load time of the packages mostly influences the startup time of the server itself and should not significantly affect the serve time of a request.
   The entries simply get loaded through a load(entry) call in maxima.
   This means you can also mount your own maxima libraries into the docker image and load them by using a path.

* `GOEMAXIMA_NUSER` sets the number of processes that can run at the same time. The maximum number is 32 and it is the default.

* `GOEMAXIMA_QUEUE_LEN` sets the target number of processes that are idle and ready to accept a request. The default is 3 (but note that there is always one extra process waiting).

* `GOEMAXIMA_TEMP_DIR` is the temporary directory where maxima processes write plots and have their named pipes. Its default is `/tmp/maxima`. It is the only location where the server writes to and should be mapped to tmpfs to increase speed.

* `GOEMAXIMA_LIB_PATH` is the path to a library that is loaded on startup of the maxima process.
   If you do not modify the container or use the webserver executable in another context than the image, you probably want to leave this variable unset.
   If you want to load extra packages/libraries, use the `GOEMAXIMA_EXTRA_PACKAGES` variable instead.

Metrics
=======
Prometheus metrics are published on `/metrics` and include number of success, timeout and internal error responses and also histograms of process startup and response times.

License
=======
goemaxima is licensed under the GPLv3.
