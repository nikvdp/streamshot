# Streamshot

Streamshot is a ready-to-go [mitmproxy][mitmproxy] setup, that allows you to test your
code in a Docker container, while analyzing the resulting HTTP / HTTPS flows. The only
prerequisite is to install [VirtualBox][virtualbox] and [Vagrant][vagrant].

## How does it work?

Create a virtual machine using the provided Vagrantfile. Then create one or more docker
container(s) from the provided docker image. Run your code in a container, and inspect
the outbound HTTP / HTTPS traffic on the host (the vagrant box) thanks to mitmproxy.

For instance, you could use streamshot this way:

- create a docker container and run a web client that will interact with an external
server. You can use any kind of client, even X applications (see more on that below).
- create two docker containers: one to host a web server and one to run client code.
Again, the client can be any kind of application: a library, a browser, an android
emulator...

Streamshot does the following for you:

- It creates an Ubuntu 15.10 vagrant box
- It installs mitmproxy on the vagrant box
- It generates the certificate for mitmproxy CA
- It mounts a shared folder containing your projects on the vagrant box
- It adds a helper function to vagrant user's .bashrc to conveniently activate or
deactivate iptables redirection rules
- And finally it builds an Ubuntu 15.10 docker image and adds mitmproxy's CA certificate
to the list of system trusted certificates (/etc/ssl/certs/ca-certificates.crt)

## How to use it?

Clone or fork this repository and adjust the content of the Vagrantfile to fit your needs.
You will at least have to change the following variable:

```
LOCAL_FOLDER = "/Users/yann/Projects"
```

And probably this one too:

```
VAGRANT_TIMEZONE = "Europe/Paris"
```

Then run the following command:

```
$ vagrant up
```

The first build will complete after a few minutes. The use of a proxy is not supported
for now, so you should not use any proxy to launch the first 'vagrant up' command. Once
the vagrant box is installed, you will still be able to use it behind a proxy (see below).

At the end of the build, log into the box:

```
$ vagrant ssh
```

The user vagrant belongs to the docker group, so you can issue docker commands:

```
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
docker_base         latest              20ae54d270da        5 minutes ago       164 MB
ubuntu              15.10               3e0c71ada2db        5 weeks ago         133.5 MB
```

At this point, there is no active container. Run the test container with ```docker run```
command, taking care of mounting a volume for your project directory. For instance, to mount
/home/vagrant/projects on /projects inside the container, you would have to type the following:

```
$ docker run -d --name="client" -v /home/vagrant/projects:/projects docker_base
```

To check container state:

```
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
26ba0f2cbbda        docker_base         "/sbin/init"        5 seconds ago       Up 4 seconds                            client
```

To log into the container:

```
$ docker exec -it client bash
```

A docker container uses Etc/UTC timezone by default, and you may get wrong time information.
You can fix it in different ways. The first one is to configure an explicit timezone in the
container (you can add this to the Dockerfile if you want):

```
# echo "Europe/Paris" > /etc/timezone
# dpkg-reconfigure tzdata
```

The second one is to mount the host's /etc/locatime file to the container at runtime:

```
$ docker run -d --name="client" -v /etc/localtime:/etc/localtime:ro -v /home/vagrant/projects:/projects docker_base
```

In another terminal, log into the vagrant box:

```
$ vagrant ssh
```

For now, the container has direct access to the outside world. The first thing to do is
to activate the redirection:

```
$ redirection on
```

The ```redirection```function runs the following commands (see .bashrc file) on the host:

```bash
sudo iptables -t nat -I PREROUTING -p tcp --dport 80 -j REDIRECT --to 8080 -w
sudo iptables -t nat -I PREROUTING -p tcp --dport 443 -j REDIRECT --to 8080 -w
```

Then run mitmproxy in transparent mode:

```
$ mitmproxy -T --host
```

In the 'client' docker container, test the setup:

```
# curl 'https://github.com'
```

The request should appear on the mitmproxy window.

**Note:** the 'base_docker' image is intentionally very light, because I can not presume
what kind of application you intend to run. So you will have to customize the container
to install everything you need prior to making your tests.

**Note:** sometimes you may want to temporarily deactivate redirection rules, to avoid
polluting mitmproxy logs with queries that are of no interest. That is exactly what the
'redirection' function is for. You can switch on / switch off iptables rules without
shutting down mitmproxy, and your container will still be able to communicate with the
outside.

## What if I need a GUI?

I will describe here a procedure that has been tested on Mac OS X, but it should work almost
the same way under Linux or Windows. For Mac OS X, here are the steps:

* Install xquartz and socat:

```
$ brew cask install xquartz
$ brew install socat
```

* Run xquartz

By default, xquartz is launched with the ```-nolisten tcp``` flag and connects to a UNIX
domain socket:

```
$ echo $DISPLAY
/private/tmp/com.apple.launchd.nUPyy0QxtI/org.macosforge.xquartz:0
```

Your X client application will try to connect to the port 6000, so we need to forward data
from a TCP socket to the UNIX one, which can be done with socat. Run this command in your terminal:

```
$ socat TCP-LISTEN:6000,reuseaddr,fork UNIX-CLIENT:\"$DISPLAY\"
```

And that is pretty much. The IP address of the X server is already configured in the docker
container (it belongs to the private network created by vagrant). In the container:

```
# echo $DISPLAY
192.168.10.1:0
```
* Run the client application

## A word about certificates

As stated above, the mitmproxy CA certificate has been added to the list of certificates
trusted by the container. But it may or may not work "out of the box", depending on the
validation mechanism your application relies on. For instance, browsers have their own
trustores. I you want to manually add the certificate - say to Firefox, follow these steps:

* Open menu
* Select 'Preferences' -> 'Advanced' -> 'Certificates'
* Click on the 'View Certificates' button
* Select the tab 'Authorities', and click on the 'Import' button
* Select the file '/usr/share/ca-certificates/mitmproxy-ca-cert.pem' and open it
* On the pop up window, check the box 'Trust this CA to identify websites.'

## What about proxies?

Once the environment is build, you can use it behind a proxy. However, it involves
a different approach that the one exposed above. First you have to configure your
application to use an explicit proxy, mitmproxy in this case. It could be as simple
as doing something like this in your container:

```
$ export http_proxy="http://192.168.10.1:8080"
$ export https_proxy="http://192.168.10.1:8080"
```

Secondly, you have to launch mitmproxy in upstream proxy mode:

```
$ mitmproxy -U http://hostname[:port]
```

Change 'hostname' and 'port' with their respective values.

[mitmproxy]: http://mitmproxy.org
[virtualbox]: https://www.virtualbox.org
[vagrant]: https://www.vagrantup.com
