# Private DNS Architecture for SAP PCE and Public Cloud Integration

> **Document Type:** Architecture / Operations Guide  
> **Scope:** SAP PCE ↔ KC Internal Systems  
> **DNS Platform:** BIND9  
> **Cloud DNS:** AWS Route 53  
> **HA Model:** Primary / Secondary DNS

---

# 1. Overview

## 1.1 Purpose

This document defines the Private DNS architecture for communication between **SAP PCE** and **KC Internal Systems** hosted in a Public Cloud environment.

The primary objective is to provide SAP PCE with a DNS resolution path that:

- Uses Private Network connectivity
- Supports DNS High Availability
- Allows SAP PCE to resolve KC Internal System services
- Supports existing Public DNS namespaces where possible
- Provides an alternative architecture when SAP PCE requires fully private DNS resolution

---

# 2. Environment

The overall environment consists of the following components:

```text
SAP PCE
   │
   │ Private Connectivity
   │
   ▼
KC Internal Systems
   │
   ├── Private DNS
   ├── Applications
   └── Other Internal Services
   │
   ▼
AWS Public Cloud
   │
   └── Route 53
```

Private connectivity may be implemented using technologies such as:

- IPSec VPN
- Direct Connect / private interconnect
- Other approved private connectivity mechanisms

The exact connectivity method is outside the scope of this document.

---

# 3. Requirements

## 3.1 SAP PCE Requirements

SAP PCE requires DNS resolution for services hosted in KC Internal Systems.

Example:

```text
start.example.com
        │
        ▼
   192.168.10.100
```

The SAP PCE DNS environment may require the Conditional Forwarder destination to be reachable through a **Private IP address**.

Therefore, the KC environment provides a Private DNS Server that SAP PCE can query through the private network.

---

# 4. Private DNS HA Architecture

The KC Private DNS infrastructure consists of two BIND9 servers.

```text
                         SAP PCE
                            │
                            │ DNS Query
                            │
                            ▼
                  ┌────────────────────┐
                  │ KC Private DNS     │
                  │                    │
                  │ Primary DNS        │
                  │ 192.168.10.11      │
                  │                    │
                  │ Secondary DNS      │
                  │ 192.168.20.11      │
                  └────────────────────┘
```

The actual server IP addresses should be managed separately from this document.

---

# 5. DNS HA Model

The DNS architecture uses a **Primary / Secondary** model.

```text
                 ┌─────────────────┐
                 │  Primary DNS    │
                 │                 │
                 │  BIND9          │
                 └────────┬────────┘
                          │
                     AXFR / IXFR
                          │
                          ▼
                 ┌─────────────────┐
                 │ Secondary DNS   │
                 │                 │
                 │ BIND9           │
                 └─────────────────┘
```

## Primary DNS

Responsibilities:

- Authoritative DNS Zone management
- Zone file modification
- SOA serial management
- Zone transfer source
- NOTIFY generation

## Secondary DNS

Responsibilities:

- Maintain a synchronized copy of the Zone
- Serve authoritative DNS queries
- Receive AXFR / IXFR from Primary DNS
- Provide DNS availability when Primary DNS is unavailable

---

# 6. DNS Failover Model

DNS failover is primarily handled by the **client resolver configuration**.

The DNS Servers should be configured with multiple DNS endpoints.

Example:

```text
Client
 ├── DNS 1 → 192.168.10.11
 └── DNS 2 → 192.168.20.11
```

Normal operation:

```text
Client
   │
   ├──────────────► Primary DNS
   │                    │
   │                    ▼
   │                 Response
   │
   └──────────────► Secondary DNS
```

If the Primary DNS becomes unavailable:

```text
Client
   │
   ├──────────────► Primary DNS
   │                    X
   │
   └──────────────► Secondary DNS
                        │
                        ▼
                     Response
```

The DNS servers do not actively "take over" each other's IP addresses.

The client resolver detects an unavailable DNS server and uses another configured DNS server.

---

# 7. DNS Namespace

The example Private DNS namespace used in this document is:

```text
example.com
```

Example record:

```text
start.example.com
```

Example response:

```text
start.example.com. IN A 192.168.10.100
```

Actual production domain names and IP addresses should be maintained in the appropriate configuration management system.

---

# 8. Important DNS Concept

A Public Hosted Zone does not necessarily mean that the records contain Public IP addresses.

For example:

```text
AWS Route 53
Public Hosted Zone
example.com
        │
        └── start.example.com
                │
                └── A
                    192.168.10.100
```

The Zone is a **Public Hosted Zone**, while the record points to a **Private IP address**.

Therefore:

```text
Public Hosted Zone
        ≠
Public IP Record
```

However, the authoritative DNS servers hosting the Public Hosted Zone are still part of the public DNS infrastructure.

This distinction is important when evaluating SAP PCE network requirements.

---

# 9. DNS Architecture Options

Two architectures are considered.

## Option 1

Use the existing AWS Route 53 Public Hosted Zone and forward DNS queries through the KC Private DNS infrastructure.

```text
SAP PCE
   │
   │ Private DNS Query
   ▼
KC Private DNS
   │
   │ DNS Forwarding
   ▼
AWS Route 53
Public Hosted Zone
   │
   ▼
start.example.com
   │
   ▼
192.168.10.100
```

## Option 2

Create a dedicated Private DNS Zone for SAP PCE.

```text
SAP PCE
   │
   │ Private DNS Query
   ▼
KC Private DNS
   │
   ▼
Private DNS Zone
int-example.com
   │
   ▼
start.int-example.com
   │
   ▼
192.168.10.100
```

KG users can continue using the existing DNS namespace:

```text
KG User
   │
   ▼
Existing DNS
   │
   ▼
start.example.com
```

---

# 10. Option 1 - Route 53 Public Hosted Zone

## 10.1 Architecture

```text
                         SAP PCE
                            │
                            │ DNS Query
                            │
                            ▼
                 ┌─────────────────────┐
                 │ KC Private DNS HA   │
                 │                     │
                 │ Primary             │
                 │ Secondary           │
                 └──────────┬──────────┘
                            │
                            │ Conditional
                            │ Forwarding
                            │
                            ▼
                 ┌─────────────────────┐
                 │ AWS Route 53        │
                 │ Public Hosted Zone  │
                 │                     │
                 │ example.com         │
                 └──────────┬──────────┘
                            │
                            ▼
                    start.example.com
                            │
                            ▼
                       192.168.10.100
```

---

# 11. Option 1 - DNS Query Flow

The complete DNS resolution flow is:

```text
1. SAP PCE
   │
   │ Query:
   │ start.example.com
   ▼

2. SAP PCE DNS
   │
   │ Conditional Forwarder
   ▼

3. KC Private DNS
   │
   │ Query matches example.com
   ▼

4. KC Private DNS
   │
   │ Forward query
   ▼

5. Route 53 Authoritative DNS
   │
   │ Resolve record
   ▼

6. DNS Response
   │
   │ 192.168.10.100
   ▼

7. KC Private DNS
   │
   ▼

8. SAP PCE
```

---

# 12. Option 1 - Required Network Paths

Option 1 contains two separate network paths.

## Path A - SAP PCE → KC Private DNS

```text
SAP PCE
   │
   │ Private Network
   │ UDP/TCP 53
   ▼
KC Private DNS
```

This path must be permitted through the private network.

---

## Path B - KC Private DNS → Route 53

```text
KC Private DNS
   │
   │ UDP/TCP 53
   ▼
Route 53 Authoritative DNS
```

This path is critical.

The fact that SAP PCE reaches the KC DNS Server privately does not automatically mean that the entire DNS resolution path is private.

The KC DNS Server may still need to communicate with the Route 53 authoritative DNS infrastructure through the Public Network.

---

# 13. Option 1 - When It Is Available

Option 1 is available when the SAP PCE requirement is:

> The Conditional Forwarder destination must be reachable through a Private IP.

Example:

```text
SAP PCE
   │
   │ Private
   ▼
KC Private DNS
   │
   │ Public DNS Query
   ▼
Route 53
```

This architecture is possible if:

- SAP PCE can reach KC Private DNS through the private network
- SAP PCE allows DNS forwarding to the KC Private DNS
- KC Private DNS can communicate with Route 53 authoritative DNS
- UDP/TCP port 53 is permitted where required
- Public Network DNS resolution is permitted by SAP PCE policy
- There is no requirement for the entire DNS resolution path to remain private

---

# 14. Option 1 - When It Is Not Available

Option 1 becomes unavailable if SAP PCE requires the **entire DNS resolution path to remain private**.

Example:

```text
SAP PCE
   │
   │ Private
   ▼
KC Private DNS
   │
   │
   X
   │
   │ Public DNS Resolution
   ▼
Route 53
```

If the SAP PCE security policy prohibits the KC DNS Server from relying on Public DNS infrastructure, Option 1 cannot be used.

Option 2 should then be considered.

---

# 15. SAP PCE Policy Questions

The following questions must be confirmed with the SAP PCE team before selecting Option 1.

### Question 1

Does SAP PCE only require the Conditional Forwarder DNS Server to have a Private IP?

```text
SAP PCE
   │
   ▼
Private DNS IP
```

Or:

### Question 2

Must the entire DNS resolution path remain private?

```text
SAP PCE
   │
   ▼
Private DNS
   │
   ▼
Private Authoritative DNS
```

---

### Question 3

Is Public Network DNS communication permitted from the KC Private DNS Server?

```text
KC Private DNS
       │
       ▼
Public DNS Infrastructure
```

---

### Question 4

Are UDP/53 and TCP/53 allowed from the KC DNS Server to the Route 53 authoritative DNS servers?

---

# 16. Option 2 - Dedicated Private DNS Zone

If SAP PCE requires the entire DNS resolution path to remain private, a dedicated Private DNS Zone can be created.

Example:

```text
SAP PCE
   │
   │ Private Network
   ▼
KC Private DNS
   │
   ▼
Private Zone
int-example.com
   │
   ▼
start.int-example.com
   │
   ▼
192.168.10.100
```

No Public DNS dependency is required for SAP PCE DNS resolution.

---

# 17. Option 2 - KG User Flow

KG users can continue using the existing DNS namespace.

```text
KG User
   │
   ▼
Existing DNS
   │
   ▼
example.com
   │
   ▼
start.example.com
   │
   ▼
192.168.10.100
```

SAP PCE uses a separate namespace:

```text
SAP PCE
   │
   ▼
Private DNS
   │
   ▼
int-example.com
   │
   ▼
start.int-example.com
```

---

# 18. Option 2 - Advantages

- Entire SAP PCE DNS resolution path remains private
- No dependency on Public DNS infrastructure
- Clear separation between SAP PCE and KG DNS namespaces
- Easier alignment with strict security policies
- Independent DNS management for SAP PCE
- Reduced dependency on external DNS availability

---

# 19. Option 2 - Disadvantages

- Additional DNS Zone management
- DNS records may need to be maintained in multiple Zones
- Different hostnames may be required for SAP PCE and KG users
- Application configuration may differ between environments
- TLS certificate management may require additional SANs/domains
- Additional operational procedures required

---

# 20. Option Comparison

| Category | Option 1 | Option 2 |
|---|---|---|
| SAP PCE DNS Server | KC Private DNS | KC Private DNS |
| SAP PCE Namespace | `example.com` | `int-example.com` |
| KG Namespace | `example.com` | `example.com` |
| Route 53 Public Hosted Zone | Used | Not required |
| Public DNS Dependency | Yes | No |
| Private DNS Path | Partial | Full |
| Duplicate Records | Minimal | Possible |
| Namespace Management | Single | Separate |
| Operational Complexity | Lower | Higher |
| Security Policy Dependency | Higher | Lower |
| SAP PCE Compatibility | Policy dependent | Generally easier |

---

# 21. BIND9 Configuration Model

The Private DNS Servers use BIND9.

## Primary DNS

Example:

```text
/etc/bind/named.conf.local
```

```conf
zone "example.com" {
    type primary;
    file "/etc/bind/db.example.com";

    allow-query {
        trusted;
    };

    allow-transfer {
        192.168.20.11;
    };

    also-notify {
        192.168.20.11;
    };
};
```

The actual ACL and IP addresses must be adapted to the production network.

---

# 22. Secondary DNS Configuration

The Secondary DNS should be configured as:

```conf
zone "example.com" {
    type secondary;

    primaries {
        192.168.10.11;
    };

    file "/var/cache/bind/db.example.com";
};
```

The Secondary DNS does not require manual editing of the Zone file.

The Zone is received from the Primary DNS through AXFR / IXFR.

---

# 23. Zone Transfer

The Primary DNS maintains the authoritative Zone data.

```text
Primary DNS
     │
     │ AXFR / IXFR
     ▼
Secondary DNS
```

## AXFR

Full Zone Transfer.

Used when the complete Zone needs to be synchronized.

## IXFR

Incremental Zone Transfer.

Only changes since the previous serial are transferred.

IXFR is preferred for normal incremental updates.

---

# 24. SOA Serial Number

Every Zone modification requires an SOA serial increment.

Example:

```text
2026082501
```

After a change:

```text
2026082502
```

Example:

```text
example.com. IN SOA ns1.example.com. admin.example.com. (
    2026082502
    3600
    600
    86400
    300
)
```

The Secondary DNS detects the new serial and retrieves the updated Zone.

---

# 25. Zone Validation

Before restarting or reloading BIND9:

```bash
sudo named-checkconf
```

Validate the Zone:

```bash
sudo named-checkzone example.com /etc/bind/db.example.com
```

Expected result:

```text
zone example.com/IN: loaded serial 2026082502
OK
```

---

# 26. DNS Resolution Validation

## Local DNS

```bash
dig @127.0.0.1 start.example.com
```

Expected:

```text
status: NOERROR
```

---

## Primary DNS

```bash
dig @192.168.10.11 start.example.com
```

Expected:

```text
status: NOERROR
```

---

## Secondary DNS

```bash
dig @192.168.20.11 start.example.com
```

Expected:

```text
status: NOERROR
```

---

# 27. SOA Synchronization Test

Check Primary:

```bash
dig @192.168.10.11 example.com SOA +noall +answer
```

Check Secondary:

```bash
dig @192.168.20.11 example.com SOA +noall +answer
```

Expected:

```text
example.com. 300 IN SOA ns1.example.com. admin.example.com. 2026082502 ...
```

The serial numbers must match.

---

# 28. DNS Failover Test

## 28.1 Baseline

Confirm both DNS Servers are working.

```bash
dig @192.168.10.11 start.example.com
dig @192.168.20.11 start.example.com
```

Both should return the expected IP address.

---

# 29. Stop Primary DNS

On the Primary DNS Server:

```bash
sudo systemctl stop bind9
```

Verify:

```bash
sudo systemctl status bind9
```

Expected:

```text
Active: inactive (dead)
```

---

# 30. Test Primary DNS Failure

From the test server:

```bash
dig @192.168.10.11 start.example.com
```

Expected:

```text
connection refused
```

or:

```text
no servers could be reached
```

---

# 31. Test Secondary DNS

```bash
dig @192.168.20.11 start.example.com
```

Expected:

```text
status: NOERROR
```

The expected A record should still be returned.

---

# 32. Test Client Resolver Failover

The client should have both DNS servers configured.

Example:

```text
DNS Servers:
192.168.10.11
192.168.20.11
```

Check:

```bash
resolvectl status
```

or:

```bash
resolvectl dns
```

Then test:

```bash
getent hosts start.example.com
```

Application-level validation:

```bash
curl http://start.example.com
```

---

# 33. Important Failover Behavior

DNS failover is not necessarily instantaneous.

The exact behavior depends on:

- Linux resolver implementation
- `systemd-resolved`
- Application DNS caching
- DNS query timeout
- DNS retry behavior
- DNS server ordering
- Existing DNS cache

Therefore:

```text
Primary DNS failure
        ↓
Resolver detects timeout/failure
        ↓
Secondary DNS query
        ↓
DNS response
```

A direct `dig` test and an application-level test may show different timing.

---

# 34. Resolver Configuration

Example Linux configuration:

```bash
resolvectl status
```

Example:

```text
DNS Servers:
    192.168.10.11
    192.168.20.11
```

The DNS Domain does not need to be set to the Private DNS Zone merely to resolve records in that Zone.

For example:

```text
DNS Domain:
example.internal
```

is not required simply because the DNS server hosts:

```text
example.internal
```

A DNS search domain may be configured if short hostnames need to be resolved.

Example:

```text
app01
```

instead of:

```text
app01.example.internal
```

This is a separate function from selecting the authoritative DNS server.

---

# 35. Security Requirements

DNS traffic should be explicitly controlled.

## SAP PCE → Private DNS

Allow:

```text
UDP/53
TCP/53
```

from approved SAP PCE source networks.

---

## Primary → Secondary

Allow:

```text
TCP/53
```

for Zone Transfer.

Depending on the BIND configuration and environment, DNS query traffic may also require:

```text
UDP/53
TCP/53
```

---

## Private DNS → Route 53

For Option 1, allow DNS traffic from the Private DNS Servers to the Route 53 authoritative DNS infrastructure as required.

The exact destination IPs should not be statically documented because authoritative DNS infrastructure may change.

---

# 36. Do Not Hard-Code Route 53 Authoritative IPs

Route 53 authoritative DNS servers are represented by NS records.

Example:

```bash
dig NS example.com
```

Example output:

```text
example.com. IN NS ns-123.awsdns-xx.com.
example.com. IN NS ns-456.awsdns-yy.net.
example.com. IN NS ns-789.awsdns-zz.org.
example.com. IN NS ns-012.awsdns-aa.co.uk.
```

The authoritative server IP addresses should not be treated as permanent configuration values.

DNS infrastructure can change.

Therefore, network/security policies should be designed carefully rather than permanently depending on manually documented Route 53 IP addresses.

---

# 37. Conditional Forwarding

Conditional forwarding is used when queries for a specific DNS namespace should be sent to a specific upstream DNS Server.

Example:

```text
example.com
        │
        ▼
Specific DNS Forwarder
```

Example BIND configuration:

```conf
zone "example.com" {
    type forward;

    forward only;

    forwarders {
        192.168.30.53;
        192.168.30.54;
    };
};
```

However, the actual upstream architecture depends on whether the target DNS servers are reachable through the required network path.

---

# 38. Important Consideration for Route 53

If the target is a Route 53 **Public Hosted Zone**, the Private DNS Server does not normally forward the query to a single fixed Route 53 IP address.

Instead, the DNS resolution process ultimately reaches the authoritative DNS servers listed in the Zone's NS records.

Therefore, the architecture must consider:

```text
KC Private DNS
       │
       ▼
DNS Resolution Path
       │
       ▼
Route 53 Authoritative DNS
```

Network policy must allow the required DNS traffic.

---

# 39. Application Traffic vs DNS Traffic

DNS resolution and application traffic are separate flows.

Example:

```text
DNS Flow
SAP PCE
   │
   ▼
KC Private DNS
   │
   ▼
Route 53
   │
   ▼
10.10.100.1
```

After DNS resolution:

```text
Application Flow
SAP PCE
   │
   │ HTTP / HTTPS
   ▼
10.10.100.1
```

The DNS Server does not proxy the application traffic.

Its role is only to resolve:

```text
start.example.com
        ↓
10.10.100.1
```

---

# 40. DNS and TLS

DNS and TLS are separate components.

Example:

```text
DNS
start.example.com
        ↓
192.168.10.100
```

Then:

```text
HTTPS
https://start.example.com
        ↓
TLS Certificate Validation
```

If the hostname remains:

```text
start.example.com
```

the TLS certificate must cover:

```text
start.example.com
```

A Private DNS Server does not itself require a TLS certificate simply because it hosts a DNS Zone.

TLS certificates are relevant to the application endpoint using HTTPS.

---

# 41. Operational Guidelines

## Primary DNS

DNS Zone changes should be performed only on the Primary DNS.

Example workflow:

```text
1. Modify Zone
2. Increment SOA serial
3. Validate configuration
4. Validate Zone
5. Reload BIND9
6. Verify Primary
7. Verify Secondary
```

Commands:

```bash
sudo named-checkconf

sudo named-checkzone example.com \
    /etc/bind/db.example.com

sudo rndc reload
```

---

# 42. Secondary DNS

Do not manually modify the synchronized Zone file on the Secondary DNS.

Expected workflow:

```text
Primary
   │
   │ SOA Serial Change
   ▼
NOTIFY
   │
   ▼
Secondary
   │
   │ IXFR / AXFR
   ▼
Updated Zone
```

---

# 43. Monitoring

Recommended monitoring items:

### BIND9 Process

```bash
systemctl is-active bind9
```

### DNS Resolution

```bash
dig @192.168.10.11 start.example.com
dig @192.168.20.11 start.example.com
```

### SOA Serial

```bash
dig @192.168.10.11 example.com SOA +noall +answer

dig @192.168.20.11 example.com SOA +noall +answer
```

### Zone Transfer

Check BIND logs:

```bash
journalctl -u bind9
```

Look for:

```text
Transfer completed
transferred serial
```

---

# 44. Troubleshooting

## 44.1 DNS Server Does Not Respond

Check:

```bash
sudo systemctl status bind9
```

Check listening ports:

```bash
sudo ss -lntup | grep :53
```

Check firewall:

```bash
sudo ufw status
```

---

## 44.2 Zone Returns REFUSED

Check:

```conf
allow-query {
    trusted;
};
```

The client IP must belong to the allowed ACL.

---

## 44.3 Zone Returns NXDOMAIN

Check:

```bash
sudo named-checkzone example.com /etc/bind/db.example.com
```

Then:

```bash
dig @192.168.10.11 example.com SOA
```

Verify the correct Zone is loaded.

---

## 44.4 Secondary Has Old Records

Compare SOA serials:

```bash
dig @192.168.10.11 example.com SOA +noall +answer

dig @192.168.20.11 example.com SOA +noall +answer
```

If the serial numbers differ:

```text
Primary Serial
    >
Secondary Serial
```

check:

- Zone Transfer ACL
- TCP/53 connectivity
- `also-notify`
- `allow-transfer`
- Secondary logs

---

# 45. Common Misconceptions

## Misconception 1

> Public Hosted Zone means records must contain Public IP addresses.

Incorrect.

```text
Public Hosted Zone
        │
        └── A record
             │
             └── Private IP
```

is technically possible.

---

## Misconception 2

> If SAP PCE queries a Private DNS Server, the entire DNS resolution path is private.

Not necessarily.

Example:

```text
SAP PCE
   │
   │ Private
   ▼
KC Private DNS
   │
   │ Public
   ▼
Route 53
```

The first hop is private, but the subsequent DNS resolution may use the Public Network.

---

## Misconception 3

> Secondary DNS automatically replaces the Primary DNS.

Not exactly.

The Secondary DNS provides another authoritative DNS endpoint.

The client resolver determines which configured DNS server to use when one becomes unavailable.

---

# 46. Decision Record

## Decision

The KC environment will provide a highly available Private DNS infrastructure using:

```text
Primary DNS
      +
Secondary DNS
```

The final upstream DNS architecture will be selected after confirming SAP PCE's DNS/network policy.

---

# 47. Architecture Decision

### Option 1 - Preferred if SAP PCE permits Public DNS resolution

```text
SAP PCE
   │
   │ Private
   ▼
KC Private DNS
   │
   │ DNS Forwarding
   ▼
Route 53 Public Hosted Zone
```

Use when:

- Private IP is required only for the SAP PCE Conditional Forwarder
- Public DNS resolution is permitted from KC Private DNS
- Existing DNS namespace should be maintained

---

### Option 2 - Preferred if SAP PCE requires fully private resolution

```text
SAP PCE
   │
   │ Private
   ▼
KC Private DNS
   │
   ▼
Private DNS Zone
```

Use when:

- Entire DNS resolution path must remain private
- Public DNS infrastructure cannot be used
- Separate DNS namespace is acceptable

---

# 48. Open Questions

The following items must be confirmed with SAP PCE.

| Question | Status |
|---|---|
| Conditional Forwarder requires Private IP | To be confirmed |
| Public DNS resolution allowed | To be confirmed |
| DNS resolution must remain fully private | To be confirmed |
| UDP/53 permitted | To be confirmed |
| TCP/53 permitted | To be confirmed |
| KC DNS → Route 53 allowed | To be confirmed |
| Route 53 authoritative DNS traffic permitted | To be confirmed |
| Dedicated Private Zone acceptable | To be confirmed |
| Separate SAP PCE namespace acceptable | To be confirmed |

---

# 49. Final Architecture Selection

The final decision should follow this process:

```text
                    SAP PCE DNS Requirement
                              │
                              ▼
              ┌────────────────────────────┐
              │ Conditional Forwarder      │
              │ requires Private DNS IP?   │
              └──────────────┬─────────────┘
                             │
                            YES
                             │
                             ▼
              ┌────────────────────────────┐
              │ Must entire DNS resolution │
              │ path remain private?       │
              └──────────────┬─────────────┘
                             │
                 ┌───────────┴───────────┐
                 │                       │
                YES                      NO
                 │                       │
                 ▼                       ▼
          ┌─────────────┐          ┌─────────────┐
          │  OPTION 2   │          │  OPTION 1   │
          │             │          │             │
          │ Private DNS │          │ Route 53    │
          │ Zone        │          │ Public Zone │
          └─────────────┘          └─────────────┘
```

---

# 50. Summary

The KC Private DNS architecture provides a private DNS entry point for SAP PCE while supporting High Availability through a Primary / Secondary BIND9 configuration.

Two DNS resolution models are available.

### Option 1

```text
SAP PCE
   ↓
KC Private DNS
   ↓
Route 53 Public Hosted Zone
   ↓
Private IP Record
```

Advantages:

- Existing DNS namespace maintained
- Minimal DNS record duplication
- Lower operational complexity

Constraint:

- Depends on SAP PCE and network policy allowing the DNS resolution path to reach Public DNS infrastructure

---

### Option 2

```text
SAP PCE
   ↓
KC Private DNS
   ↓
Dedicated Private DNS Zone
   ↓
Private IP Record
```

Advantages:

- Fully private DNS resolution
- No dependency on Public DNS infrastructure for SAP PCE
- Clear separation of SAP PCE DNS

Constraints:

- Additional Zone management
- Potential DNS record duplication
- Potentially different service namespaces

---

# 51. Recommended Next Step

Before implementing the final architecture, confirm the following requirement with SAP PCE:

> **Does SAP PCE require only the Conditional Forwarder destination to be reachable through a Private IP, or must the entire DNS resolution path remain within the Private Network?**

This single requirement determines whether **Option 1** or **Option 2** is the appropriate architecture.

Once this requirement is confirmed, the selected architecture can be finalized and the corresponding DNS, firewall, routing, and operational configurations can be applied.