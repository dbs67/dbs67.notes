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

  * Discovered that the JS `for` loop is the fast way to iterate over an array.  Got the time down to 3-5 minutes.
  * Created a function that takes 3 arguments, **array**, **attribute name**, and **value**.  This seems to scan the hosts array the fastest.  In xPlore it only takes around 3-5 minutes to process both the **hosts** and **service** array together.
  * Finding that inserting the Javascript array of objects, ciMonitorNames, into the `u_imp_nagios` table takes a lot longer then processing jointly the **hosts** and **service** array.

* Created a Nagios XI data source script that seems to take hours to load `u_imp_nagios` table.
  * Wondering if chunking up the data into chunks of 1000 records might do the trick. Greg says about 15K is a good number.

```js
/* It's a self-executing function that loads data from Nagios XI. */
(function loadData(import_set_table) {
  var rest = new NagiosXiRestUtils();
  var ciMonitorNames = rest.getCiMonitorNames(); // Runs for about 3-5 minutes.
  var grNagiosImports = new GlideRecord("u_imp_nagios");
  var lastUpdateDate = new GlideDateTime();
  var chunkSize = 1000;

  for (var i = 0; i < ciMonitorNames.length; i += chunkSize) {
    var chunk = ciMonitorNames.slice(i, i + chunkSize);

    chunk.forEach(function (ciMonitorName) {
      grNagiosImports.initialize();
      grNagiosImports.u_alias = ciMonitorName.alias;
      grNagiosImports.u_last_update_date = lastUpdateDate;
      grNagiosImports.u_nagios_host_id = ciMonitorName.hostId;
      grNagiosImports.u_nagios_service_id = ciMonitorName.serviceId;
      grNagiosImports.u_service_description = ciMonitorName.serviceDescription;
      grNagiosImports.u_source = "Nagios XI";
      grNagiosImports.u_source_id = ciMonitorName.sourceId;
      grNagiosImports.setWorkflow(false);
      grNagiosImports.insert();
    });
  }
})(import_set_table);
```

* Sure enough, it did reduce the time!  Without running the transform it only took 6:03 minutes to request and load the table.  Testing it next with the transform.
