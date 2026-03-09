Lab: Security Telemetry Optimization Lab: Reducing Falcon NG-SIEM Ingestion Costs with Cribl Stream
================================================

1\. Objective
-------------

The goal of this lab is to validate an end-to-end detection pipeline where Suricata detects suspicious network activity on the local host, forwards alerts through syslog/rsyslog to Cribl, and then pushes the events into Falcon Next-Gen SIEM for search and analysis.

The diagram below illustrates a security telemetry pipeline designed to detect network threats while reducing SIEM ingestion costs. On the left, a Linux host running Suricata IDS monitors network traffic and detects suspicious activity or attacks. This host also communicates with an Azure Virtual Machine, representing infrastructure in a cloud environment. When Suricata generates alerts, they are forwarded via syslog to Cribl Stream, which acts as the log processing and optimization layer. Cribl processes the incoming alerts by applying filtering, suppression, and sampling techniques, reducing duplicate or low-value events to control the amount of data sent to the SIEM and optimize ingestion costs. After this processing, the refined alerts are forwarded to CrowdStrike Falcon Next-Gen SIEM, where they are indexed and analyzed. Finally, security analysts in the SOC environment interact directly with Falcon Next-Gen SIEM, using it to investigate alerts, perform threat hunting, and monitor security events across the environment.

<img width="1536" height="1024" alt="ChatGPT Image Mar 10, 2026, 08_38_21 AM" src="https://github.com/user-attachments/assets/493b3cde-d41f-4fad-8dde-d3e3b98ca115" />


This lab proves five things:

1.  Suricata is generating IDS alerts.

2.  Those alerts are being forwarded over syslog.

3.  Cribl is receiving and parsing the events.

4.  Cribl is able to route the events to Falcon NG-SIEM.

5.  The events are searchable inside Falcon NG-SIEM.

* * * * *

2\. Architecture
----------------

The pipeline you built is:

Test traffic\
   ↓\
Suricata IDS\
   ↓\
syslog / rsyslog\
   ↓\
Cribl Syslog Source (in_syslog)\
   ↓\
Cribl Pipeline (test)\
   ↓\
Falcon Next-Gen SIEM Destination (integration-falcon-ng)\
   ↓\
Falcon NG-SIEM Search

This is a realistic SOC-style ingestion pattern for lightweight network IDS telemetry.

* * * * *

3\. Lab Environment Summary
---------------------------

From your screenshots, the important configuration elements are:

### Suricata side

-   Suricata is producing syslog-formatted alerts.

-   The events show:

    -   `appname = suricata`

    -   `facilityName = local5`

    -   `host = bmmau`

### Cribl source

The Syslog source `in_syslog` is configured with:

-   Address: `0.0.0.0`

-   UDP port: `9514`

-   TCP port: `9514`

This means Cribl is listening on all interfaces for syslog events on port 9514.

### Cribl destination

The Falcon NG-SIEM destination `integration-falcon-ng` is configured with:

-   CrowdStrike ingest endpoint

-   Authentication token

-   Request format: JSON

-   Backpressure behavior: Block

This means Cribl is pushing structured events to Falcon.

### Routing

In QuickConnect:

-   Source: `Syslog (in_syslog)`

-   Destination: `Falcon Next-Gen SIEM (integration-falcon-ng)`

This confirms the path is wired correctly.

### Pipeline

You created a pipeline named `test` with:

-   `Suppress`

-   `Sampling`

Both functions are enabled.

* * * * *

4\. Test Methodology
--------------------

You generated repeated IDS events using:

for i in {1..50}; do curl -s http://testmyids.com > /dev/null; sleep 1; done

This command is useful because `testmyids.com` intentionally returns content that matches common IDS signatures. It is widely used for validating that IDS signatures, log forwarding, and SIEM ingestion are functioning.

* * * * *

5\. What Suricata Detected
--------------------------

Your captured events show repeated Suricata alerts like:

-   `GPL ATTACK_RESPONSE id check returned root`

-   `SURICATA STREAM Packet with invalid ack`

-   `SURICATA STREAM FIN invalid ack`

-   `SURICATA STREAM ESTABLISHED packet out of window`

-   `ET INFO Spotify P2P C...`

-   `GNU/Linux APT User-Agent Outbound`

-   `PNG in HTTP POST (Outbound)`

The main detection you intentionally triggered is:

GPL ATTACK_RESPONSE id check returned root

### Why this fired

`testmyids.com` is designed to trigger classic IDS signatures. The response payload resembles output associated with command execution, especially the type of text a rule may identify as command output from a compromised host, such as `uid=0(root)`-style content.

Suricata sees the HTTP response, matches the payload against a rule, and generates an alert.

### Interpretation

This does not mean your host is compromised. It means:

-   the sensor is inspecting traffic correctly

-   the rules are loaded correctly

-   the HTTP response matched a known signature

-   the alert pipeline is working end-to-end

That is exactly what you want in a validation lab.

* * * * *

6\. Cribl Ingestion Findings
----------------------------

Your Cribl Capture Sample Data screenshot confirms Cribl is ingesting the Suricata syslog records successfully.

The fields visible include:

-   `_raw`

-   `_time`

-   `appname`

-   `facility`

-   `facilityName`

-   `host`

-   `message`

-   `procid`

-   `severity`

-   `severityName`

### What this means

Cribl is correctly parsing syslog structure from the incoming Suricata logs. This is important because it means the alerts are not arriving as opaque blobs only; they already have usable metadata for search and routing.

### Key observation

The events are consistently tagged with:

-   `appname = suricata`

-   `facilityName = local5`

These two fields become your best selectors for both Cribl filtering and Falcon searches.

* * * * *

7\. Falcon NG-SIEM Findings
---------------------------

Your Falcon query:

facilityName="local5" appname="suricata"\
| table([@timestamp, @rawstring])

successfully returns events.

This proves:

-   Falcon is receiving the logs from Cribl

-   the Cribl destination configuration is valid

-   the CrowdStrike collector endpoint and token are correct

-   the events are indexed and searchable

### Additional observation

The search results show numerous Suricata alerts over time, not just a single test event. That means the pipeline is stable enough to ingest repeated activity bursts.

* * * * *

8\. Connection and Ingest Findings
----------------------------------

From the Data Connections screenshot in Falcon:

-   Connection name: `AWS`

-   Vendor: `Generic`

-   Product: `Cribl`

-   Connector type: `Push`

-   Parser: `cribl-integration-...`

-   Subscription: `Next-Gen SIEM`

-   Ingest (24h): about `560.58 kB`

-   Daily ingest shown: about `559.62 kB today`

### Interpretation

This confirms Falcon recognizes Cribl as an upstream push connector and is receiving data volume from it.

### Important note

The connection status shows `Idle`, which is not a failure. It usually means:

-   the connection is healthy

-   no event burst is actively arriving at that exact moment

Because your search returns data and the 24-hour ingest count is non-zero, the connector is working.

* * * * *

9\. Pipeline Function Findings
------------------------------

Your pipeline contains two enabled functions:

1.  `Suppress`

2.  `Sampling`

### 9.1 Suppress

The Suppress function is intended to reduce duplicate events.

If configured too broadly, it can hide useful alerts. For example, if you suppress on `host` only, then many different Suricata detections from the same machine may be dropped even though they are different alerts.

A better approach for Suricata is usually:

-   filter only Suricata events

-   suppress by `message`, or by `host + message`

Example logic:

-   allow the first identical Suricata alert

-   drop duplicates for 30--60 seconds

This prevents alert storms while preserving unique detections.

### 9.2 Sampling

Sampling reduces overall event volume by only forwarding a percentage of events.

This is useful for high-volume data streams, but it can be dangerous for security alerts because:

-   you may lose true positives

-   event counts become unreliable

-   repeated alerts may not appear consistently in Falcon

### Recommendation

For a security detection lab:

-   keep `Suppress`

-   disable `Sampling` for Suricata alerts unless you explicitly want cost control testing

Because Suricata alerts are already detections, not raw packet logs, sampling them weakens visibility.

* * * * *

10\. What the Results Mean
--------------------------

This lab is successful.

You proved all core stages:

### Detection success

Suricata generated real alerts from live HTTP requests to `testmyids.com`.

### Transport success

The logs were sent via syslog and received by Cribl on port 9514.

### Parsing success

Cribl extracted fields such as `appname`, `facilityName`, `host`, and `message`.

### Routing success

QuickConnect sent the events from the Syslog source to the Falcon destination.

### SIEM ingestion success

Falcon NG-SIEM indexed the events and allowed them to be queried using CQL.

* * * * *

11\. Recommended Falcon Queries
-------------------------------

### Basic Suricata search

facilityName="local5" appname="suricata"

### Table view

facilityName="local5" appname="suricata"\
| table([@timestamp, @rawstring])

### Group by message

facilityName="local5" appname="suricata"\
| groupBy([message], function=count())

### Look for the testmyids alert

facilityName="local5" appname="suricata" message=/id check returned root/i

### Count alerts by host

facilityName="local5" appname="suricata"\
| groupBy([host], function=count())

* * * * *

12\. Recommended Cribl Improvements
-----------------------------------

Keep
----

-   Syslog source on 9514

-   Falcon destination

-   QuickConnect path

Change
------

-   disable `Sampling` for security alerts

-   narrow the `Suppress` filter to Suricata events only

-   suppress on a better key than just `host`

### Better Suppress strategy

Use:

**Filter**

appname=="suricata"

**Key expression**

`${host}-${message}`

**Number to allow**

1

**Suppression period**

60

**Drop suppressed events**

On

This lets one copy of each unique alert through per host every 60 seconds.

* * * * *

13\. Limitations of the Lab
---------------------------

This lab validates ingestion and detection, but there are limitations:

### 1\. Syslog is flat text

The current Suricata-to-Cribl flow is based on syslog text, so Falcon mostly sees string-based data.

That means fields like:

-   source IP

-   destination IP

-   signature ID

-   alert category

are embedded in the message, not fully structured.

### 2\. Better option exists

A better production design is to send Suricata `eve.json` into Cribl instead of syslog. That would preserve structured fields such as:

-   `src_ip`

-   `dest_ip`

-   `event_type`

-   `alert.signature`

-   `alert.category`

-   `proto`

This makes Falcon hunting much stronger.

### 3\. Sampling can distort reality

If left enabled, sampling makes event frequency analysis less reliable.

* * * * *

14\. Final Conclusion
---------------------

This lab successfully demonstrates a functional end-to-end IDS telemetry pipeline from Suricata into Falcon NG-SIEM through Cribl.

### Confirmed findings

-   Suricata detects the `testmyids.com` traffic correctly.

-   rsyslog/syslog forwarding works.

-   Cribl receives, parses, and routes the data.

-   Falcon NG-SIEM ingests and indexes the alerts.

-   Searches using `facilityName="local5"` and `appname="suricata"` return the expected events.

-   Falcon's data connection metrics confirm Cribl push ingestion is active.

### Security value of the lab

This lab gives you a working detection engineering environment where you can:

-   validate new Suricata rules

-   test Cribl routing and suppression logic

-   confirm SIEM ingestion

-   practice Falcon hunting queries

-   experiment with alert reduction strategies
