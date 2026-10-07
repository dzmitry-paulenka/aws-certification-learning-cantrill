# High Availability, Fault Tolerance, Disaster Recovery

- **High Availability (HA)** maximises uptime. Some disruption for users is fine, as long as the system recovers fast enough.
- **Fault Tolerance (FT)** keeps the system running while a failure happens. Users notice nothing. It is much harder to build, so it costs more.
- **Disaster Recovery (DR)** is the set of plans and steps to recover when HA and FT both fail. It has two parts: pre-planning and the recovery process itself.
- The three build on each other. An FT system is also highly available, but an HA system is not always fault tolerant. DR covers what's left when both give out.

```
failure happens
      │
      ├── FT:  system keeps working, no visible outage
      │
      ├── HA:  short outage, automatic or quick switch to a spare
      │
      └── DR:  both failed, run the recovery plan (backups, other site, runbooks)
```

---

## High Availability

HA aims for an agreed level of uptime, higher than normal. It **reduces outages**. Some still happen.

Uptime is given in "nines":

| Uptime | Downtime per year |
|---|---|
| 99% | 3.65 days |
| 99.9% ("three nines") | 8.77 hours |
| 99.99% ("four nines") | 52.6 minutes |
| 99.999% ("five nines") | 5.26 minutes |

Each extra nine costs more and needs more automation. A person can't log in and fix a server inside 5 minutes a year.

- Example: a car with a spare tyre. A puncture stops the car, you change the tyre, you drive on. There is downtime, but it is short.
- In IT: a primary server and a standby. When the primary fails, traffic moves to the standby. Users may see errors for a moment or need to log in again.
- HA usually means redundant components plus fast, ideally automatic, switchover.

---

## Fault Tolerance

FT means the system **keeps operating properly when some of its parts fail**. The failure doesn't reach the users.

- Example: a plane with several engines. One engine fails, the plane keeps flying. Changing a tyre in flight is not an option, so HA isn't enough.
- In IT: a patient monitoring system. Even a few seconds of outage is not acceptable.
- FT needs redundancy at every level, and the redundant parts run at the same time (active-active). Each part must be able to take the full load if its pair fails.
- Every layer that can fail needs a spare: servers, disks, network paths, power, the whole AZ or region.
- That is why FT costs more: you pay for capacity that sits ready all the time, and the design is more complex.

### HA or FT?

| | HA | FT |
|---|---|---|
| Goal | short outages | no outage |
| On failure | switch to a standby | keep running on the other parts |
| Users notice | maybe, briefly | no |
| Spare capacity | standby, may need warm-up | active, carrying load already |
| Cost | lower | higher |

Exam tip: if a question says the system "must continue to operate" or "no interruption", it wants FT. If it says "minimise downtime" or "recover quickly", HA is enough.

---

## Disaster Recovery

DR is the set of policies, tools and steps to recover vital systems after a disaster, natural or caused by people, when HA and FT didn't help. Examples: a whole site lost to fire or flood, a region-wide outage, data deleted or encrypted by ransomware.

DR has two parts.

### Pre-planning

Work done before a disaster, so recovery is possible at all:

- **Backups** stored off-site, away from the systems they protect. A backup in the same building burns with it.
- **Standby site**: somewhere to run the systems. It can range from an empty office to a full copy of the setup.
- **Credentials and access** kept outside the main site, so you can still log in to backups and accounts.
- **Documented process**: who does what, in which order. Nobody should have to work it out during the disaster.
- **Regular tests** of the plan. An untested backup or runbook may not work when you need it.

### The DR process

Work done during and after the disaster:

- declare the disaster and start the plan;
- restore data from backups;
- bring systems up at the standby site;
- move users and traffic over;
- later, return to the main site once it's rebuilt.

The goal is to protect data first and bring the business back with as little loss as possible.
