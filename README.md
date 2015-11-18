# Udacity Full-Stack Nanodegree Project 5 - Linux Server

Author: Dave Voutila

# Access to the System
The only remote access is allowed via key-based SSH connections using the keyset
provided in the "notes to the reviewer". Installing the keys locally will allow
remote access under the 'grader' account, of which the local password is
communicated in the notes as well:

``` bash
ssh -i ~/.ssh/udacity-grader grader@52.34.69.185 -p 2200
```

The web interface is available at:
(ec2-52-34-69-185.us-west-2.compute.amazonaws.com)[http://ec2-52-34-69-185.us-west-2.compute.amazonaws.com]


# Server Setup

The initial task was to start bootstrapping the server with the packages needed
to run our Python web app in Apache2's mod_wsgi.

``` bash
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi
sudo apt-get install git
```

Getting ready for our Flask app:
``` bash
sudo apt-get install postgresql python-psycopg2
sudo apt-get install python-flask python-sqlalchemy
sudo apt-get install python-pip
sudo pip install bleach
sudo pip install oauth2client
sudo pip install requests
sudo pip install httplib2
```

## Database Setup

I've created a user called _catalog_ that does not have sudo rights and using
the following PostgreSQL config pages, have locked down access to only local
users that match users on the host:
* [http://www.postgresql.org/docs/9.1/static/auth-methods.html#AUTH-TRUST]
* [http://www.postgresql.org/docs/9.1/static/auth-username-maps.html]

Since my DB is bootstrapped by a python script, I used the `sudo su` trick to
switch to the _catalog_ user since I purposely locked the account using
`sudo passwd -l catalog` so nobody can log into it.

# Deploying the App
After git was installed, I whipped up a shell script (`deploy.sh` in the _grader_
home directory) that uses `git archive` to properly "export" the local git
master branch to a temporary location before moving files to _/var/www/catalog_.

Right now it's pretty crude and will wipe out things like the _uploads_ folder,
but could be improved to properly preserve that.

## Apache2 Config

Getting mod_wsgi to launch the Catalog app was surprisingly easy in some ways
and surprisingly confusing in others (mostly to do with learning that mod_wsgi
swallows most error and exception logging...hency having to wrap the app with
logging middleware per [Debugging Techniques]
(https://code.google.com/p/modwsgi/wiki/DebuggingTechniques)).

Ultimately, Apache2 is pointing to /var/www/catalog for the app and running as
the _catalog_ user (which owns that directory). See the config in
`/etc/apache2/sites-enabled/000-default.conf`.


# Resolving Web App issues for Production
Moving the Catalog app to run in mod_wsgi exposed a lot of configuration issues
with my app that went unnoticed earlier as they just assumed the app would
always run in Flask's built-in http server instance.

First, I needed to properly create a wsgi script that properly configured the
application and setup things like the secret key, etc. as seen in
[catalog.wsgi](https://github.com/voutilad/udacity-project3/blob/master/vagrant/catalog/catalog.wsgi)

(Obviously, we shouldn't be putting secret key's and client_id's like that in
github, but for the sake of making the app easier to review I've done so.)

Then, in other modules using local paths (like for file uploads) or the oauth2
code looking for the json config file, I found a tip from the [mod_wsgi homepage]
(https://code.google.com/p/modwsgi/wiki/TipsAndTricks) that showed how to use
a try/except block to detect if running in mod_wsgi:

``` python
try:
    from mod_wsgi import version
    # Put code here which should only run when mod_wsgi is being used.
except:
    pass
```

# Little Things

## Auto update of Packages
Un-attended Upgrades has been installed per [https://help.ubuntu.com/lts/serverguide/automatic-updates.html]
and allows for automatic security updates to be applied and for package cleanup
to occur.

## NTP updates
Keeping the clock in sync is important for multiple reasons, so a script was
put in /etc/cron.daily to call ntpdate daily to set the clock. Thanks to
the doc here: [https://help.ubuntu.com/community/UbuntuTime#Command_Line_ntpdate]

# Monitoring with [Monitorix](http://www.monitorix.org/)
Looking for something lightweight, I found Monitorix after some googling. It
requires adding an additional Apt repo, so I installed the repo and it's
public key described at [IzzySoft](http://apt.izzysoft.de/ubuntu/dists/generic/)
by adding the following source to _/etc/apt/sources.list_
```
deb http://apt.izzysoft.de/ubuntu generic universe
```
And installing the key with:
``` bash
sudo apt-key add izzysoft.asc
```
After a quick _sudo apt-get update_ it was as easy as doing an install of the
_monitorix_ package.

Before it could monitor Apache2, I needed configure the mod_status module as
described [here](http://www.rackspace.com/knowledge_center/article/enabling-and-using-apaches-modstatus-on-debian-and-ubuntu)
and modified my apache2 site config adding the following within the VirtualHost
block:

```
# For mod_status
<Location /server-status>
  SetHandler server-status
  Order Deny,Allow
  Deny from all
  Allow from localhost
</Location>
```
This enabled the mod_status interface on http://localhost/server_status, but
denied external access so only Monitorix or a local user can query it.

Then, I configured Monitorix to use its embedded http server, but by not exposing
its port (8888) in the UFW firewall, it's inaccessible to external users. To
see the web page, you need to do some SSH tunneling like so (assuming you're
on a \*nux/mac system):

``` bash
ssh -i ~/.ssh/udacity-grader -f grader@52.34.69.185 -L 9999:localhost:8888 -N -p 2200
```

This will bind your local port 9999 to the port 8888 on the host system and
securely tunnel the traffic via SSH. Now the Monitorix interface should be
available at [http://localhost:9999/monitorix] on your local machine.



# Some References
Some other helpful links that helped with Project 5:

* [Flask's mod_wsgi instructions](http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/)
* [How to Monitor with Monitorix](http://freedif.org/how-to-monitor-your-server-with-monitorix/)
