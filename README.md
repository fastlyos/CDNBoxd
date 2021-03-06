# CDNBoxd
A DNS distributed Global Load Balancing with geolocation and bandwith shaping.

CDNBox is a solution to mix private CDN with legacy CDN. CDNBoxd is originaly the specific software used to clusterize multiple CDNBoxes. Nowdays, CDNBoxd is used out of CDNBox solution, to distribute load on Edge endpoints for example.
A CDNBox is a node of CDNBoxd.

## Features

* DNS load balancer
* Distribute DNS/HTTP trafic based on geolocaion and performance.
* Based on resolver geolocation or EDNS client geolocation if available.
* Limit bandwith by node with a global saturation indicator.
* Alerting based on error rate.
* Logging to elasticsearch thru rsyslog.
* API for monitoring and configuration changes.
* UI (companion project), https://github.com/Doloros/CDNBOX-d-Admin
* Client side DNS resolution and connection time measurment based on Resource Timing.
* varnishstat metric collector.
* CDNBox's specific (local) service testing (HTTP).

## How to install

* Require node (8+ LTS) and npm. Production tested releases Node v8.9.4, v8.11.1, v8.11.4, v10.14.1
* git clone https://github.com/Francois-v3/CDNBoxd.git
* npm install
* download GeoLite2 Country from https://dev.maxmind.com/geoip/geoip2/geolite2/ and copy GeoLite2-Country.mmdb into CDNBoxd directory
* copy config-template.json to config.json and change it to your settings.
* copy cdnboxd-template.service to cdnboxd.service and change \<home\> to your directory.
* install cdnboxd.service into systemd
* start CDNBoxd with "systemctl start cdnboxd.service"

Repeat for each node.

## Deployment and upgrade tips

When you have different nodes other different operators to run CDNBoxd, the major risk is bug. To manage this risk, we advice you to adopt a canari strategy. Choose a referral node that you upgrade first, then wait for at least one day and have a look to errors. If everything looks ok, upgrade an other set of nodes, wait a least one day. Repeat until all nodes are updated.

We advice you to include added files in your deployment script or tool. For example, clone the repo files in another repo where you add node binary, node modules, GEO database and configuration file, then deploy with remote command  "git pull; systemctl restart cdnboxd.service". Automate your deployment, but keep a human to launch deployment of each set of nodes. 

## Configuation file

All fields with default value are optionnal. Config is exactly the same on each node.

```
  "dnsserver": { 
     "domain": domain answered, any request to other domain will not be answered,
     "port": DNS port, default to 53,
     "nsnum": number of NS record,
     "nsttl": ttl for NS and SOA records (second),
     "attl": ttl for A and AAAA records (second), 
     "dnscountdownratio" : global request rate reduction ratio (default to 1.2).
     "dnscountavgmin": minimum global request rate by node (default to 1.5).
     "globalthrottlelimit": global request rate growup ratio to throttle (default to 2), only or cdnboxes
                            with targetbw.
     "globalthrottlebwratio": targetbw ratio to activate global request throttle (default to 0.4).
  },
  "httpserver": {
     "port": HTTP port,
     "authorization": HTTP credential (Basic xxxxxx)
  },
  "states": { 
     "penalgdown": reduction of global saturation indicator period (default to 120)
  },
  "perf": { // optional
    "domain": domain to mesure,
    "beaconurl": beacon URL prefix,
    "delay": time to start measurment,
    "weights": measurment weight by country (percentage), ex: { "FR": 50, "default": 100 },
  },

  "varnishmetricsoutput": optional, defaults to "log". If "log" a log record is isuued each 10s. If "status" a
               log record is issued for each call to /cdn/status and results are included in status response.
  "varnishmetrics":   // optional
    "es_name": "varnish_name", define a varnish numeric metric. varnish_name is the varnishstat entry name. es_name
               is the json name sent to ES. If varnish_name is not present, -1 value will be set.
    ...
  },
  "tlsticketkeys": { // if present, compute TLS ticket keys file for Haproxy (bind's option tls-ticket-keys).
    "filename": file's name. Current directory id cdnboxd one.
    "ttl": duration of keys in ms, defaults to 86400000 (1 day),
    "refresh": refresh period in ms. defaults to 3600000 (1 hour),
  },
  "tcprtt": { // if present, collect TCP RTT metrics.
    "period": collect period in ms. defaults to 5000 (5 seconds),
  },
  "clustersecret": secretkey of cluster, use for communication between nodes, default to httpserver.authorization.
  "cdnboxes": [ // order matters to who am i and tlsticket processes.
    { "name": node's name,
      "hostname": node's hostname without final dot,
      "countries": node's country localisation, "ALL" or array of countries like [ "US", "CA" ],
      "continents": node's continent localisation, array like [ "EU", "NA" ]. For NS records order, we use
                    array of countries, then continent, then country "ALL". For A and AAAA records, we use
                    continents only if no conntries are defined.
      "proto": "http:" ou "https:", default protocol for this CDNBox (default to "https:"),
      "isns": is this CDNBox is a DNS server (default to false),
      "cname": serve CNAME record instead of A or AAAA, (default to false),
      "penal": default value of global bandwith saturation indicator, default to 100. Will be increased
               if BW saturated.
      "ishttp": is this CDNBox serve HTTP (other than CDNBoxd trafic itself),
      "status": initial HTTP trafic status (on/off),
      "targetbw": target bandwith for this node,
      "floorbw": floor bandwith targetbw ratio to start reducing bandwith (default to 0.9).
      "ceilbw": ceil bandwith targetbw ratio to stop trafic, (default to 1.1).
      "perfURL": performance measurment URL (default to <proto>://<hostname>/cdn/image.gif),
      "addscore": add a fixed value to performance (millisecond, default to 0),
      "fixscore": set a fixed value to performance (desactivate performance measurment),
      "dnsthrottlebwratio": define high/low targetbw ratio for targeted bandwith DNS throttle.
      "dnsthrottlelowratio": enable and define ratio of low targetbw ratio for targeted bandwith DNS throttle.
      "dnsthrottlehighratio": enable and define ratio of high targetbw ratio for targeted bandwith DNS throttle.
      "nspriorityratio": used to change distribution of a node un NS list in case of natural node overload.
                         Default to 1. 0 means maximum.
      "nsgroup": if defined, enforce that nslist contains always different nsgroup. Usually, nsgroup
                 contains operator's name.
      "varnishmetrics": if true, we collect varnishstate metric defined in "varnishmetrics". Defaults to false.
      "localtests": define local http test (to be used in appli section). [
        { "name": name of the test. Warning, only fisrt letter is shown in console. 
          "url": URL to test, must return a 200 status.
          "timeout": timeout in ms. Optional, defaults to 3500.
        },
        ... 
      ],
      "notification": { if defined, enable error notification ont this node. Defining it on all nodes is
                        a bad idea !
        "email": notification are sent to this emails, which can be comma separated list.
        "threshold": threshold of error number on period to notify.
        "period": theshold period.
        "lastmessagenumber": number of error to show in the message body.
        "remindertime": period of reminder.
      ,}
      "dontbindall4": by default we don't bind on port 53 (DNS) on all local ipv4 adresses. Setting this
                      to false to bind on all local ipv4 addresses. Useful to set a avoid conflits with
                      bind9 listening on 127.0.0.1:53 or systemd resolver on 127.0.0.53.
    },
    { ...
    }
  ],

  "applis": [
    "a1": [
      {
        // first node will be serve if none of the other match.
        "cdnbox": "ABC-victim2" node's name,
        "addscore": if defined, overload addscore for this node in this application,
        "fixscore": if defined, overload fixscore for this node in this application,
        "countries": if defined, overload cdnbox's countries "ALL", or [ "US", "CA" ],
        "continents": if defined, overload cdnbox's continents, like [ "EU", "NA" ]. 
        "localtests": if defined, exclude this cdnbox if one the test is false. [ "apache", ... ]
      },
      { ...
      }
    ],
    "a2": [
      { ...
      },
      { ...
      }
    ]
  ]
```

## API

### global (no authentcation)
*  /cdn/image.gif: get a pixel image. Use by default to measure response time.
*  /cdn/perf.js: get the script to measure cdnboxd performance (Resource Timing).
*  /cdn/beacon: beacon to upload perf.js measurements.
*  /cdn/penalite: return a saturation indicator (100 means no saturation, growing if saturated).
*  /cdn/whoami: return an 204 (no content), whith external IP in X-IP header (used to determine which node we are).

### global (with authentcation)
*  /cdn/status: reports metrics to be use by monitoring. For example, Metricbeat with http module.
*  /cdn/cdnboxStates: dumps memory of cdnbox in JSON format (debuging).
*  /cdn/api/states: returns a JSON with metrics and state of cdnbox.
*  /cdn/cdnboxes: returns a HTML with metrics and state of cdnbox.
*  /cdn/console: show a summary of all cdnboxes.

### Config API (authentication required).
*  /cdn/config GET: get the config.json content of cdnbox.
*  /cdn/config PUT: set the config.json on cdnbox and reload the config.
*  /cdn/config/copyfrom/\<cdnboxname\>: copy and reload the config.json from \<cdnboxname\> on cdnbox.

## Example

Serve www.mydomain.org on a hybrid CDN with one legacy CDN (legacycdn.mydomain.org) and two Vanish boxes.

Config file (config-template.json) is:
```javascrit
{
  "dnsserver": { "domain": "cdnboxd.mydomain.org", "port": 53, "nsnum": 2, "nsttl": 3600, "attl": 20 },
  "httpserver": { "port": 1080, "authorization": "Basic XXXXXXXXXX==" },
  "states": { "penalgdown": 60 },
  "perf": {
    "domain": "www.mydomain.org", "beaconurl": "http://test.mydomain.org",
    "delay": 5000, "weights": { "FR": 50, "default": 100 }
  },
  "varnishmetrics": {
    "sess_conn": "MAIN.sess_conn",
    "client_req": "MAIN.client_req",
    "g_bytes": "SMA.s0.g_bytes",
    "g_space": "SMA.s0.g_space"
  },
  "cdnboxes": [
    {
      "name": "Node1", "hostname": "node1.mydomain.org",
      "targetbw": 500, "countries": [ "FR", "BE" ], "isns": true
    },
    {
      "name": "Node2", "hostname": "node2.mydomain.org",
      "targetbw": 500, "countries": [ "FR", "BE" ], "isns": true
    },
    {
      "name": "Node3", "hostname": "legacycdn.mydomain.org",
      "fixscore": 1000, "countries": "ALL", "isns": false, "cname": true
    }
  ],
  "applis": {
    "a1": [
      { "cdnbox": "Node3" }, { "cdnbox": "Node1" }, { "cdnbox": "Node2" }
    ],
    "test": [
      { "cdnbox": "Node1", "fixscore": 1000, "countries": "ALL" },
      { "cdnbox": "Node1" },
      { "cdnbox": "Node2" }
    ]
  }
}
```

DNS records:

* cdnboxd.mydomain.org IN NS node1.mydomain.org.
* cdnboxd.mydomain.org IN NS node2.mydomain.org.
* www.mydomain.org IN CNAME a1.cdnboxd.mydomain.org.
* test.mydomain.org IN CNAME test.cdnboxd.mydomain.org.


