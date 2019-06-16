## Synopsis

Using Fail2Ban, we can receive Slack notifications when a jail executes a ban or unban action. When the action is trigger, a notification will be sent to the slack channel of your choice with the corresponding jail name and offending IP.

## Requirements

Slack
Fail2Ban
CURL


## Installation

## **Step 1.&nbsp;**

**[Create a Slack App and enable / configure the Incoming WebHook feature within the app:](https://api.slack.com/incoming-webhooks)**

The first thing you will need is a webhook URL that will allow us to issue commands to the Slack REST API. Using an Incoming Webhook, we can send message to the channel of your choice.

## **Step 2.&nbsp;**

**Create a new ban action for Fail2Ban**

With root, use your favorite editor to create the following file:

**../fail2ban/action.d/slack-notify.conf**

```
#
# Author: Michael Lehmann
#

[Definition]

#(optional) Prevent notification/action for re-banned IPs when Fail2Ban restarts.
norestored = 1

# Option:  actionstart
# Notes.:  command executed once at the start of Fail2Ban.
# Values:  CMD
#
# original  # actionstart = curl -X POST -H 'Content-type: application/json' --data '{"text":"Fail2Ban (<name>) jail has started"}' <slack_webhook_url>
# one-liner # actionstart = curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"[$(hostname)] Fail2Ban (<name>) jail has started\"}" <slack_webhook_url>
actionstart = curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"[<host_name>] Fail2Ban (<name>) jail has started\"}" <slack_webhook_url>

# Option:  actionstop
# Notes.:  command executed once at the end of Fail2Ban
# Values:  CMD
#
# original # actionstop = curl -X POST -H 'Content-type: application/json' --data '{"text":"Fail2Ban (<name>) jail has stopped"}' <slack_webhook_url>
# one-liner # actionstop = curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"[$(hostname)] Fail2Ban (<name>) jail has stopped\"}" <slack_webhook_url>
actionstop = curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"[<host_name>] Fail2Ban (<name>) jail has stopped\"}" <slack_webhook_url>

# Option:  actioncheck
# Notes.:  command executed once before each actionban command
# Values:  CMD
#
actioncheck =

# Option:  actionban
# Notes.:  command executed when banning an IP. Take care that the
#          command is executed with Fail2Ban user rights.
# Tags:    <ip>  IP address
#          <failures>  number of failures
#          <time>  unix timestamp of the ban time
# Values:  CMD
#
# original  # actionban = curl -X POST -H 'Content-type: application/json' --data '{"text":"Fail2Ban (<name>) banned IP *<ip>* for <failures> failure(s)"}' <slack_webhook_url>
# one-liner # actionban = curl ipinfo.io/<ip>/country | (read COUNTRY; curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"[$(hostname)] Fail2Ban (<name>) banned IP *<ip>* :flag-$COUNTRY: ($COUNTRY) \"}" <slack_webhook_url>)
actionban = curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"[<host_name>] Fail2Ban (<name>) banned IP *<ip>* :flag-<country_name>: (<country_name>) \"}" <slack_webhook_url>

# Option:  actionunban
# Notes.:  command executed when unbanning an IP. Take care that the
#          command is executed with Fail2Ban user rights.
# Tags:    <ip>  IP address
#          <failures>  number of failures
#          <time>  unix timestamp of the ban time
# Values:  CMD
#
# original # actionunban = curl -X POST -H 'Content-type: application/json' --data '{"text":"Fail2Ban (<name>) unbanned IP *<ip>*"}' <slack_webhook_url>
# one-liner # actionunban = curl ipinfo.io/<ip>/country | (read COUNTRY; curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"[$(hostname)] Fail2Ban (<name>) unbanned IP *<ip>* :flag-$COUNTRY: ($COUNTRY) \"}" <slack_webhook_url>)
actionunban = curl -X POST -H 'Content-type: application/json' --data "{\"text\":\"[<host_name>] Fail2Ban (<name>) unbanned IP *<ip>* :flag-<country_name>: (<country_name>) \"}" <slack_webhook_url>

[Init]

init = 'Sending notification to Slack'

slack_webhook_url = <Add_Your_Full_Slack_Webhook_URL_Here>

#The following are variables that will be evaluated at runtime in bash, thus cannot be used inside of single-quotes
host_name = $(hostname)

# lets find out from what country we have our hacker
country_name = $(curl ipinfo.io/<ip>/country)
```


Replace&nbsp;**<Add_Your_Full_Slack_Webhook_URL_Here>**&nbsp;with the Slack App webhook URL you created in **Step 1**.

```Norestored = 1``` is an optional setting you can enable or disable to prevent or allow notifications/actions for re-banned IPs when the Fail2Ban service restarts.

Also note that I added lines *original* and *one-liner* (commented out) to each action section. These are just alernatives and should work just as well, in their own way.

* *original* is a plain basic curl request without adding hostname, country code, or country emoji flag.
* *one-liner* performs everything in one-line (via piping) and has no need for the host_name nor country_name variables at the bottom of the config file.

Save the file. Now it’s time to add this action to one of our jails.

## **Step 3.&nbsp;**

**Apply the action to your jail(s)**

For this demonstration we are going to be using the SSH jail. If you haven’t already, create a jail.local file for Fail2Ban in case a package update overwrite the default configuration:

`sudo cp ../fail2ban/jail.conf ../fail2ban/jail.local`

Now let’s open&nbsp;**../fail2ban/jail.local** and add the Slack notification action.


Apply the action to your jail(s) or all jails in (defaults) section

**NOTE**: Additional actions just get added on a *newline*.
If you only want slack-notifications for specific jails, then only define the action within those jails, instead of at the top level. 

To apply to all jails, you might have something like the following (if you use email notifcations too):

```
[DEFAULT]
...

action = %(action_mwl)s
         slack-notify
...
```

To apply to only a specific jail (e.g. only ssh): 

```
[ssh]

enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 6
banaction = iptables-multiport
            **slack-notify**
```


The&nbsp;“ssh” configuration block will most likely use the default banaction, which means the property won’t be listed. Add the banaction line, using&nbsp;“slack-notify” as the second command. Save and close the file.

**Now restart the Fail2Ban service (or entire letsencrypt docker container) and you should see your jails starting up:**

_Fail2Ban (ssh) jail has started_

## License

Use it and abuse it, just don't lose it.

## Troubleshooting

- If nothing happens, make sure that Fail2Ban actually started back up successfully. Make sure it actually starts. Are there any errors when it starts? If there are errors, fix them :)
```
fail2ban-client start
```

- A good general troubleshooting step is to try manually running the curl requests in your shell outside of fail2ban. Do they work at all?
