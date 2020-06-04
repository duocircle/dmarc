# DMARC how-to
The inspiration for this document is based on a how-to created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)).

# Table of contents
- [What is DMARC?](#what-is-dmarc-)
- [Why use DMARC?](#why-use-dmarc-)
- [Tips, tricks and notices for implementation](#tips--tricks-and-notices-for-implementation)
- [Overview of DMARC configuration tags](#overview-of-dmarc-configuration-tags)


# What is DMARC?
DMARC is short for **D**omain based **M**essage **A**uthentication, **R**eporting and **C**onformance and is described in [RFC 7489](https://tools.ietf.org/html/rfc7489). With DMARC the owner of a domain can, by means of a DNS record, publish a  policy that states how to handle e-mail (deliver, quarantine, reject) which is not properly authenticated using SPF and/or DKIM. Because DMARC depends on the security of DNS, the use of DNSSEC is highly recommended. 

At the same time DMARC also provides the means for receiving reports which allows a domain's administrator to detect whether their domainname is used for phishing or spam. 
 
# Why use DMARC?
Before DMARC, organizations already took several measures to determine the authenticity of an e-mail (like SPF and DKIM) to reduce the received amount of SPAM to a minimum. This is basically a good thing, but if these measures fail to choose whether or not an email is SPAM with a high level of certainty, the choice is redirected to the addressee (receiving party). This methodology is prone to abuse, since users are generally not equiped with the knowledge and/or means to classify incoming emails. 
 
DMARC addresses this problem and enables the owner of a domain to take explicit responsiblity with regard to the actions taken by the sending party when the validity of an incoming email cannot be determined.  

# Tips, tricks and notices for implementation
* Interoperabily issues: https://tools.ietf.org/html/rfc7960
* DMARC does not require both DKIM or SPF. But implementation of both is strongly advised.
* DMARC is about aligning the domain in the DKIM header (the "d=" value) and SPF header (*a.k.a. RFC5321.MailFrom, Return-Path, Envelope Sender, Envelope From*) with the organizational domain in the message From header (*a.k.a. RFC5322.From, Header From, Message From*) which is visible to the user.
  * If these values do not align this could mean for example, that an attacker placed a valid DKIM signature header in an email with a "d=" value that points to a domain the attacker controls, allowing DKIM to pass while still spoofing the From address to the user.
* Parked domain: “DMARC p=reject”. Make sure to include rua and ruf addresses, since this allows monitoring of possible abuse attempts. Implement additional records (SPF, DKIM, NullMX) if possible, see also our [Parked domain how-to](https://github.com/internetstandards/toolbox-wiki/blob/master/parked-domain-how-to.md).
* RFC 7489 [states](https://tools.ietf.org/html/rfc7489#section-6.4) that the tags dmarc-version ("v=") and dmarc-request ("p=") should be on the first and second position of the DMARC record. The order of the other tags does not matter: "components other than dmarc-version and dmarc-request may appear in any order".
* The verified [erratum 5440 of RFC 7489](https://www.rfc-editor.org/errata_search.php?rfc=7489) states that a semicolon should be included in the DMARC version tag. Correct: "v=DMARC1;". Incorrect: "v=DMARC1". 
* When using office 365, the forwarding of calendar appointments from a DMARC projected domain fails. This is a known issue. Read more on the [Office 365 UserVoice forum](https://office365.uservoice.com/forums/264636-general/suggestions/34012756-forwarding-of-calendar-appointments-from-a-dmarc-p) and don't forget to submit your vote! 
  * There is a workaround: Forward the appointment as an "iCalendar file" or as an attachment. 
* When processing incoming mail we advise to favor a DMARC policy over an SPF policy. Do not configure SPF rejection to go into effect early in handling, but take full advantage of the enhancements DMARC is offering. A message might still pass based on DKIM.
  * At the same time, be aware that some operaters still allow a hard fail (-all) to go into effect early in handling and skip DMARC operations. 
* When using a different domain for the rua and/or ruf address, make sure that the DMARC authorization records (example.nl._report._dmarc.differentdomain.nl) are properly DNSSEC signed. A regularly occuring mistake is the presence of "proof of non-existence" (NSEC3) for the ancestor domain (_report._dmarc.differentdomain.nl). If this happens then resolvers that use qname minimization (like the resolver used by [Internet.nl](https://internet.nl)) think that example.nl._report._dmarc.differentdomain.nl does not exists since _report._dmarc.differentdomain.nl does not exists. Therefore the resolver can't get the DMARC authorization record which makes DMARC fail. 
  * Check your DNSSEC implementation on [DNSViz](https://dnsviz.net/). Enter "example.nl._report._dmarc.differentdomain.nl".
  * You can also manually check for this error. `dig example.nl._report._dmarc.differentdomain.nl +dnssec` results in a NOERROR response, while `dig _report._dmarc.differentdomain.nl +dnssec` results in a NXDOMAIN response. 
  
# Overview of DMARC configuration tags
The DMARC policy is published by means of a DNS TXT record. A DMARC record can contain several configuration tags. The table below will list all configuration tags and explain their purpose.

| DMARC configuration tag | Required? | Value(s) | Explanation |
| ---  | --- |  --- | --- |
| v | mandatory | DMARC1; | |
| p | mandatory | none<br>quarantine<br>reject | None: don't do anything if DMARC verification fails (used for testing)<br>quarantine: treat mail that fails DMARC check as suspicious<br>reject: reject mail that fail DMARC check |
| rua | optional | rua@example.nl | This field contains the email address used to send **aggregate** reports to |
| ruf | optional |ruf@example.nl | This field contains the email address used to send **forensic** reports to |
| fo | mandatory | <br>0<br>1<br>s<br>d | Reporting options for failure reports. Generates a report if:<br>- both SPF and DKIM tests fail (0)<br>- either SPF or DKIM test fail (1)<br>- SPF test fails (s)<br>- DKIM test fails (d) |
| adkim | optional | s<br>r | Controls how strict the result of DKIM alignment checks should be intepreted. Strict or relaxed. |
| aspf | optional | s<br>r | Controls how strict the result of SPF alignment checks should be intepreted. Strict or relaxed. |
| pct | optional | 0..100 | Determine percentage of mail from your domain to have the DMARC verificaton done by other mail providers. Default is 100. |
| rf | optional | afrf<br>aodef | Preferred reporting format for forensic reports. Default is afrf. |
| ri | optional | 60..86400 | Interval (in seconds) at which you want to receive DMARC reports. The DMARC RFC specifies that organizations should be able to send reports at least once every day (86400 seconds) and also states a minimum interval of one hour (60 seconds). Default is 86400. |
| sp | optional | none<br>quarantine<br>reject |  Subdomain policy. If not set the policy defined under "p=" will apply to all your subdomains. |

Be aware that implementing a DMARC record without a rua configuration is possible, this is not advised because the DMARC XML files that are received by implementing a rua email address can help with implementing DKIM or SPF to meet the DMARC requirements.
