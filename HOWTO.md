How to run your own Electrum server
===================================

Abstract
--------

This document is an easy to follow guide to installing and running your own
Electrum server on Linux. It is structured as a series of steps you need to
follow, ordered in the most logical way. The next two sections describe some
conventions we use in this document and hardware, software and expertise
requirements.

The most up-to date version of this document is available at:

    https://gitorious.org/electrum/server/blobs/master/HOWTO

Conventions
-----------

In this document, lines starting with a hash sign (#) or a dollar sign ($)
contain commands. Commands starting with a hash should be run as root,
commands starting with a dollar should be run as a normal user (in this
document, we assume that user is called 'bitcoin'). We also assume the
bitcoin user has sudo rights, so we use '$ sudo command' when we need to.

Strings that are surrounded by "lower than" and "greater than" ( < and > )
should be replaced by the user with something appropriate. For example,
<password> should be replaced by a user chosen password. Do not confuse this
notation with shell redirection ('command < file' or 'command > file')!

Lines that lack hash or dollar signs are pastes from config files. They
should be copied verbatim or adapted, without the indentation tab.

Prerequisites
-------------

**Expertise.** You should be familiar with Linux command line and
standard Linux commands. You should have basic understanding of git,
Python packages. You should have knowledge about how to install and
configure software on your Linux distribution. You should be able to
add commands to your distribution's startup scripts. If one of the
commands included in this document is not available or does not
perform the operation described here, you are expected to fix the
issue so you can continue following this howto.

**Software.** A recent Linux distribution with the following software
installed: `python`, `easy_install`, `git`, a SQL server, standard C/C++
build chain. You will need root access in order to install other software or
Python libraries. You will need access to the SQL server to create users and
databases.

**Hardware.** Running a Bitcoin node and Electrum server is
resource-intensive. At the time of this writing, the Bitcoin blockchain is
3.5 GB large. The corresponding SQL database is about 4 time larger, so you
should have a minimum of 14 GB free space. You should expect the total size
to grow with time. CPU speed is also important, mostly for the initial block
chain import, but also if you plan to run a public Electrum server, which
could serve tens of concurrent requests. See step 6 below for some initial
import benchmarks.

Instructions
------------

### Step 0. Create a user for running bitcoind and Electrum server

This step is optional, but for better security and resource separation I
suggest you create a separate user just for running `bitcoind` and Electrum.
We will also use the `~/bin` directory to keep locally installed files
(others might want to use `/usr/local/bin` instead). We will download source
code files to the `~/src` directory.

    # sudo adduser bitcoin
    # su - bitcoin
    $ mkdir ~/bin ~/src
    $ echo $PATH

If you don't see `/home/bitcoin/bin` in the output, you should add this line
to your `.bashrc`, `.profile` or `.bash_profile`, then logout and relogin:

    PATH="$HOME/bin:$PATH"

### Step 1. Download and install Electrum

We will download the latest git snapshot for Electrum and 'install' it in
our ~/bin directory:

    $ mkdir -p ~/src/electrum
    $ cd ~/src/electrum
    $ git://gitorious.org/electrum/server.git
    $ chmod +x ~/src/electrum/server/server.py
    $ ln -s ~/src/electrum/server/server.py ~/bin/electrum

### Step 2. Configure and start bitcoind

In order to allow Electrum to "talk" to `bitcoind`, we need to set up a RPC
username and password for `bitcoind`. We will then start `bitcoind` and
wait for it to complete downloading the blockchain.

    $ mkdir ~/.bitcoin
    $ $EDITOR ~/.bitcoin/bitcoin.conf

Write this in `bitcoin.conf`:

    rpcuser=<rpc-username>
    rpcpassword=<rpc-password>
    daemon=1

Restart `bitcoind`:

    $ bitcoind

Allow some time to pass, so `bitcoind` connects to the network and starts
downloading blocks. You can check its progress by running:

    $ bitcoind getinfo

You should also set up your system to automatically start bitcoind at boot
time, running as the 'bitcoin' user. Check your system documentation to
find out the best way to do this.

### Step 3. Install Electrum dependencies

Electrum server depends on various standard Python libraries. These will be
already installed on your distribution, or can be installed with your
package manager. Electrum also depends on two Python libraries which we wil
l need to install "by hand": `Abe` and `JSONRPClib`.

    $ sudo easy_install jsonrpclib
    $ cd ~/src
    $ git clone git://github.com/jtobey/bitcoin-abe.git
    $ cd bitcoin-abe
    $ sudo python setup.py install

Please note that the path below might be slightly different on your system,
for example python2.6 or 2.8.

    $ sudo chmod +x /usr/local/lib/python2.7/dist-packages/Abe/abe.py
    $ ln -s /usr/local/lib/python2.7/dist-packages/Abe/abe.py ~/bin/abe

### Step 4. Configure the database

Electrum server uses a SQL database to store the blockchain data. In theory,
it supports all databases supported by Abe. At the time of this writing,
MySQL and PostgreSQL are tested and work ok, SQLite was tested and *does not
work* with Electrum server.

For MySQL:

    $ mysql -u root -p
    mysql> create user 'electrum'@'localhost' identified by '<db-password>';
    mysql> create database electrum;
    mysql> grant all on electrum.* to 'electrum'@'localhost';
    mysql> exit

For PostgreSQL:

    TBW!

### Step 5. Configure Abe and import blockchain into the database

When you run Electrum server for the first time, it will automatically
import the blockchain into the database, so it is safe to skip this step.
However, our tests showed that, at the time of this writing, importing the
blockchain via Abe is much faster (about 20-30 times faster) than
allowing Electrum to do it.

    $ cp ~/src/bitcoin-abe/abe.conf ~/abe.conf
    $ $EDITOR ~/abe.conf

For MySQL, you need these lines:

    dbtype MySQLdb
    connect-args = { "db" : "electrum", "user" : "electrum" , "passwd" : "<database-password>" }

For PostgreSQL, you need these lines:

    TBD!

Start Abe:

    $ abe --config ~/abe.conf

Abe will now start to import blocks. You will see a lot of lines like this:

    'block_tx <block-number> <tx-number>'

You should wait until you see this message on the screen:

    Listening on http://localhost:2750

It means the blockchain is imported and you can exit Abe by pressing CTRL-C.
You will not need to run Abe again after this step, Electrum server will
update the blockchain by itself. We only used Abe because it is much faster
for the initial import.

Important notice: This is a *very* long process. Even on fast machines,
expect it to take hours. Here are some benchmarks for importing
~196K blocks (size of the Bitcoin blockchain at the time of this writing):

  * System 1: ~9 hours.
	  * CPU: Intel Core i7 Q740 @ 1.73GHz
	  * HDD: very fast SSD
  * System 2: ~55 hours.
	  * CPU: Intel Xeon X3430 @ 2.40GHz
	  * HDD: 2 x SATA in a RAID1.

### Step 6. Configure Electrum server

Electrum reads a config file (/etc/electrum.conf) when starting up. This
file includes the database setup, bitcoind RPC setup, and a few other
options.

    $ sudo cp ~/src/electrum/server/electrum.conf.sample /etc/electrum.conf
    $ sudo $EDITOR /etc/electrum.conf

Write this in `electrum.conf`:

    # sample config for a public Electrum server	
    [server]
    host = host-fqdn
    native_port = 50000
    stratum_tcp_port:50001
    stratum_http_port:8081
    password = <electrum-server-password>
    banner = Welcome to Electrum server!
    irc = yes
    cache = yes

    # sample config for a private server (does not advertise on IRC)
    [server]
    host = localhost
    native_port = 50000
    stratum_tcp_port:50001
    stratum_http_port:8081
    password = <electrum-server-password>
    banner = Welcome to my private Electrum server!
    irc = no
    cache = yes

    # database setup - MySQL
    [database]
    type = MySQLdb
    database = electrum
    username = electrum
    password = <database-password>

    # database setup - PostgreSQL
    TBD!

    # bitcoind RPC setup
    [bitcoind]
    host = localhost
    port = 8332
    user = <rpc-username>
    password = <rpc-password>

### Step 7. (Finally!) Run Electrum server

The magic moment has come: you can now start your Electrum server:

    $ server

You should see this on the screen:

    starting Electrum server
    cache: yes

If you want to stop Electrum server, open another shell and run:

    $ electrum-server stop

You should also take a look at the 'start' and 'stop' scripts in
`~/src/electrum/server`. You can use them as a starting point to create a
init script for your system.

### 8. Test the Electrum server

We will assume you have a working Electrum client, a wallet and some
transactions history. You should start the client and click on the green
checkmark (last button on the right of the status bar) to open the Server
selection window. If your server is public, you should see it in the list
and you can select it. If you server is private, you need to enter its IP
or hostname and the port. Press Ok, the client will disconnect from the
current server and connect to your new Electrum server. You should see your
addresses and transactions history. You can see the number of blocks and
response time in the Server selection window. You should send/receive some
bitcoins to confirm that everything is working properly.
