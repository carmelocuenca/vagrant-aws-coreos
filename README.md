# A Vagrantfile for a CoreOS Cluster in Amazon Web Services

## Version 0.4
Flanneld is working in aws. Changes were needed in Vagrantfile and user-data.
udp Backend is used in order to keep things simple.

* The flannel unit for systemd was also changed ( avoid other subnets in the same vpc)

```
'{ "Network": "172.17.0.0/16/16", \
   "Backend": { "Type": "udp" } }'
```

* After a lot of hours, the flannel entry in coreos key in user-data was completely removed

In order to check flannel service, use docker inspect ... to get an ipaddress and from the other machine ping, something like this ```ping ip-addres-to-container-in-other-machine```.

## Version 0.3
The file ```use-data.sample``` define a unit (```systemd``` ) ```nginx.fleet.service``` which launchs a ```nginx.service``` in all
machine with metadata ```host=hello-world```. The file ```config.rb.sample```
instruments the metadata.

Listado de las unidades por MVs
```
$ for i in `seq 1 3`; do vagrant ssh core-0$i -c 'fleetctl list-units'; done
UNIT			MACHINE				ACTIVE	SUB
hello-world.service	5860ac61.../34.200.178.239	active	running
nginx.service		2c89b8dd.../34.205.222.231	active	running
Connection to ec2-34-205-222-231.compute-1.amazonaws.com closed.
UNIT			MACHINE				ACTIVE	SUB
hello-world.service	5860ac61.../34.200.178.239	active	running
nginx.service		2c89b8dd.../34.205.222.231	active	running
Connection to ec2-34-206-189-110.compute-1.amazonaws.com closed.
UNIT			MACHINE				ACTIVE	SUB
hello-world.service	5860ac61.../34.200.178.239	active	running
nginx.service		2c89b8dd.../34.205.222.231	active	running
Connection to ec2-34-200-178-239.compute-1.amazonaws.com closed.
```

Y comprobación de las unidades
```
$ for i in `seq 1 3`; do vagrant ssh core-0$i -c 'curl http://localhost/index.html'; done

```
## Version 0.2
A ```Vagrantfile``` to launch ```$num_instances``` with CoreOS in Amazon Web
Services (AWS). The file ```use-data.sample``` define a unit (```systemd``` ) ```hello-world.fleet.service``` which launchs a ```hello-world.service``` in all
machine with metadata ```host=hello-world```. The file ```config.rb.sample```
instruments the metadata.

Listado de MVs.

```
$ vagrant ssh core-01 -c 'fleetctl list-machines'
MACHINE		IP		METADATA
7ddca11c...	34.205.231.105	host=hello-world
95987f44...	34.197.162.32	host=hello-world
b51dec65...	34.205.3.71	host=hello-world
```
Listado de las unidades por MVs
```
$ for i in `seq 1 3`; do vagrant ssh core-0$i -c 'fleetctl list-units'; done
UNIT			MACHINE				ACTIVE	SUB
hello-world.service	95987f44.../34.197.162.32	active	running
Connection to ec2-34-205-231-105.compute-1.amazonaws.com closed.
UNIT			MACHINE				ACTIVE	SUB
hello-world.service	95987f44.../34.197.162.32	active	running
Connection to ec2-34-197-162-32.compute-1.amazonaws.com closed.
UNIT			MACHINE				ACTIVE	SUB
hello-world.service	95987f44.../34.197.162.32	active	running
Connection to ec2-34-205-3-71.compute-1.amazonaws.com closed.
```

## Version 0.1
A ```Vagrantfile``` to launch ```$num_instances``` with CoreOS in Amazon Web
Services (AWS). It also launchs a ```hello-world``` unit systemd.


## Requirements

It is necessary to have installed Vagrant (of course) and the plugin for using
AWS. All information about the plugin is in the URL https://github.com/mitchellh/vagrant-aws.

To install the vagrant-aws plugin execute
```
$ vagrant plugin install vagrant-aws
```

## What is each file?

* ```Vagrantfile``` What can I say about this file?
* ```user-data``` A cloud-config file to upload to AWS in setup time.
* ```config.rb``` Ruby code to instrument the ```user-data``` file.


## How does it work?
Setup an initial configuraion
```
$ cp user-data.sample user-data
$ cp config.rb.sample config.rb
```
an then up the infraestructure
```
$ vagrant up --provider aws
```

When ```vagrant up --provider aws``` is executed, Vagrant loads the file
```config.rb``` which adds a discovery token to ```user-data``` file and it
rewrite the file ```user-data```.
It uses the discovery service given by the cloud service (insecure) https://discovery.etcd.io/new?size=#{$num_instances}.
The global variable ```$num_instances``` fixs the number of instances.
This is not a solution for production environment because all machine needs Internet access.

The ```Vagranfile``` could set the fields  ```access_key_id``` and ```aws_secret_key```.
Insted of that the home directory contains the file ```~/.aws/credentials```.
This file is used by the API when both the access_key and the secret_key is not set.

```~/.aws/credentials``` file should look like similar to
```
[default]
aws_access_key_id=AKIAZZZZZZZZZZZZZZZZ
aws_secret_access_key=ZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZ
```

Don't forget to kill all EC2 instances (money is money)
```
$ vagrant destroy -f
```

## Service etcd2
After the infraestructure is deployed, we can check the service etcd2 with
```
vagrant ssh core-01 -c 'etcdctl member list'
```
or
```
$ vagrant ssh core-01 -c 'etcdctl cluster-health'
```
and the CoreOS should be healthy

## Service hello-world
Cheack the availability of service hello-world with
```
$ vagrant ssh core-01 -c 'journalctl -u hello-world'
```
or
```
$ vagrant ssh core-01 -c 'systemctl status hello-world'
```

## Is something wrong?
Obtain more logs from Vagrant

```
$ vagrant up --debug
```

* When we destroy and up quickly it happens the message
```InUse => Address 10.0.0.x is in use```. Be patient and try again.
* Are you in the free layer? You can only have 3 machines up, so check if you
have more than 3 machines in all zones.


# Creating a conteneraizing Rails example application

To create the application ```webapp```
```
$ docker run -it --rm --user "$(id -u):$(id -g)" -v "$PWD":/usr/src/app -w \
    /usr/src/app rails rails new --skip-bundle --database postgresql webapp
```

To generate a ```Gemfile.lock``` file.
```
$ cd webapp
$ docker run --rm -v "$PWD":/usr/src/app -w /usr/src/app ruby:2.3 \
    bundle install
```

Then, create a Dockerfile with this content
```
FROM ruby:2.3

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        postgresql-client nodejs \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src/app
COPY Gemfile* ./
RUN bundle install
COPY . .

EXPOSE 3000
CMD ["rails", "server", "-b", "0.0.0.0"]
```

To build the image
```
$ docker build -t webapp .
```

To run the image
```
$ docker run --name some-webapp -p 8080:3000 -d webapp
```
And to check the application
```
$ curl -L http://localhost:8080
```

To create dabases something similar to:

```
$ docker run  --link some-postgres:db \
  -e "POSTGRES_USER=${POSTGRES_USER}" \
  -e "POSTGRES_PASSWORD=${POSTGRES_PASSWORD}" \
  -v "$PWD":/usr/src/app -w /usr/src/app webapp  bin/rails db:create

```

To create a resource
```
$ docker run  --user "$(id -u):$(id -g)" -v "$PWD":/usr/src/app \
    -w /usr/src/app --link some-postgres:db \
    -e "POSTGRES_USER=${POSTGRES_USER}" \
    -e "POSTGRES_PASSWORD=${POSTGRES_PASSWORD}" webapp bin/rails g scaffold \
    article title:string body:text
```

To migrate
```
$ docker run  --link some-postgres:db \
  -e "POSTGRES_USER=${POSTGRES_USER}" \
  -e "POSTGRES_PASSWORD=${POSTGRES_PASSWORD}" \
  -v "$PWD":/usr/src/app -w /usr/src/app webapp  bin/rails db:migrate

```


To run the image
```
$ docker run  --rm -it --link some-postgres:db \
  -e "POSTGRES_USER=${POSTGRES_USER}" \
  -e "POSTGRES_PASSWORD=${POSTGRES_PASSWORD}" \
  -v "$PWD":/usr/src/app -w /usr/src/app -p 8080:3000 webapp
```

And setup the root page as usual



## More?
* Multitenant
* ELK
* Monitoring
* Scaling
* Terraform
* EBS for volumes
* Production? TLS, SECRET_KEY
* Cluster formation without public internet access
* Change udp by aws-pc in flannel
