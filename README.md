# OSSEC Security Monitoring Lab

## Overview
I used a Labtainers environment in Oracle VirtualBox to validate custom OSSEC rules for Apache web requests. I investigated monitoring failures, restored container connectivity, and verified alerts using controlled requests.

This project builds on an existing school lab. My work included troubleshooting, testing existing custom rules, analyzing alerts, and documenting the results.

## Tools and Environment

- Oracle VirtualBox and Labtainers
- OSSEC HIDS 3.6.0
- Apache HTTP Server
- Docker networking
- Linux command-line tools and curl

## Monitoring Workflow

1. Client1 sends an HTTP request to WebServer.
2. Apache records the request in `/var/log/httpd/access_log`.
3. The WebServer OSSEC agent forwards the event to the OSSEC server.
4. OSSEC evaluates the event against its rules and records matching alerts.

## Detection Rules

The tested configuration is saved in [rules/local_rules.xml](rules/local_rules.xml).

- **Rule 140234 — Level 7:** Matches parent rules 31108 or 31100 and the URL pattern `plan.html`.
- **Rule 140235 — Level 7:** Matches parent rule 31101 and the URL pattern `plan`.

## Tests and Results

Tests were performed on October 19, 2026. Times below are UTC.
### Test 1: Business Plan Access
- **Request:** `/plan.html`
- **HTTP response:** 200 OK
- **Observed rule:** 140234, level 7
- **Request time:** 23:23:22
- **Alert time:** 23:23:24

### Test 2: Missing Business Plan Page
- **Request:** `/plan9.html`
- **HTTP response:** 404 Not Found
- **Observed rule:** 140235, level 7
- **Request time:** 23:27:53
- **Alert time:** 23:27:54

### Test 3: Unrelated Missing Page
- **Request:** `/about.html`
- **HTTP response:** 404 Not Found
- **Observed rule:** 31101, level 5
- **Request time:** 23:32:48
- **Alert time:** 23:32:49
  
Both custom rules generated alerts for their test requests. The unrelated missing page triggered the standard web error rule instead of either custom rule.

The timestamp differences were approximately one to two seconds for these individual tests. They are not a performance benchmark.

### Test Commands

Run from Client1 in the configured lab:

    curl -i --max-time 10 http://172.0.0.4/plan.html
    curl -i --max-time 10 http://172.0.0.4/plan9.html
    curl -i --max-time 10 http://172.0.0.4/about.html

Review alerts on the OSSEC server:

    sudo tail -n 50 /var/ossec/logs/alerts/alerts.log

## Troubleshooting

### Container networking

Initially, the containers had no attached Docker network. The existing `lan` network used `172.25.0.0/24`, while the OSSEC lab configuration required `172.0.0.0/24`.

I backed up the custom rules, recreated the empty network with the expected settings, and connected each container using its configured IP address. A successful HTTP request confirmed client-to-server connectivity.

### OSSEC receiver and agent connection

The OSSEC server log showed that `ossec-remoted` had failed to bind to port 1514 during startup.

After restoring networking, I restarted OSSEC and confirmed that the receiver was running. I then restarted the WebServer agent and verified its Active status using:

    sudo /var/ossec/bin/agent_control -l

Fresh web requests subsequently produced alerts on the OSSEC server.

### Log collection path

The manager configuration referenced a missing local Apache log path. Further investigation showed that the WebServer agent monitored the actual Apache access log and forwarded events remotely. A shared log directory was not required for this workflow.

## Evidence

Screenshots in this repository document:

- OSSEC processes and custom rules
- Successful HTTP connectivity
- Receiver recovery and WebServer agent connection
- Successful access to `/plan.html` and its custom alert
- A missing `/plan9.html` request and its custom alert
- An unrelated `/about.html` request and its standard alert

## Analysis and Limitations

- An alert shows that activity matched a rule; it does not prove malicious intent or unauthorized access.
- A 404 response means the requested resource was not found. It does not by itself establish an attack.
- Rule 31101 displayed “Web server 400 error code,” while the underlying event recorded HTTP 404.
- The current URL expressions are broad: `plan` can match unrelated URLs containing that text, and the dot in `plan.html` is not escaped. Additional testing and more precise expressions would improve specificity.
- The comparison test validates one unrelated URL; it does not establish a false-positive rate.
- Client1’s OSSEC agent remained disconnected at the last status check. Its HTTP requests still worked, and this project validated monitoring through the WebServer agent.
- Active-response actions were configured, but their blocking behavior was not validated in this project.

## Skills Demonstrated

- Linux troubleshooting
- Docker network inspection and repair
- Apache access-log analysis
- OSSEC agent and receiver troubleshooting
- Custom detection-rule validation
- Comparison testing and evidence collection
- Technical documentation

## What I Learned

Working detection rules depend on a functioning monitoring pipeline. I learned to check network connectivity, service processes, agent status, and log collection before assuming a rule was broken.

Comparing a targeted request with an unrelated request also helped me distinguish custom detections from standard web error alerts.
