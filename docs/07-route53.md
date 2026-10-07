# Route 53

- Route 53 (R53) is the AWS DNS service. It has two main jobs: **register domains** and **host zones**.
- The two are separate. You can register a domain with R53 and host its zone elsewhere, or register it elsewhere and host the zone in R53.
- R53 is a **global service** with a single database. You don't pick a region, and its data is replicated worldwide.
- The name: DNS runs on port 53.

---

## Registering a domain

R53 is a **registrar**: it sells domain names. Each top-level domain (TLD) such as `.com` or `.org` is run by a **registry**, which keeps the TLD zone. R53 has deals with many registries.

When you register `animals4life.org`:

1. R53 asks the `.org` registry whether the name is free.
2. R53 creates a **hosted zone** for the domain: a zone file served by 4 name servers that R53 allocates.
3. R53 asks the `.org` registry to add **NS records** for those 4 name servers into the `.org` zone.

Step 3 makes the R53 name servers **authoritative** for the domain. Anyone resolving `animals4life.org` gets sent from the `.org` servers to them.

```
root zone
   │  NS → .org servers
   ▼
.org TLD zone (run by the .org registry)
   │  NS records for animals4life.org → R53 name servers   ← R53 asks the registry to add these
   ▼
R53 hosted zone animals4life.org (4 name servers)
   └─ www  A  203.0.113.10
```

Registration costs a yearly fee that depends on the TLD.

---

## Hosted zones

A hosted zone is R53's version of a DNS zone file.

- R53 allocates and runs **4 name servers** per zone and stores the zone on each of them. The 4 servers sit under different TLDs (e.g. `.com`, `.net`, `.org`, `.co.uk`), so a problem with one TLD doesn't break the zone.
- A hosted zone costs a monthly fee, plus a charge per query.
- The zone holds DNS **records**. R53 groups them into **record sets** (see below).

### Public and private zones

| | Public hosted zone | Private hosted zone |
|---|---|---|
| Who can query it | anyone on the public internet | only the VPCs you associate with it |
| Name servers | 4 public R53 name servers | none public; VPCs resolve it through the VPC resolver |
| Typical use | your website, public APIs | internal names, e.g. `db.internal.animals4life.org` |

- A public zone is also reachable from inside VPCs, through the VPC's DNS resolver.
- You can associate a private zone with VPCs in other accounts.
- **Split-view DNS**: a public and a private zone with the same name. Internal clients get internal answers, the internet gets public ones. Records only in the private zone stay hidden from the internet.

---

## Records and record sets

A **record set** is all records in a zone with the same **name** and **type**. For example, two `A` records for `www.animals4life.org` pointing to two IPs form one record set. R53 returns both, and the client picks one.

Each record set has a **TTL**: how long resolvers may cache the answer, in seconds. Low TTL means changes spread fast but more queries hit R53.

Common record types:

| Type | Points a name to | Example |
|---|---|---|
| `NS` | the name servers for a zone | the 4 R53 servers |
| `A` | an IPv4 address | `www` → `203.0.113.10` |
| `AAAA` | an IPv6 address | `www` → `2001:db8::10` |
| `CNAME` | "canonical" name, the equivalent of DNS "shortcuts" | `blog` → `www.animals4life.org` |
| `MX` | mail servers, with a priority | `10 mail.animals4life.org` |
| `TXT` | free text, often to prove you own a domain | `"google-site-verification=..."` |

A `CNAME` can't sit at the zone apex (`animals4life.org` itself), only on names under it.
