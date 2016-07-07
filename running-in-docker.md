# how to run sparkhara apps in docker

## components

* ghost-pathfinder, this is the data feeding application. it will listen on
  a user specified socket and write newline terminated messages to a receiver.
* shiny-squirrel, this is the http application that will create the web page
  for viewing processed data. it requires a mongo database to retrieve log
  data, and metadata.
* whirlwind-caravan, this is the main spark application. it will receive
  messages from the ghost-pathfinder and signal the shiny-squirrel as well as
  write data to a mongo database.
* mongo database, this is required for the storage of log data and metadata

## image setup

first you will need to create all the images necessary for the sparkhara
apps. all of the applications required have dockerfiles in their root
directories.

for shiny-squirrel and whirlwind-caravan, creating the images requires
cloning the repository and running the docker build.

### shiny-squirrel

    $ git clone https://github.com/sparkhara/shiny-squirrel
    $ cd shiny-squirrel
    $ docker build -t shiny-squirrel .

### whirlwind-caravan

    $ git clone https://github.com/sparkhara/whirlwind-caravan
    $ cd whirlwind-caravan
    $ docker build -t whirlwind-caravan .

for the ghost-pathfinder, a gzip'd file containting the lines to be processed
should be installed in the root directory of the repository before building
the image. this is a simple shortcut to have the data file included in the
image.

### ghost-pathfinder

    $ git clone https://github.com/sparkhara/ghost-pathfinder
    $ cd ghost-pathfinder
    $ cp /path/to/logdata.gz .
    $ docker build -t ghost-pathfinder .

also, a mongo database should be installed. for simplicity, i have been using
the upstream mongodb image from docker hub. this image has no authentication
limitations on its use.

### mongodb

    $ docker pull docker.io/mongo

## running the containers

the main points to keep track of while spawning the containers for sparkhara
are the linkage between the components and the environment variables that
must be passed into the applications.

to start with, i spawn the mongo database

    $ docker run -d --name sparkhara-mongo mongo

*in the following commands i make use of the `-it` option to attach the
 terminal to my docker containers. this can be ommitted if need be, my main
 rationale  for this is to see the output and have the ability to kill the
 commands with a CTRL+c.*

next, i start the ghost-pathfinder

    $ docker run --rm -it --name ghost-pathfinder \
      -e GHOST_PATHFINDER_LOGFILE=/opt/ghost/logdata.gz \
      -e GHOST_PATHFINDER_TIMESCALAR=100 \
      ghost-pathfinder

notice that i am passing in 2 environment variables.
* `GHOST_PATHFINDER_LOGFILE` specifies the location of the data file to be
  processed. during image creation, all files in the repository will be copied
  to `/opt/ghost/` in the container.
* `GHOST_PATHFINDER_TIMESCALAR` is an integer to define how fast the logs
  should be played back. by default, the scalar is 1, meaning realtime. any
  number greater than 1 will attempt to play the log data back at that
  multiple of realtime.

with the pathfinder running, i now start the shiny-squirrel

    $ docker run --rm -it -p 9050:9050 --name shiny-squirrel \
      --link sparkhara-mongo \
      -e SHINY_SQUIRREL_MONGO_URL=mongodb://sparkhara-mongo/sparkhara \
      shiny-squirrel

this container must link to the mongo database instance, and we must pass
the url as an environment variable. in this case, we need no credentials to
speak with the mongo database and the url reflects this.
also, it is worth noting that we are exposing the port that the http server
will use for communication. this can be modified for the host, but should
always be `9050` inside the container.

at this point i am ready to start the whirlwind-caravan

    $ docker run --rm -it --name whirlwind-caravan \
      --link ghost-pathfinder \
      --link shiny-squirrel \
      --link sparkhara-mongo \
      -e WHIRLWIND_CARAVAN_MONGO_URL=mongodb://sparkhara-mongo/sparkhara \
      -e WHIRLWIND_CARAVAN_REST_URL=http://shiny-squirrel:9050/count-packets \
      -e WHIRLWIND_CARAVAN_SOCKET=ghost-pathfinder \
      -e SKIP_COMMON=true \
      whirlwind-caravan

ok, that's a mouthful. let's break it down.
we need to link with the mongo database, ghost-pathfinder, and shiny-squirrel,
this linkage will enable all the communication we need using the internal
dns provided by docker.
there are also several environment variables which must be configured, the
mongo url should be fairly obvious and is similar to what we've done with the
shiny-squirrel.
* `WHIRLWIND_CARAVAN_REST_URL` is the endpoint that the whirlwind-caravan will
  signal when new logs have been processed. this value is highly specific to
  the sparkhara application pipeline.
* `WHIRLWIND_CARAVN_SOCKET` is the name of the host to attach for the text
  stream socket from spark. in this case we are simply attaching to the
  ghost-pathfinder container.
* `SKIP_COMMON` is a minor hack for running on docker as opposed to openshift,
  it simply skips some configuration that must be performed for openshift.

with all these containers running you should start to see communication
happening between them. if you have chosen to keep the `-it` option on these
docker runs, then you will see the log output from each container. all these
applications will make some noise when they are communicating properly.
ghost-pathfinder should emit logs about receiving a connection and sending
bytes of log data.
shiny-squirrel should emit logs for all http requests, you should see GETs
when using a browser, and POSTs from the whirlwind-caravan.
whirlwind-caravan is a spark app and will be quite noisy, keep an eye out
for exception traces as that will be a clear sign that something is not
quite right.

if all the logs look normal and happy you should be able to open a browser
pointed at `http://localhost:9050` and see a few graphs with data coming
through.

