# osx-docker-openshift What is this?

These are notes regarding our installation of OpenShift running in docker on Mac OSX.

In particular, we want to achieve the following:

* installation of docker on our Mac OSX host, using `docker-machine`
* installation of OpenShift via its docker image, running in our `docker-machine` vm
* the ability to interface with the OpenShift web console on our Mac OSX host

Our main objective is to be able to get OpenShift running under docker, such that we can run it up anywhere
(Mac OSX, EC2, Digital Ocean etc).


## Installation

Install [homebrew](http://brew.sh/)

    /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

Install [dockertoolbox](https://docs.docker.com/mac/step_one/)

    brew install Caskroom/cask/dockertoolbox

Run `docker-machine` to create the VM

    # create the docker machine
    docker-machine create --driver virtualbox --engine-insecure-registry 0.0.0.0/0 vm-docker

Start the vm

    # start the docker machine
    docker-machine start vm-docker

    # this command allows the docker commands to be used in the terminal
    eval "$(docker-machine env vm-docker)"

If we don't create the VM with `--engine-insecure-registry`, we can always add it to the local config
afterwards:

    vim  ~/.docker/machine/machines/vm-docker/config.json

    # Add this:
    "InsecureRegistry": ["0.0.0.0/0"],

We can also do this directly in our docker-machine vm:

    # ssh into our vm running docker
    docker-machine ssh vm-docker

    sudo vi /var/lib/boot2docker/profile

    # Add the following two lines to the `procfile`
    # (`DOCKER_TLS=no` suppresses output on the Mac OSX host terminal)
    DOCKER_TLS=no
    EXTRA_ARGS="--insecure-registry 192.168.59.103:5000 --insecure-registry dockerhost:5000 --insecure-registry 10.245.2.3:5000 --insecure-registry 10.245.2.4:5000"

Port forward 8443 in virtualbox:

    Protocol: 443
    Host IP: 127.0.0.1
    Host Port: 8443
    Guest Port: 8443

Use docker from our host to run openshift:

    docker run \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v /:/rootfs:ro -v /var/run:/var/run:rw -v /sys:/sys -v /v/lib/docker:/var/lib/docker:rw \
      --name=origin \
      --net=bridge \
      -p 8443:8443 \
      --privileged \
      openshift/origin start --listen=https://0.0.0.0:8443/ --public-master=https://127.0.0.1:8443 --loglevel=4 && \
    docker exec -ti origin bash

Note that we have to let openshift know that it will be serving on 127.0.0.1: `--public-master=https://127.0.0.1:8443`

In a new tab:

    # this command allows the docker commands to be used in the terminal
    eval "$(docker-machine env vm-docker)"

    # ssh into our vm running docker
    docker-machine ssh vm-docker

Now that we're in `vm-docker`:

    # We want to check the IP of our openshift/origin container
    docker ps

    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                            NAMES
    cb776495a7b9        openshift/origin    "/usr/bin/openshift s"   6 minutes ago       Up 6 minutes        53/tcp, 0.0.0.0:8443->8443/tcp   origin

    # Either run a bash shell in our container:
    docker exec -it cb776495a7b9 bash

        $ openshift cli status

    # Or simply query the container from our docker machine vm
    docker exec -it cb776495a7b9 openshift cli status

        $ In project default on server https://172.17.0.2:8443

          svc/kubernetes - 172.30.0.1 ports 443, 53, 53

          View details with 'oc describe <resource>/<name>' or list everything with 'oc get all'.

    # Ensure that we can curl our openshift console page from our docker-machine vm:
    curl -ik  https://172.17.0.2:8443/console/

    # exit the docker-machine vm
    exit

From our Mac OSX host command line:

    # We should be able to curl our openshift console page on localhost:
    curl -ik  https://127.0.0.1:8443/console/

Visit the console in our browser:

    https://localhost:8443/console/

Initially, you can use any username / password, and OpenShift will use this to create an account,
such that you can use these credentials on successive logins.


We can also access our account directly in the OpenShift container

    # ssh into our vm running docker
    docker-machine ssh vm-docker

    # We want to check the IP of our openshift/origin container
    docker ps

    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                            NAMES
    cb776495a7b9        openshift/origin    "/usr/bin/openshift s"   6 minutes ago       Up 6 minutes        53/tcp, 0.0.0.0:8443->8443/tcp   origin

    # Either run a bash shell in our container:
    docker exec -it cb776495a7b9 bash

    # Now that we're in the OpenShift container, login:
    oc login https://localhost:8443


## References:

https://kurtstam.github.io/2015/03/31/Getting-started-with-Fabric8-and-OpenShift-v3-on-OSX.html
https://sosiouxme.wordpress.com/2015/01/02/openshift-3-from-zero/
https://blog.openshift.com/openshift-v3-deep-dive-docker-kubernetes/
https://zwischenzugs.wordpress.com/2015/04/01/play-with-an-openshift-paas-using-docker/
https://www.manning.com/books/docker-in-practice?a_bid=e0d48f62&a_aid=zwischenzugs
https://github.com/openshift/openshift-ansible
https://blog.openshift.com/quick-tip-running-wildfly-docker-image-on-openshift-origin/
https://github.com/openshift/origin/blob/master/examples/sample-app/container-setup.md
https://github.com/dlbewley/dlbewley.github.io/blob/master/_posts/2015-06-28-Testing-Openshift-Origin-V3-with-Ansible-and-Vagrant-on-OS-X.md
