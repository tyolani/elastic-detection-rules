# Azure Entra Unusual Failed Authentication Attempts Behind Rare User Agents

---

## Metadata

- **Author:** Elastic
- **Description:** This hunting query gathers evidence of failed authentication attempts in Azure Entra ID where unusual user agents are used. Adversaries may use tools like FastHTTP to conduct brute force attacks against Azure Entra ID user accounts. FastHTTP is a high-performance HTTP client optimized for speed and efficiency, making it a popular choice for password spraying attacks. By identifying failed authentication attempts behind rare user agents, security teams can detect and respond to unauthorized access attempts in Azure Entra ID.

- **UUID:** `3f26f262-fe14-11ef-9ee5-f661ea17fbcd`
- **Integration:** [azure](https://docs.elastic.co/integrations/azure)
- **Language:** `[ES|QL]`
- **Source File:** [Azure Entra Unusual Failed Authentication Attempts Behind Rare User Agents](../queries/entra_authentication_attempts_behind_rare_user_agents.toml)

## Query

```sql
from logs-azure.signinlogs*
| where @timestamp > now() - 14 day
| keep
    @timestamp,
    event.dataset,
    event.category,
    azure.signinlogs.properties.authentication_requirement,
    azure.signinlogs.properties.status.error_code,
    azure.signinlogs.properties.app_id,
    source.as.organization.name,
    user_agent.original,
    azure.signinlogs.category,
    event.outcome,
    azure.signinlogs.properties.user_principal_name,
    source.ip
| WHERE
    event.dataset == "azure.signinlogs"
    and event.category == "authentication"
    and source.as.organization.name != "MICROSOFT-CORP-MSN-AS-BLOCK"
    and event.outcome != "success"
    and azure.signinlogs.properties.status.error_code in (50053, 50126, 50055, 50056, 50064, 50144)
    and (
        to_lower(user_agent.original) LIKE "%go-http-client/1.1%" or
        to_lower(user_agent.original) LIKE "%fasthttp%" or
        to_lower(user_agent.original) LIKE "%python%" or
        to_lower(user_agent.original) LIKE "%curl%" or
        to_lower(user_agent.original) LIKE "%wget%" or
        to_lower(user_agent.original) LIKE "%httpclient%" or
        to_lower(user_agent.original) LIKE "%postmanruntime%" or
        to_lower(user_agent.original) LIKE "%headlesschrome%" or
        to_lower(user_agent.original) LIKE "%phantomjs%" or
        to_lower(user_agent.original) LIKE "%selenium%" or
        to_lower(user_agent.original) LIKE "%okhttp%" or
        to_lower(user_agent.original) LIKE "%scrapy%" or
        to_lower(user_agent.original) LIKE "%hydra%" or
        to_lower(user_agent.original) LIKE "%patator%" or
        to_lower(user_agent.original) LIKE "%metasploit%" or
        to_lower(user_agent.original) LIKE "%curio%" or
        to_lower(user_agent.original) LIKE "%hyper%" or
        to_lower(user_agent.original) LIKE "%kali%" or
        to_lower(user_agent.original) LIKE "%hydra%" or
    )
// count the number of unique user login attempts
| stats
    unique_user_app_login_count = count(*) by
        azure.signinlogs.properties.user_principal_name,
        azure.signinlogs.properties.authentication_requirement,
        azure.signinlogs.properties.app_id
| sort unique_user_app_login_count asc
```

## Notes

- Review `azure.signinlogs.properties.authentication_protocol` to verify the authentication method used. Non-interactive SFA is typically reserved for automated processes or legacy authentication methods.
- Review `azure.signinlogs.properties.error_code` to identify the specific error codes associated with the failed authentication attempts. Common error codes include `50053` for account lockouts, `50126` for invalid credentials, and `50055` for expired passwords.
- Investigate `azure.signinlogs.properties.user_principal_name` to determine whether the user typically authenticates using SFA. Unusual use by regular accounts may indicate compromise.
- Analyze `source.as.organization.name` to determine if the request originated from a known hosting provider, VPN, or anonymization service that is unexpected in your environment.
- Examine `source.address` to check if the IP address is associated with previous suspicious activity, high-risk geolocations, or known threat infrastructure.
- Pivot on `azure.signinlogs.properties.user_principal_name` to identify any other high-risk activities within the same session.
- Correlate findings with `azure.signinlogs.properties.authentication_processing_details` to identify possible legacy protocol usage, token replay, permission scopes or bypass mechanisms.
- Review `user_agent.original` to identify the user agent used in the authentication request. Determine if this user agent is expected in your environment or if it is associated with known malicious activity.

## MITRE ATT&CK Techniques

- [T1078.004](https://attack.mitre.org/techniques/T1078/004)
- [T1110.003](https://attack.mitre.org/techniques/T1110/003)

## References

- https://securityscorecard.com/wp-content/uploads/2025/02/MassiveBotnet-Report_022125_03.pdf

## License

- `Elastic License v2`
