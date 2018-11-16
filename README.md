# fuseki-starter

Here are resources dedicated to starting up a [Fuseki](https://jena.apache.org/documentation/fuseki2/) server, in particular on an AWS EC2 instance.

## Setting up AWS

Here are directions for preparing an AWS EC2 server to serve as a host to a Fuseki server.

I'll assume you've already established an account on AWS, given them your credit card number and signed over your first-born child.

If you haven't already, follow [these instructions](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) to get a key pair with an associated [pem file](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail). Give this file a meaningful name and keep it in an easy-to-remember place.

Follow [these instructions](https://aws.amazon.com/ec2/getting-started/) to start a basic EC2 server.
  
The default values should work fine (I chose an Ubuntu image), until you get to <em>Step 6: Configure [Security Group](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html)</em>, at which point you should use the `add-rule` button to add a rule with these parameters:
```
  Type = custom TCP
  Protocol = TCP
  Port Range = 3030
  Source = 0.0.0.0/0
  Description = Opens the default Fuseki SPARQL endpoint port
``` 
then press the 'review and launch' button.

I'm working from an ubuntu machine. I keep an entry like this in my
  .ssh/[config file](https://www.ssh.com/ssh/config/):
  
```
Host fuseki-1 
  HostName ec2-yadda-yadda.amazonaws.com
  User ubuntu
  IdentityFile /path/to/aws_key_pair.pem
```

...where `ec2-yadda-yadda.amazonaws.com` is the value of the [Public DNS
column](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.html#vpc-dns-viewing) on your AWS instances console

This allows me to log in to a remote secure shell on the server from my local linux computer using

```
$ ssh fuseki-1
```
No doubt you can do something similar on Windows or Apple.

Using the above command should bring you into a secure shell that looks like this:
```
$ ll
total 32
drwxr-xr-x 5 ubuntu ubuntu 4096 Nov 15 18:31 ./
drwxr-xr-x 3 root   root   4096 Nov 15 17:54 ../
-rw-r--r-- 1 ubuntu ubuntu  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 ubuntu ubuntu 3771 Apr  4  2018 .bashrc
drwx------ 2 ubuntu ubuntu 4096 Nov 15 18:31 .cache/
drwx------ 3 ubuntu ubuntu 4096 Nov 15 18:31 .gnupg/
-rw-r--r-- 1 ubuntu ubuntu  807 Apr  4  2018 .profile
drwx------ 2 ubuntu ubuntu 4096 Nov 15 17:54 .ssh/
```

Java is not installed yet, so...

```
$ sudo apt-get update
$ sudo apt-get install openjdk-8-jdk
$ java -version

openjdk version "1.8.0_181"
OpenJDK Runtime Environment (build 1.8.0_181-8u181-b13-1ubuntu0.18.04.1-b13)
OpenJDK 64-Bit Server VM (build 25.181-b13, mixed mode)
```

## Installing Fuseki on a Linux machine

This section picks up from the point we left off in the previous section, but it should apply to Linux systems generally. 

From here we want to follow [the Fuseki installation doc](https://jena.apache.org/documentation/fuseki2/).

I'll install fuseki in a directory called `~/opt/fuseki`:

```
$ mkdir opt
$ cd opt
$ mkdir fuseki
$ cd fuseki
$ wget http://mirror.reverse.net/pub/apache/jena/binaries/apache-jena-fuseki-3.9.0.tar.gz
$ tar -xvzf apache-jena-fuseki-3.9.0.tar.gz
$ rm apache-jena-fuseki-3.9.0.tar.gz
$ cd apache-jena-fuseki-3.9.0
# For convenience, we set an environment variable....
$ export FUSEKI_BASE=$(pwd)
```

The current version as of this writing.

Henceforth I'll call the directory into which you unpacked this code as $FUSEKI_BASE. You may find it convenient declare an enviroment variable to that effect.

The $FUSEKI_BASE directory should look like this...

```
$ ll
total 48304
drwxrwxr-x 4 ubuntu ubuntu     4096 Nov 15 18:42 ./
drwxrwxr-x 3 ubuntu ubuntu     4096 Nov 15 18:44 ../
-rw-r--r-- 1 ubuntu ubuntu    30358 Sep 28 17:15 LICENSE
-rw-r--r-- 1 ubuntu ubuntu     9534 Sep 28 17:15 NOTICE
-rw-r--r-- 1 ubuntu ubuntu     2463 Sep 28 17:15 README
drwxrwxr-x 2 ubuntu ubuntu     4096 Nov 15 18:42 bin/
-rwxr-xr-x 1 ubuntu ubuntu    12342 Sep 28 17:15 fuseki*
-rwxr-xr-x 1 ubuntu ubuntu     1241 Sep 28 17:15 fuseki-backup*
-rwxr-xr-x 1 ubuntu ubuntu     2276 Sep 28 17:15 fuseki-server*
-rw-r--r-- 1 ubuntu ubuntu     1264 Sep 28 17:15 fuseki-server.bat
-rw-r--r-- 1 ubuntu ubuntu 26054183 Sep 28 17:31 fuseki-server.jar
-rw-r--r-- 1 ubuntu ubuntu     2217 Sep 28 17:15 fuseki.service
-rw-r--r-- 1 ubuntu ubuntu 23307303 Sep 28 17:30 fuseki.war
drwxr-xr-x 8 ubuntu ubuntu     4096 Sep 28 17:15 webapp/
```

You should now be able to invoke the server...

```
$ ./fuseki-server
```
Going to your public IP and specifying port 3030:

http://ec2-yadda-yadda.amazonaws.com:3030

This should give you a landing page, but there's not much you can do
with it yet because we don't have any datasets, and we haven't configured authentication.

### Configure Authentication in with Shiro

Fuseki's authentication is done with [Apache Shiro](https://en.wikipedia.org/wiki/Apache_Shiro). To configure, we edit `$FUSEKI_BASE/run/shiro.ini`.

The file comes pre-configured with a user called 'admin', whose password is 'pw'

```
[users]
# Implicitly adds "iniRealm =  org.apache.shiro.realm.text.IniRealm"
admin=pw

```
You can add and change users in this section with  `<user>=<password>`. For simplicity, I'll assume you just left it as-is.

Comment out this line:
```
# /$/** = localhostFilter
```

Add this to require basic authentication for your user:

```
/$/** = authcBasic,user[admin]
```

### Configuring Fuseki

We can declare datasets in Fuseki with a configuration file written in Turtle making use of [Jena's assembler RDF vocabulary](http://jena.apache.org/documentation/assembler/assembler-howto.html).

https://jena.apache.org/documentation/fuseki2/fuseki-configuration.html


This github repo has a [little starter config file](https://github.com/ont-app/fuseki-starter/blob/master/fuseki-config.ttl) that should get the ball rolling.

```
$ cd $FUSEKI_HOME
$ mkdir config
$ pushd config
$ wget https://raw.githubusercontent.com/ont-app/fuseki-starter/master/fuseki-config.ttl
$ popd
```

Now we should be able to invoke the fuseki server:

```
$ ./fuseki-server --config config/fuseki-config.ttl 
```

Navigating to `http://ec2-yadda-yadda.amazonaws.com:3030` should now prompt you for a user name and password, and you're off to the races. 

