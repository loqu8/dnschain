# How-to setup a DNSChain Server on Ubuntu

Here is a quick *how-to* for setting up a <a href="https://github.com/okTurtles/dnschain">DNSChain</a> server running on [Ubuntu 14.04 LTS](https://www.ubuntu.org). This will run <nobr>PowerDNS</nobr> recursor software, which will simply pass queries for next generation domain names to our DNSChain server. 

These blockchain-based domain names are resolved by simply querying the local blockchain; bypassing the conventional DNS altogether. This approach will allow you to resolve these new-fangled domain names as well, thanks to DNSChain. This same approach can be applied using any nameserver software, but to we'll be using PowerDNS in our example just to demonstrate the idea.

So our recursor software will issue DNS queries for `.com` and `.net` domains as you would expect, but will consult the local Namecoin blockchain in order to resolve `.bit` domains. So results for, say, `okturtles.bit` will be returned without engaging other servers on the Internet.

We start with a fresh install of Ubuntu 14.04 LTS.

## Getting Started

First, update apt-get and install npm (this will also install nodejs)

	$ sudo apt-get update
	$ sudo apt-get install npm

## Namecoin install

In our example, we'll use Namecoin, DNSChain and PowerDNS. It's probably a good idea to first install the Namecoin daemon, since it requires some time to download the blockchain. You can find the latest packages for various linux distros at the [Namecoin site](https://software.opensuse.org/download.html?project=home%3Ap_conrad%3Acoins&package=namecoin).

	# identify source for Namecoin packages
	$ sudo sh -c "echo 'deb http://download.opensuse.org/repositories/home:/p_conrad:/coins/xUbuntu_14.04/ /' >> /etc/apt/sources.list.d/namecoin.list"
	$ wget http://download.opensuse.org/repositories/home:p_conrad:coins/xUbuntu_14.04/Release.key
sudo apt-key add - < Release.key
	$ apt-get update
	$ apt-get install namecoin

To get started with namecoin, follow the [Quick start](https://wiki.namecoin.info/index.php?title=Install_and_Configure_Namecoin)

	$ mkdir -p ~/.namecoin \
&& echo "rpcuser=`whoami`" >> ~/.namecoin/namecoin.conf \
&& echo "rpcpassword=`openssl rand -hex 30/`" >> ~/.namecoin/namecoin.conf \
&& echo "rpcport=8336" >> ~/.namecoin/namecoin.conf

Our config file will specify a valid RPC username and password, as well as optionally specifying a port to bind to (default is 8336). There are other options, see the [Namecoin wiki](https://wiki.namecoin.info/index.php?title=Install_and_Configure_Namecoin) for more details. To run on server, we will add daemon=1

	$ echo "daemon=1" >> ~/.namecoin/namecoin.conf

Go ahead and run `namecoind` to get things going. It is going to take a while to do what it needs to do.	

For Ubuntu, instead of systemd, we use Upstart -  write this file /etc/init/namecoind.conf 

	description "namecoind"
	
	start on filesystem
	stop on runlevel [!2345]
	oom never
	expect daemon
	respawn
	respawn limit 10 60 # 10 times in 60 seconds
	
	script
	user=ubuntu
	home=/home/$user
	cmd=/usr/bin/namecoind
	pidfile=$home/.namecoin/namecoind.pid
	# Don't change anythinsudo initctl reload-configurationg below here unless you know what you're doing
	[[ -e $pidfile && ! -d "/proc/$(cat $pidfile)" ]] && rm $pidfile
	[[ -e $pidfile && "$(cat /proc/$(cat $pidfile)/cmdline)" != $cmd* ]] && rm $pidfile
	exec start-stop-daemon --start -c $user --chdir $home --pidfile $pidfile --startas $cmd -b --nicelevel 10 -m
	end script
	
Then `namecoind stop` to stop the process and restart it with `sudo initctl reload-configuration`
	
	TODO: This doesn't actually work to run or automatically start namecoind
	
As mentioned, `namecoind` is going to begin downloading the blockchain soon after startup. We won't be able to lookup domain names from the blockchain until it has made some progress, so let's revisit testing our namecoin install later.

Meanwhile, we can setup PowerDNS and DNSChain, and then come back and test this, as follows:

	$ namecoind getinfo
	$ namecoind name_show d/okturtles

	
	--------------------
	
OK, so basic operations work directly from the command line, now let's check it via the RPC interface.

	$ curl --user dnsuser:dnsuser --data-binary '{"jsonrpc":"1.0","id":"curltext","method":"getinfo","params":[]}'  -H 'content-type: text/plain;' http://127.0.0.1:8336
	$ curl -v -D - --user dnsuser:dnsuser --data-binary '{"jsonrpc":"1.0","id":"curltext","method":"name_show","params":["d/okturtles"]}' -H 'content-type: text/plain;' http://127.0.0.1:8336
   
## PowerDNS install

We need [PowerDNS](https://www.powerdns.com/) version 3.6.x or higher. This is currently newer than the version in _stable_ we'll use _wheezy-backports_. Append the following onto __/etc/apt/sources.list__:
 
	deb http://http.debian.net/debian wheezy-backports main

Download and install from the repo, and check to see that it installed, and that it runs.

	apt-get update
	apt-get -t wheezy-backports install pdns-recursor
	rec_control ping   # check if server is alive

Next, we need to tell PowerDNS to send requests for `.bit` domain names to port 5333, where we will soon tell DNSChain to listen. This configuration is specified in __/etc/powerdns/recursor.conf__

	forward-zones=bit.=127.0.0.1:5333,dns.=127.0.0.1:5333,eth.=127.0.0.1:5333,p2p.=127.0.0.1:5333
	export-etc-hosts=off
	allow-from=0.0.0.0/0
	local-address=0.0.0.0
	local-port=53

Notice in particular our *forward-zones* declaration. Even though in our example, we're simply setting up our server to resolve Namecoin's `.bit` domain names, support for `.eth` and `.p2p` domains is on the current roadmap. 

Since we have not yet setup DNSChain, let's just make sure our PowerDNS recursor can correctly resolve conventional domain names before we move on.

	dig @127.0.0.1 okturtles.com

You should get a result similar to this, with an IP address found for okturtles.com.

![](http://i.imgur.com/iL881lF.png)
   

## DNSChain install

then install coffee-script:
	
	$ sudo npm install -g coffee-script
	

DNSChain is written using NodeJS and we need to install this and a few other javascript tools:
  
	apt-get install libc6-dev zlib1g-dev libssl-dev nodejs-dev  
	update-alternatives --install /usr/bin/node nodejs /usr/bin/nodejs 100
	node -v
	curl https://www.npmjs.org/install.sh | sudo sh
	npm -v
	npm install -g coffee-script
	npm install -g grunt-cli

Now we're ready to install DNSChain, and once again, we'll create a user to run DNSChain:

	npm install -g dnschain
	adduser dnschain

We will tell DNSChain to bind to port 5333, but you can use any high port number as long as it matches the port number that PowerDNS is handing off requests to. This was specified earlier in __/etc/powerdns/recursor.conf__. 

Another great feature of DNSChain is that we can expose the lookup results via HTTP. We'll specify port 8000 for this, but you can use any high number port that's open. DNSChain can be setup to be accesed by webserver, via port 8000 for example. Here's an example DNSChain configuration file __/home/dnschain/.dnschain.conf__
  
	[log]
	level=info
	pretty=true
	cli=true

	[dns]
	port = 5333
	oldDNS.address = 8.8.8.8
	oldDNS.port = 53

	[http]
	port=8000
	tlsPort=4443


This process will be run by our *dnschain* user, so it needs to be readable.

	chown dnschain.dnschain /home/dnschain/.dnschain.conf

As with the others, we're going to run this as a `systemd` service. Here's [our example unit file](../scripts/dnschain.service), feel free to adjust as needed. 

Note that this unit file also sets up port forwarding so our DNSChain install can run unprivileged using port 5333, while still receiving traffic from port 53. Here is a [more detailed discussion](https://stackoverflow.com/questions/413807/is-there-a-way-for-non-root-processes-to-bind-to-privileged-ports-1024-on-l/21653102#21653102) about this problem of running user processes that listen on ports < 1024.

Let's start DNSChain to ensure that we have it configured correctly.

	$ systemctl enable dnschain
	$ systemctl start dnschain

Finally, let's test it by trying to resolve a `.bit` domain name.

	$ dig @127.0.0.1 okturtles.bit
	$ curl http://127.0.0.1:8000/d/okturtles

The first `dig` command ought to return the IP address for `okturtles.bit` and the second should return all the information associated with this domain name, including IP address, TLS fingerprint and more. If so, congratulations, everything works just fine! 