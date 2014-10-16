Installing Travis CI Enterprise
===============================

**Please email enterprise@travis-ci.com for more information on pricing and to register for a 45 day trial.**

## Prerequisites

- Two dedicated hosts or hypervisors (KVM/QEMU or VMWare) with Ubuntu 14.04 installed
- A Travis CI Enterprise license file
- A GitHub Enterprise OAuth app


## Host Machines

The standard setup consists of two hosts, a te-main server which hosts the web UI and 
related services, and a te-worker server which runs the tests/jobs in isolated containers using LXC and Docker.

If you are using EC2 then we recommend the instance types:

* `te-main`:   c3.2xlarge
* `te-worker`: c3.2xlarge

Otherwise we recommend hosts with 16 gigs of RAM and 8 CPUs.


## Obtaining a license.yml file

**Please email enterprise@travis-ci.com for your license.yml file.**

Please copy/paste or upload the generated license to the te-main box and place 
it in the root users home directory (/home/license.yml).


## Register a GitHub OAuth app

Travis CI talks to GitHub Enterprise via OAuth. You will need to create an OAuth app 
on your GitHub Enterprise which Travis CI Enterprise can connect to.

The OAuth app registered will use the domain name pointing to your te-main host for 
the Homepage URL (e.g. https://travis-ci.your-domain.com). Append /api to this for 
the Authorization callback URL (e.g. https://travis-ci.your-domain.com/api).


## Installation

### Setting up te-main

During this process you will be asked for:

* the hostname of your te-main box
* the GitHub endpoint, it's fine to just use github.com
* credentials for a GitHub OAuth application for te-main

It will also ask you for other configuration values, which are optional (e.g.
email smtp settings). It is fine to just hit enter and leave these empty.

For the GitHub OAuth application set "Homepage URL" to the hostname of your
te-main box (e.g. https://te-dev.travis-ci.com) and "Authorization callback
URL" to `/api` on your te-main box (e.g. https://te-dev.travis-ci.com/api).

Before running the following commands, please make sure you are logged in as the root user first.

If you are running the host on EC2 then please run the following command:

```
export AWS=true
```

Then run:

```
curl -R https://raw.githubusercontent.com/travis-ci/enterprise-installation/master/setup-main.sh | bash
```


### Setting up te-worker

For setting up the `te-worker` box you'll need the RabbitMQ password which was
generated by setting up `te-main`. See above.

Before running the following commands, please make sure you are logged in as the root user first.

If you are running the host on EC2 then please run the following command:

```
export AWS=true
```

Then run:

```
curl -R https://raw.githubusercontent.com/travis-ci/enterprise-installation/master/setup-worker.sh | bash
```

Once rebooted the te-worker container should start and connect to te-main automatically.


## Maintenance

### Updating Travis CI Enterprise

In order to update the Docker images and restart the Web UI and Worker you can run the following on each host:

```
te pull
te start
```


### Inspecting logs and running services

On both hosts the logs are located at /var/travis/log/travis.log, but also symlinked to /var/log/travis.log for convenience.

Also, on both boxes there's an `~/.ssh/config` host configured named `te-main`
and `te-worker` respectively. You should be able to ssh into the Docker
containers by running `ssh te-main` and `ssh te-worker` on the respective
boxes.


### Accessing the UI

A self-signed certificate is generated for your hostname. When you point your browser to your te-main hostname you'll get an SSL warning and it might not let you access the page until you've trusted the certificate.
If you are using Chrome on Mac OS X then you may want to import and explicitly add and trust a certificate to the Mac OS X keychain. You can find instructions on how to do that here: https://github.com/travis-ci/enterprise-installation/blob/master/self-signed-ssl-cert.md
If you have signed into your Travis CI Enterprise UI previously, or re-installed your Web UI host, you may see an empty page with loading indicators. This is a known bug in our UI. To fix this please sign out and back in.


### Reconfiguring the container

To re-enter or change your configuration (e.g. fix a host name or add email smtp configuration), please run:

```
te configure --prompt
te start

# or:
te start --prompt
```

### Configuring your installation with advanced options

During normal install you'll be asked to provide a few required configuration
settings such as the instance's hostname and GitHub OAuth credentials.
However, there are more configuration settings that can be specified.

E.g. in order to enable email notifications you can provide settings for your
SMTP service like so:

```
te configure --prompt --group smtp
```

The following configuration groups are currently available:

```
# on te-main
ghe      - GHE endpoint configuration options
graphite - graphite endpoint for collecting metrics (beta)
smtp     - SMTP service for email notifications

# on te-worker
rabbitmq - RabbitMQ configuration options
s3       - S3 bucket credentials for dependency caching
worker   - number of VMs to be used
```

### Specifying an SSL certificate

During installation of the `te-main` container, a self-signed certificate is generated for your
hostname. In order to provide your own SSL certificate you can replace the
following two files:

```
/etc/travis/ssl/your_hostname.bundle
/etc/travis/ssl/your_hostname.key
```

Then restart the `te-main` container:

```
te start
```

### Specifying a custom root certificate

If your GHE instance uses a special (e.g. self-signed) root certificate, then you will want to import this to your
`te-main` instance so it can connect via SSL.

In order to provide a root certificate you can place it in:

```
/etc/travis/ssl/ca-certificates/some-name.crt
```

Then restart the `te-main` container:

```
te start
```

`te-main` will look for files in this directory, symlink them to `/usr/shared/ca-certificates`
and run `update-ca-certificates`.


### Starting a build container on te-worker

In order to start a build container on the te-worker you can do the following:

```
# start a container and grab the port
id=$(docker run -d -p 22 travis:php /sbin/init)
port=$(docker port $id 22 | sed 's/.*://')

# ssh into the container (the default password is travis)
ssh travis@localhost -p $port

# stop and remove the container
docker kill $id
docker rm $id
```

### Customizing build images

Once you have pulled the build images from quay.io, and they've been re-tagged to
`travis:[language]` you can fully customize these images according to your needs.

Be aware that you'll need to re-apply your customizations after upgrading build
images from quay.io.

The basic idea is to:

* start a Docker container based on one of the default build images `travis:[language]`,
* run your customizations inside that container, and
* commit the container to a Docker image with the original `travis:language` name (tag).

For example, in order to install a particular Ruby version which is not available on
the default `travis:ruby` image, and make it persistent, you can run:

```
docker run -it --name travis_ruby travis:ruby su travis -l -c 'rvm install [version]'
docker commit travis_ruby travis:ruby
```
