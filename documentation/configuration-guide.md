# Project Sentinel: Configuration Guide

## 1. Overview

Project Sentinel is a hands-on Security Operations Center (SOC) lab built with Wazuh. The project focused on centralized security monitoring, Windows event collection, custom detection rules, event archiving, and File Integrity Monitoring (FIM).

This guide documents the main configurations used in the lab to help others understand the setup and reproduce the relevant steps.

## 2. Wazuh Event Archiving

**Configuration file:** `/var/ossec/etc/ossec.conf`

Event archiving was enabled by configuring the following settings inside the existing `<global>` section:

```xml
<logall>yes</logall>
<logall_json>yes</logall_json>
```

### Purpose

- `logall`: Enables archiving of events received by the Wazuh manager.
- `logall_json`: Enables archiving of events in JSON format.

These settings allow analysts to investigate collected events that may not have generated security alerts.

After updating the configuration, the Wazuh manager was restarted.

**Note:** Archiving can consume significant disk space, so storage and retention should be considered.

## 3. Filebeat Archive Configuration

**Configuration file:** `/etc/filebeat/filebeat.yml`

The Wazuh archives module was enabled so archived events could be indexed and viewed through the dashboard.

Relevant configuration:

```yaml
filebeat.modules:
  - module: wazuh
    alerts:
      enabled: true
    archives:
      enabled: true
```

This is a reference excerpt, not a complete replacement for `filebeat.yml`. The existing output, TLS certificates, authentication, and other settings must be preserved.

After making the changes, Filebeat was restarted.

## 4. Wazuh Archives Dashboard

An index pattern was created in the Wazuh dashboard using:

```text
wazuh-archives-*
```

The `timestamp` field was selected as the time field.

This allowed archived events to be explored through the dashboard rather than relying only on alert data.

## 5. Windows Sysmon Event Collection

**Configuration file:** Windows Wazuh agent `ossec.conf`

The following configuration was added to collect events from the Sysmon operational event channel:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

### Purpose

Sysmon generates detailed Windows system activity telemetry. The Wazuh agent collects events from the configured event channel and forwards them to the Wazuh manager for analysis.

Sysmon must be installed and generating events for this configuration to provide useful telemetry.

## 6. Custom Windows Guest Account Detection

**Reference file:** `detection-rules/windows-guest-account.xml`

The lab included a custom Wazuh rule intended to detect Windows Guest account enablement.

The rule uses Windows Security Event ID `4722` and checks whether the target account is named `Guest`.

The intended detection logic is:

1. Receive a Windows security event.
2. Check whether the event represents an account being enabled.
3. Check whether the target username is `Guest`.
4. Generate an alert when the configured conditions match.

Event ID `4725` represents an account being disabled and can be used in a separate rule.

**Validation note:** Confirm the parent rule ID and decoded event fields against the installed Wazuh version. Only report the disablement rule as implemented if it was added and tested.

## 7. File Integrity Monitoring

File Integrity Monitoring was configured to monitor selected files on the Windows and Ubuntu endpoints.

FIM helps identify changes to monitored files, including modifications that may warrant investigation.

During the lab, a change to the monitored Ubuntu file `test.txt` produced a modification event in the Wazuh dashboard.

The observed event included the description `Integrity checksum changed.`

This demonstrated how file changes can be surfaced for security investigation.

## 8. Validation and Troubleshooting

During configuration and testing, the following checks were important:

- Confirm that endpoint agents were connected to the Wazuh manager.
- Verify that Sysmon was installed and producing events.
- Check that configuration changes were saved before restarting services.
- Confirm that archive events appeared under the correct index pattern.
- Review custom rule matches and generated alerts.
- Check agent and manager logs when expected events were missing.

Custom Wazuh rules should be tested with the `wazuh-logtest` utility and validated against real events before being considered reliable.

## 9. Security Considerations

- Do not publish passwords, private keys, authentication tokens, or sensitive internal details.
- Keep custom rules in the Wazuh configuration directory rather than modifying the built-in ruleset directly.
- Monitor disk usage when event archiving is enabled.
- Test detection rules before using them in a production environment.
- Treat an alert as an investigation lead, not automatic proof of malicious activity.

## 10. Conclusion

Project Sentinel provided practical experience configuring a Wazuh monitoring environment, collecting endpoint telemetry, enabling event archiving, creating custom detection logic, and monitoring file integrity.

The lab strengthened my understanding of how endpoint data moves through a SIEM workflow and how security events can be used to support investigation and detection.

**Documentation status:** This guide describes the lab configuration and observed results. Any additional detection rules or tests should be documented only after they have been implemented and verified.
