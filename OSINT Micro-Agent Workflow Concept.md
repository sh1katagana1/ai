# OSINT Micro-Agent Workflow

***

## Goal
To build an OSINT engine that utilizes AI and micro-agent design, similar to what Jason Haddix uses.

## Layout
```
                   ┌───────────────┐
                   │ Case Manager  │
                   └───────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼

   Domain Agent      Person Agent       Infrastructure Agent

        ▼                  ▼                  ▼

   DNS/WHOIS         Social/LinkedIn      IP/CIDR/ASN
   Subdomains        Breaches             Hosting

        └──────────────┬──────────────────┘
                       ▼

              Correlation Agent

                       ▼

                 Intel Analyst

                       ▼

                 Report Agent
```

The concept revolves around:
1. One agent investigates domains.
2. One investigates people.
3. One investigates infrastructure.
4. One investigates malware.
5. One investigates threat actor chatter.
6. Another agent compares all findings.

## Example OSINT Investigation
Let's say your SOC gives you an intelligence requirement to investigate this domain:
```
suspicious-domain-example.com
```
### Agent 1: Scope Agent
Determines:
1. Is this a domain?
2. IP?
3. Email?
4. Person?
5. Company?

Output:
```
{
  "type": "domain",
  "priority": "high"
}
```

### Agent 2: Domain Agent
Tasks:
1. WHOIS
2. DNS
3. Subdomains
4. Certificate transparency

Output:
```
{
  "registrar": "...",
  "creation_date": "...",
  "subdomains": [...]
}
```

### Agent 3: Infrastructure Agent
Tasks:
1. Resolve IPs
2. ASN lookup
3. Hosting provider
4. Neighbor domains

Output:
```
{
  "asn": "AS13335",
  "provider": "Cloudflare",
  "related_domains": [...]
}
```

### Agent 4: Reputation Agent
Checks:
1. Abuse feeds
2. Threat feeds
3. Malware references

Outputs:
```
{
  "malicious_score": 87,
  "sources": [...]
}
```

### Agent 5: Correlation Agent
This is where AI shines. Prompt:
```
You are a senior threat intelligence analyst.

Review the outputs from:
- Domain Agent
- Infrastructure Agent
- Reputation Agent

Identify:

1. Relationships
2. Suspicious indicators
3. Investigation gaps
4. Recommended next actions
```
This agent doesn't gather data. It thinks.

### Agent 6: Gap Hunter Agent
One of the most valuable agents. Prompt:
```
What information is missing?

What additional OSINT would increase confidence?
```
Output:
```
- Search GitHub
- Check Telegram mentions
- Search breach datasets
- Investigate ASN neighbors
```

Many investigators stop too early. This agent prevents that.

## Folder Structure
```
threat-swarm/

agents/

    scope_agent.py
    domain_agent.py
    infra_agent.py
    reputation_agent.py
    social_agent.py
    malware_agent.py
    correlation_agent.py
    report_agent.py

workflows/

    phishing.py
    ransomware.py
    vendor_risk.py

memory/

    investigations/
    entities/

outputs/

    reports/
```

## Hypothesis Agent
Based on me being a Cyber Threat Intel Analyst, I should include something like this. 

Input:
```
IOC package
```
Prompt:
```
Based on the collected evidence:

Generate:

- Top 3 hypotheses
- Supporting evidence
- Contradicting evidence
- Confidence score
```
Output:
```
Hypothesis #1:
Credential harvesting campaign

Confidence: 78%

Evidence:
- Newly registered domain
- Microsoft spoofing
- Shared infrastructure with prior campaigns
```


















































