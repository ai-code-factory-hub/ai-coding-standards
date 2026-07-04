# Runbook · Security Incident

> Fires on suspected compromise — leaked credentials, unauthorized access, malware, data exfiltration, or active exploitation.

Template: [../runbook-template.md](../runbook-template.md) · Index: [../README.md](../README.md)

> **First rule: preserve evidence before you change anything. Don't tip off the attacker. Don't power off — capture volatile state first.**

## Alert / Symptoms
- IDS/WAF/anomaly alert; auth anomalies (impossible-travel logins, brute force, privilege escalation).
- Secret detected in a public repo, log, or leak feed; abnormal API-key usage.
- Unexpected outbound traffic / data egress; unknown processes, binaries, or cron entries.
- Ransomware indicators, defaced content, or a credible external report / bug-bounty submission.

## Impact
- **Users:** potential exposure or theft of data, account takeover, service disruption.
- **Severity:** treat as **SEV-1 / high** by default until scoped down. A confirmed breach of personal/regulated data triggers legal/regulatory obligations — see Escalation.

## Quick checks
1. **Confirm it's real** (not a false positive) — but err toward treating it as real; over-response is cheaper than a missed breach.
2. **Scope:** which systems, accounts, credentials, and data are implicated? What did the actor touch?
3. **Still active?** Live session, ongoing egress, or historical artifact?
4. **Entry vector (initial hypothesis):** leaked credential, vulnerable dependency, exposed endpoint, phishing, insider.
5. **Data at risk:** does the affected system hold personal / regulated / secret data? This drives notification duties.

## Diagnosis — follow the phases (Contain → Eradicate → Recover)
Run diagnosis and containment together; do not wait for perfect attribution.

1. **Preserve evidence (before mutating anything):** snapshot affected hosts/volumes, export the relevant logs (auth, access, network, audit trail) to a secure, immutable location, and record a timeline. Note timestamps and who did what. Chain of custody matters for legal/forensic use.
2. **Determine the vector & blast radius** from logs: which credentials/sessions/keys were used, from where, to do what; lateral movement; what data was read/exfiltrated.
3. **Assess data impact:** confirm whether regulated/personal data was accessed or exfiltrated — this is the trigger question for breach notification.

## Mitigation
_Contain first (stop the bleeding), then eradicate, then recover. Containment steps here are urgent, not "reversible-first" — but avoid destroying evidence._

**Contain**
1. **Revoke/rotate compromised credentials and secrets** immediately (API keys, tokens, DB passwords, signing keys).
2. **Invalidate active sessions** for affected accounts; force re-authentication / reset.
3. **Isolate affected hosts** — network-quarantine (not power-off; keep volatile evidence) so the attacker can't move laterally or exfiltrate more.
4. **Block the attacker's access path** — revoke keys, tighten firewall/WAF rules, disable the exposed endpoint or compromised account.

**Eradicate**
5. **Remove the foothold:** kill malicious processes/cron/webshells, patch the exploited vulnerability, remove injected artifacts.
6. **Rotate everything the compromised system could reach** (assume secrets it held are burned).

**Recover**
7. **Rebuild from known-good** images/backups rather than cleaning in place, where feasible; verify integrity before restoring to service.
8. **Restore service** and monitor closely for re-compromise; keep heightened logging.

## Escalation
- **Immediately engage the security/incident-response lead and CISO** — security incidents are not handled solo.
- **Trigger breach-notification assessment with Legal/DPO** the moment regulated/personal data exposure is plausible. Notification clocks (e.g. **72-hour** regimes) start early — do not sit on it. See [../../compliance-kb/README.md](../../compliance-kb/README.md).
- Notify affected customers / regulators as legal counsel directs.
- Involve **law enforcement** for criminal activity per your legal team's guidance.

## Prevention / follow-up
- Complete a formal, blameless post-incident review; track remediation to closure.
- Enforce secret scanning, least-privilege, MFA, key rotation, and dependency patching as standard.
- Rehearse this runbook (tabletop / IR drills); keep the on-call security contact tree current.
- **Ties to:** [../../standards-kb/04-security.md](../../standards-kb/04-security.md), [../../standards-kb/09-compliance-privacy.md](../../standards-kb/09-compliance-privacy.md), and [../../compliance-kb/README.md](../../compliance-kb/README.md).
