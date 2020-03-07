# redis-bosh-deployment
BOSH redis deployment

We need a new BOSH release and job, so let’s run the commands that we know and love:

```
bosh init-release --dir redis-release --git
cd redis-release
bosh generate-job redis
```
Let’s download Redis and put it in the src directory:

`curl http://download.redis.io/releases/redis-4.0.9.tar.gz | tar zx -C src -`

There should now be a directory called src/redis-4.0.9. You’ll see that it’s C source code with a Makefile.

Note that v4.0.9 was the latest stable version when this was written, but the stable version will change over time. Feel free to use a later version, though you may have to make adjustments.

Now we need to tell BOSH how to package up the Redis code. We do that with a BOSH package. BOSH can create a skeleton for us:

`bosh generate-package redis`

This will create a directory called packages/redis containing spec and packaging files.

The package spec file tells BOSH where to find your package. BOSH will look first in src. Edit the spec to let BOSH know where to find Redis
```
---
name: redis

files:
- redis-4.0.9/**
```
The packaging file tells BOSH how to build your source. For Redis, change the packaging script to be:

```
set -ex

pushd redis-4.0.9
  make
  make PREFIX=${BOSH_INSTALL_TARGET} install
popd
```
Now the job needs to know how to run Redis. The Redis server should run as a background task (daemonized), so we will configure it to do that with a `jobs/redis/templates/redis.conf` Redis configuration file:
```
daemonize yes
logfile /var/vcap/sys/log/redis/redis.log
pidfile /var/vcap/sys/run/redis/pid
```
We also need a script to start and stop the Redis server. The file should be named `jobs/redis/templates/ctl.sh`. Here’s the one that we used:
```
#!/usr/bin/env bash

RUN_DIR=/var/vcap/sys/run/redis # PID file goes here
LOG_DIR=/var/vcap/sys/log/redis

case $1 in

  start)
    mkdir -p $RUN_DIR $LOG_DIR
    chown -R vcap:vcap $RUN_DIR $LOG_DIR

    /var/vcap/packages/redis/bin/redis-server \
    /var/vcap/jobs/redis/redis.conf \
    1>> $LOG_DIR/redis.out \
    2>> $LOG_DIR/redis.err

    ;;

  stop)
    /var/vcap/packages/redis/bin/redis-cli shutdown

    ;;

  *)
    echo "Usage: ctl {start|stop}" ;;

esac
```
Edit the job spec file jobs/redis/spec to tell BOSH about the Redis script and configuration. Change the templates and packages stanzas to say:
```
templates:
  ctl.sh: bin/ctl
  redis.conf: redis.conf

packages:
- redis
```
BOSH uses Monit to manage running processes. We need to tell Monit how to start, stop and monitor Redis. We do this by changing the file `jobs/redis/monit` to:
```
check process redis
  with pidfile /var/vcap/sys/run/redis/pid
  start program "/var/vcap/jobs/redis/bin/ctl start"
  stop program "/var/vcap/jobs/redis/bin/ctl stop"
  group vcap
```
See how Monit will use the script ctl to start and stop Redis, and will monitor the health of Redis by reading the process ID from /var/vcap/sys/run/redis/pid file and checking that the process is still running. The Redis configuration file tells Redis to create this file.

As before, we need to create and upload the release:
```
bosh create-release --force
bosh upload-release
```
And as before, we need a BOSH manifest to describe the deployment. The only differences from the “Hello world!” manifest are the names:
```
name: redis-deployment

releases:
- name: redis
  version: latest

stemcells:
- alias: default
  os: ubuntu-trusty
  version: latest

update:
  canaries: 1
  max_in_flight: 1
  canary_watch_time: 1000-30000
  update_watch_time: 1000-30000

instance_groups:
- name: redis-server
  azs: [z1]
  instances: 1
  vm_type: default
  stemcell: default
  networks:
  - name: default
  jobs:
  - name: redis
    release: redis
    templates:
    - name: redis
      
```
Deploy the release with the deployment manifest:

`bosh -d redis-deployment deploy manifest.yml`

Check that the Redis is running with the command:

`bosh -d redis-deployment instances`

