# Captive Portal

A basic captive portal stack aiming to do the following. 

  - Present a webpage to the user
  - User submits a form
  - Backend evaluates client
  - User is granted access to whatever treasure the portal is guarding

This is a commonly seen setup in public Wifi networks or hotspots. 

At the heart of this stack is a simple Python Bottle application but this stack requires a lot of other configuration around it. 

## More documentation

I've moved all examples from the [old wiki-page](https://wiki.sydit.se/teknik:guider:networking:captive_portal_med_iptables) to the docs/examples directory.

# Get started

Quick steps will setup a locally hosted dev server which will only run the sample_log plugin and log a line to captiveportal.log.

## Dependencies

* redis as backend for rq
* rq for executing plugins (making firewall rules)

## Install

    python setup.py install

## Run RQ worker in foreground

    rq worker -u redis://127.0.0.1:6379/

## Run dev server

    python portal.py

Now visit the URL shown in the output and you'll be able to submit the portal form, wait for the background job to finish and be redirected to the default website (google.com). 

By default only sample_log plugin is executed which logs a line to captiveportal.log when rq finishes running. 

# Deployment

See examples in docs/examples directory, along with all the cfg examples in the repo.

# Technical details

## Portal server

First and foremost all traffic in a network must be routed through the server running this software stack.

* IPtables
* Captiveportal portal web app
* Associated tools must be setup here

## Portal web app

All this is of course triggered by the portal application written in Python using Bottle.

1. A clients redirected HTTP traffic puts them in the portal webpage.
2. They send a POST form to the /approve url. This can be with user info, personal info, or simply an approve button for a EULA. 
3. The portal executes its plugins in the order that their config section appears in plugins.cfg.
4. Each plugin is passed the request object from Bottle which contains form values among other things.

## IPtables

At the heart is iptables doing the following. 

1. Labeling all traffic with the number 99 in the mangle table.
2. Labeled ICMP, DNS and HTTP traffic is redirected to the portal server in the nat table.
3. All other labeled traffic is dropped.
4. Authenticated clients are jumped out of the mangle table before being labeled, using the RETURN target.
5. Authenticated clients are also deleted from conntrack after having their exception rules created in the mangle table.

## Plugins

Plugins are executed when the user clicks through the captive portal form, whether they submit data or just approve an EULA these plugins are executed. 

Plugins accept data from the request of the user, as of writing this is only wsgi environ data. 

Result of the plugins decide whether the user gets accepted into the portal or not. As such plugins have the potential to do anything from check the users IP against a whitelist to authenticate against a RADIUS server or Active Directory.

Sample plugins prefixed with sample\_ are a good starting point for understanding the plugins. 

Plugins can be made mandatory, or skipped by being disabled, see plugins.cfg for more configuration.

### Why plugins, why job queue?

My primary use case for this portal would be with tens of thousands of users so already I imagined that creating firewall rules would be a blocking action. 

Also with plugins there are options to connect other authentication methods like LDAP or RADIUS, and even other actions to the portal. All of which are possibly blocking. So plugins and job queues felt like a necessary technology to use. Otherwise this type of portal could be a very simple CGI script that runs a system() command.

### Available Plugins

There's only one relevant plugin right now, iptables. But the idea is that you could add RADIUS plugins or other services. The mandatory flag in plugins.cfg decides if a plugin must pass before a client is authenticated. So you can string several plugins together for several actions that must be performed. 

Each plugin responds with JSON.

#### iptables plugin

1. Executes the ``iptables_cmd`` defined in plugins.cfg, with arguments being the client IP and potentially the client MAC address.
2. Ensures the exit code of ``iptables_cmd`` is 0, if not 0 it sets a failed flag in its JSON response.
