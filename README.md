# Home NOC Lab

A virtual Network Operations Center (NOC) practice environment: monitoring, alerting, incident triage, and documentation, built on my own Windows PC with Docker and Zabbix.

> \\\*\\\*This is a home lab, not production experience.\\\*\\\* Outages are simulated by stopping containers on purpose. The goal was to practice the NOC workflow (detect, triage, resolve, document) with real alerts firing on a real monitoring system.

## What it does

* Runs a complete monitoring stack in Docker (Zabbix server, web interface, PostgreSQL database, and a monitoring agent).
* Monitors a test web server (`web-target`) with an HTTP check and an ICMP ping check.
* Fires a High severity alert when the web server stops responding.
* Supports practicing four incident scenarios and writing each one up as an incident record.

## Stack

|Component|Purpose|
|-|-|
|Docker Desktop (WSL 2)|Runs every service in a container|
|Zabbix 7.0 LTS|Monitoring, triggers, and alerting|
|PostgreSQL 16|Zabbix database|
|Zabbix Agent 2|Monitors the Docker environment itself|
|nginx|Test web server that can be stopped to create incidents|
|Grafana (optional)|Included in `docker-compose.yml` behind a profile; **not used** in the exercises below|

## Quick start (Windows)

1. Install [Docker Desktop](https://www.docker.com/products/docker-desktop/) with the WSL 2 backend and confirm it works:

```
   docker run --rm hello-world
   ```

2. Copy `env.example` to `.env` and set your own passwords:

```
   copy env.example .env
   notepad .env
   ```

3. Start the stack:

```
   docker compose up -d
   ```

4. Open [http://localhost:8080](http://localhost:8080) and log in as `Admin` with password `zabbix`. **Change this password immediately.**
5. In Zabbix:

   * Point the built-in **Zabbix server** host at the agent: interface DNS name `zabbix-agent`, *Connect to: DNS*, port `10050`.
   * Create a host named `web-target` (DNS name `web-target`) with two simple-check items: `net.tcp.service\\\[http,,80]` and `icmpping`, each every 15 seconds.
   * Add the trigger described under [Trigger tuning](#trigger-tuning).

To break the web server on purpose and watch the alert:

```
docker compose stop web-target
docker compose start web-target
```

To shut down, run `docker compose down`. To also wipe all lab data, run `docker compose down -v`.

## The four incidents

All incidents ran on 10/26/2026. Full records, the alert log, and a shift handoff note are in [`docs/Home\\\_NOC\\\_Lab\\\_Incident\\\_Log.xlsx`](docs/Home_NOC_Lab_Incident_Log.xlsx).

|ID|Exercise|What happened|Lesson|
|-|-|-|-|
|INC-001|Service down|Stopped `web-target`. A High alert fired at 1:47 PM and cleared at 1:49 PM after restart.|A healthy baseline (both checks at 1) makes an incident easy to recognize.|
|INC-002|Monitoring agent unavailable|Stopped the agent. An Average alert ("agent not available for 3m") fired at 2:08 PM and cleared at 2:12 PM. No monitored service was actually down.|Tell a real outage from a monitoring gap.|
|INC-003|Flapping service and trigger tuning|Repeated stop/start produced three High alerts in about 7 minutes. I tuned the trigger until short blips stopped alerting while real outages still did.|A trigger has several conditions; tuning one does not tune the others.|
|INC-004|Two problems at once|Stopped `web-target` and the agent together. `web-target` alerted 1m 05s later, the agent 3m 26s later. Worked the High outage first.|Alerts have different detection delays, so a quiet screen is not proof that everything is healthy.|

## Trigger tuning

INC-003 started with a trigger that alerted on every brief outage. I changed it in two passes and tested each one.

|Version|Expression|Result|
|-|-|-|
|v1 (original)|`last(...)=0 or nodata(...,1m)=1`|Three separate alerts in about 7 minutes (flapping).|
|v2|`last(...)=0 or nodata(...,3m)=1`|A real 3-minute outage alerted after about 3 minutes, but 30-second blips still alerted because `last()=0` fires on a single failed check.|
|v3 (final)|`max(...,#4)=0 or nodata(...,3m)=1`|Two 30-second stops produced **no alerts**. A 2-minute stop still alerted (3:33 PM).|

The final trigger in full:

```
max(/web-target/net.tcp.service\\\[http,,80],#4)=0 or nodata(/web-target/net.tcp.service\\\[http,,80],3m)=1
```

`max()` over the last 4 checks is 0 only if all 4 failed, which means about a minute of continuous failure at 15-second checks. (`min()` would be 0 if any single check failed, so it would not have fixed the problem.)

**Tradeoff:** fewer pages for short blips, at the cost of about a minute of extra detection delay. Outages under about a minute are no longer reported. Whether that is acceptable depends on how critical the service is.

## What I practiced

* Triaging alerts by severity and impact
* Telling real outages from monitoring gaps
* Writing incident records with timelines, root cause, and follow-ups
* Tuning alert thresholds to reduce noise and testing the result
* Writing a shift handoff note
* Working from a monitoring stack I set up and troubleshot myself (Docker, WSL 2, Zabbix)

## What I learned



My exercises taught me to tell a real outage from a monitoring gap. When I stopped the web server, that was a real outage, because a service was down. When I stopped the agent, nothing was actually broken: monitoring had lost contact with a healthy system. Running two problems at once showed me that alerts arrive late and in different orders, so a quiet screen doesn’t mean everything is fine. It also taught me that simultaneous alerts can share a cause, so I’d restore service first when that’s appropriate, then dig into the root cause instead of stopping at the quick fix. When Setting up the stack taught me that a monitoring system is only useful once it is tuned and trusted. The flapping exercise showed me that an alert can fire for the wrong reasons, and that I only found the real cause by testing each change instead of assuming. Handling two alerts at once made it clear that alerts arrive late, in different orders, and that silence on the screen does not mean everything is healthy.

## Limitations

* Single PC, simulated outages (containers stopped manually), no real network hardware or production traffic.
* No notification or on-call tooling (alerts are viewed in the Zabbix web interface).
* Not meant for production: default credentials must be changed and the setup is not hardened.

## Repository contents

```
README.md
docker-compose.yml
env.example            copy to .env (never commit .env)
docs/
  Home\\\_NOC\\\_Lab\\\_Incident\\\_Log.xlsx
  Home\\\_NOC\\\_Lab\\\_Guide.docx 
```
* Zabbix Problems page during an active alert, and again after it clears
* The trigger configuration (final expression)
* The Alert Log or Summary tab of the incident spreadsheet

