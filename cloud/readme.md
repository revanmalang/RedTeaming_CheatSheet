# Pentesting the cloud cheatsheet

## Index
* [General](#General)
  * [Scaning tools](#Scanning-tools)
* [Recon \ OSINT](recon.md)
* [Initial access attacks](initial-access-attacks.md)
* [Cloud Services](readme.md)
  * [Azure](azure/readme.md)
  * [Amazon Web Services](aws/readme.md)
  * [Google Cloud Platform](gcb/readme.md)
  

## General
- Google Cloud Platform != Google Workspace
- Azure != Microsoft 365
- Google Workspace and Microsoft 365 are productivity suites
- Rules of engagement 
  - Azure https://www.microsoft.com/en-us/msrc/pentest-rules-of-engagement
  - AWS: https://aws.amazon.com/security/penetration-testing/
  - GCP  https://support.google.com/cloud/answer/6262505?hl=en
- Enumerate host https://github.com/dafthack/HostRecon

## Scanning tools
### Enumeration
- WeirdAAL
  - https://github.com/carnal0wnage/weirdAAL
  - AWS

### Vulnerability scanning
- Scoutsuite
  - https://github.com/nccgroup/ScoutSuite
  - Scans AWS, Azure, GCP, Alibaba cloud, Oracle cloud
- Scoutsploit
  - https://github.com/cloudsploit/scans
  - Scans AWS, Azure, GCP, Oracle

### Privesc scanning
- GCP IAM privilege esalation
  - https://github.com/RhinoSecurityLabs/GCP-IAM-Privilege-Escalation/tree/master/PrivEscScanner
  - GCP
- PACU
  - https://github.com/RhinoSecurityLabs/pacu
  - AWS 
- Stormspotter
  - https://github.com/Azure/Stormspotter
  - Azure
- Skyark
  - https://github.com/cyberark/SkyArk
