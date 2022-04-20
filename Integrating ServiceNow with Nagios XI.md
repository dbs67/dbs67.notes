# Integrating ServiceNow with Nagios XI

+++
draft = true
date = 2022-04-19T11:22:30-06:00
title = "Integrating ServiceNow with Nagios XI"
slug = ""
tags = []
categories = []
thumbnail = "images/tn.png"
description = ""
+++

* Worked out the ReST request calls with Axios and NodeJS.

```js

import axios from "axios";

const hostsURL =
  "http://<domain>/nagiosxi/api/v1/objects/host?apikey=<key>&pretty=1";

const servicesURL =
  "http://<domain>/nagiosxi/api/v1/objects/service?apikey=<key>&pretty=1";

var ciMonitorNames = [];

const reqHosts = axios.get(hostsURL);
const reqServices = axios.get(servicesURL);

axios
  .all([reqHosts, reqServices])
  .then(
    axios.spread((...responses) => {
      const resHosts = responses[0];
      const resServices = responses[1];
      const hosts = resHosts.data.host;
      const services = resServices.data.service;

      hosts.forEach((host) => {
        services.forEach((service) => {
          if (service.host_object_id == host.host_object_id) {
            const ciMonitorName = {
              alias: host.alias,
              address: host.address,
              hostId: host.host_object_id,
              hostName: host.host_name,
              serviceDescription:
                "Host: " + host.host_name + " " + service.service_description,
            };
            ciMonitorNames.push(ciMonitorName);
          }
        });
      });
    })
  )
  .catch((err) => {
    console.error(err);
  })
  .finally(() => {
    ciMonitorNames.sort((a, b) => (a.alias > b.alias) ? 1 : (a.alias === b.alias) ? ((a.hostId > b.hostId) ? 1 : -1) : -1 )
    console.info(JSON.stringify(Object.assign({}, ciMonitorNames)))
  });
ciMonitorNames.forEach((ciMonitorName) => {gs.addInfoMessage(ciMonitorName.toString())});

```

* Created ServiceNow script include to handle the ReST requests and data sanitizing.

  * Discovered that the JS `for` loop is the fast way to iterate over an array.
  * Created a function that takes 3 arguments, array, attribute name, and value.  This seems to scan the hosts array the fastest.  In xPlore it only takes around 3-5 minutes.
