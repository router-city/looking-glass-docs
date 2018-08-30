# looking-glass-docs
Information on setting up looking glass software

We want to set up Looking Glass software for our BGP network. We will use [bird-lg](https://github.com/sileht/bird-lg) but may explore other looking glass software later like [looking glass](https://github.com/respawner/looking-glass).

A test installation of `bird-lg` for router.city is available here, https://lg.router.city

## Installation

We will assume you have a Debian installation with a non-root, sudo user and `apache2` installed. First, we need to install some Python libraries. It appears that a few of these are not in Python 3, but we have Python 2.7.9 installed which happens to work.

```
$ python --version
Python 2.7.9
```

### Install bird-lg and Prerequisites

First, install `pip` and `graphviz`:

```
$ sudo apt-get install python-pip
$ sudo apt-get install graphviz
```

Now install Python packages through `pip`:

```
$ sudo pip install dnspython
$ sudo pip install flask
$ sudo pip install pydot
$ sudo pip install python-memcached
```

Next, make a web directory and clone `bird-lg`:

```
$ sudo mkdir -p /var/www/lg.router.city/private
$ cd /var/www/lg.router.city/private
$ sudo git clone https://github.com/sileht/bird-lg.git
```

### Client Configuration

Now we can edit the default client config, since the machine hosting our looking glass is also running `bird`:

```
$ cd bird-lg
$ sudo nano lgproxy.cfg
```

Edit the values to match the IPs for your interface to router.city:

```
DEBUG=False
LOG_FILE="/var/log/lg-proxy/lg-proxy.log"
LOG_LEVEL="WARNING"
BIND_IP = "0.0.0.0"
BIND_PORT = 5002
ACCESS_LIST = ["172.24.0.1"]
IPV4_SOURCE="172.24.0.1"
IPV6_SOURCE="2001:db8:dead:beef:cafe:f00d:0:1"
BIRD_SOCKET="/var/run/bird/bird.ctl"
BIRD6_SOCKET="/var/run/bird/bird6.ctl"
```

Now we will set up some log files and change permissions:

```
$ sudo mkdir /var/log/lg-proxy
$ sudo touch /var/log/lg-proxy/lg-proxy.log
$ sudo chown -R www-data:www-data /var/log/lg-proxy
```

### Running the bird-lg client

To run the client, we can launch it in the background:

```
$ cd /var/www/lg.router.city/private/bird-lg
$ sudo nohup python lgproxy.py &
```

#### Server Configuration

Now edit the server config:

```
$ sudo nano lg.cfg
```

Now fill in the appropriate info from the client you just set up. You can specify multiple clients in here:

```
DEBUG = True
LOG_FILE="/var/log/lg.log"
LOG_LEVEL="WARNING"

DOMAIN = "router.city"

BIND_IP = "0.0.0.0"
BIND_PORT = 5001

PROXY = {
                "faminet": 5002
        }

# Used for bgpmap
ROUTER_IP = {
        "faminet" : [ "172.24.0.1", "2001:db8:dead:beef:cafe:f00d:0:1" ]
}

AS_NUMBER = {
    "faminet" : "64496"
}

#WHOIS_SERVER = "whois.foo.bar"

# DNS zone to query for ASN -> name mapping
ASN_ZONE = "asn.cymru.com"

SESSION_KEY = '\xd77\xf9\xfa\xc2\xb5\xcd\x85)`+H\x9d\xeeW\\%\xbe/\xbaT\x89\xe8\xa7'
```

Now we will set up some log files and change permissions:

```
$ sudo touch /var/log/lg.log
$ sudo chown www-data:www-data /var/log/lg.log
$ sudo chown -R www-data:www-data /var/www/lg.router.city
```

### Apache Configuration

Now we need to create some `apache2` config:

```
$ sudo nano /etc/apache2/sites-available/lg.router.city.conf
```

Paste the following:

```
<VirtualHost *:80>
    ServerName lg.router.city
    ServerAdmin admin@router.city
    WSGIScriptAlias / /var/www/lg.router.city/private/bird-lg/lg.wsgi
    <Directory /var/www/lg.router.city/private/bird-lg/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/lg.router.city/private/bird-lg/
    <Directory /var/www/lg.router.city/private/bird-lg/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Now enable the new site and restart `apache2`:

```
$ sudo a2ensite lg.router.city.conf
$ sudo systemctl restart apache2
```

The looking glass server will now run automatically!

#### Add SSL with Let's Encrypt

First, install `certbot`

```
$ sudo apt-get install python-certbot-apache 
```

Now create the certificate:

```
$ sudo certbot certonly --standalone -dlg.router.city
```