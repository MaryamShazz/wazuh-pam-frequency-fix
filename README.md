# Wazuh PAM Frequency Fix

A security engineering case study for [Wazuh](https://github.com/wazuh/wazuh) issue [#16252](https://github.com/wazuh/wazuh/issues/16252): adding a `same_user` PAM correlation rule so repeated local authentication failures get detected even when there is no source IP to correlate on.

## Status

Submitted, awaiting maintainer review, not yet merged.

- Issue: wazuh/wazuh [#16252](https://github.com/wazuh/wazuh/issues/16252)
- Pull request: wazuh/wazuh [#38270](https://github.com/wazuh/wazuh/pull/38270)
- Target branch: `4.14.9`
- Earlier pull request [#38154](https://github.com/wazuh/wazuh/pull/38154) was closed after I found it had been opened against the wrong branch, see below.

This repository is a clean write-up of that contribution. My actual fork of Wazuh, with the full commit history, lives separately.

## The problem

Wazuh's rule 5551 detects repeated PAM authentication failures by correlating events on `<same_source_ip />`. That works for something like SSH, where the PAM decoder can read a client address out of the log line's `rhost` field.

Local authentication, `su` or `login` for example, doesn't have a remote host at all, so `rhost` comes through empty and the decoder never extracts a source IP for those events. With no IP to compare, `same_source_ip` can never be satisfied, so rule 5551 stays silent no matter how many times someone brute forces `su` on the same box. That's a real detection gap, and a fairly common one to hit, since local privilege escalation attempts are exactly the kind of thing that shows up after an attacker already has a low privilege shell.

## The fix

Added rule 5558, which correlates on `<same_user />` instead of source IP.

```xml
<rule id="5558" level="10" frequency="8" timeframe="180">
    <if_matched_sid>5503</if_matched_sid>
    <same_user />
    <description>PAM: Multiple failed logins for the same user in a small period of time.</description>
    <mitre>
        <id>T1110</id>
    </mitre>
    <group>authentication_failures,pci_dss_10.2.4,pci_dss_10.2.5,pci_dss_11.4,gpg13_7.8,gdpr_IV_35.7.d,gdpr_IV_32.2,hipaa_164.312.b,nist_800_53_AU.14,nist_800_53_AC.7,nist_800_53_SI.4,tsc_CC6.1,tsc_CC6.8,tsc_CC7.2,tsc_CC7.3,</group>
</rule>
```

| Property | Value |
|---|---|
| Rule | 5558 |
| Parent rule | 5503 |
| Correlation | `same_user` |
| Frequency | 8 |
| Timeframe | 180 seconds |
| Level | 10 |
| MITRE | T1110 |
| Change type | additive, rule 5551 is untouched |

This isn't a new pattern I invented. A few other rule files in the same ruleset, the Windows authentication rules, the Serv U FTP rules, and the RSA Auth Manager rules, already fall back to `same_user` for services where an IP isn't reliably available. I just applied the same approach to PAM.

## Detection flow

```
PAM authentication failure
          |
          v
      Rule 5503
          |
          v
   Is a source IP available?
       /          \
     yes           no
      |             |
      v             v
 Rule 5551       Rule 5558
same_source_ip   same_user
      |             |
      +------+------+
             |
             v
       Repeated failures
             |
             v
           Alert
```

## Validation

Static checks, run before anything was pushed:

| Check | Result |
|---|---|
| XML parsing | pass |
| Rule 5558 uniqueness | pass |
| `git diff --check` | pass |
| Final diff scope | one file, 11 lines added |

Runtime validation: a Wazuh manager was run from the `wazuh/wazuh-manager:4.14.6` image, the modified ruleset was copied into the container, and the rule was tested with `wazuh-logtest`. Seven repeated local PAM authentication failures, all missing a source IP, matched only the base rule 5503. On the eighth event inside the 180 second window, rule 5558 fired at level 10 with MITRE technique T1110.

![Rule 5558 firing on the eighth repeated local PAM authentication failure](./evidence/04-logtest-rule-5558-fired.png)

See [`evidence/`](./evidence) for the rest of the output, and [`evidence/EVIDENCE_NOTES.md`](./evidence/EVIDENCE_NOTES.md) for which files are real screenshots. One gap is still open there, the duplicate rule warning only exists as terminal text right now.

## What I actually ran into

I opened the first pull request, #38154, against `wazuh:main`. While double checking the diff against the real upstream repository afterward, a rebase attempt failed with a modify or delete conflict, which turned out to mean the legacy `ruleset/rules/` directory this fix lives in no longer exists on `main` at all, Wazuh has been migrating the classic XML ruleset out as part of a larger rework. That pull request could never have merged as opened. I rebased the work onto `4.14.9`, the maintenance branch where the file is still present, cleaned up an accidental duplicate rule I'd introduced earlier by running `git am` twice on the same patch, and opened #38270 against the correct branch. I then closed #38154 myself with a comment explaining why, rather than leaving two open pull requests for the same issue.

The full account of that, including the mistakes, is in the report below.

## Files changed

| | |
|---|---|
| File | `ruleset/rules/0085-pam_rules.xml` |
| Lines added | 11 |
| Existing rules modified | none |
| New rule | 5558 |

## Detailed report

[Wazuh PAM Frequency Fix Report (PDF)](./report/Wazuh_PAM_Frequency_Fix_Report.pdf) covers the whole process end to end: diagnosing the bug from the decoder source, setting up the workspace, the Docker and wazuh-logtest validation, the duplicate rule mistake and how I found it, discovering and fixing the wrong target branch, and both pull requests.

## Links

- Pull request: https://github.com/wazuh/wazuh/pull/38270
- Closed pull request: https://github.com/wazuh/wazuh/pull/38154
- Issue: https://github.com/wazuh/wazuh/issues/16252

## License

[MIT](./LICENSE) for the contents of this repository. Wazuh itself is licensed separately under GPLv2.
