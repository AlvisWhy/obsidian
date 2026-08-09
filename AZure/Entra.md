Entra ID directory
Add new user/ invite guest user

Security Defaults
Required for MFA for tenant by default

Roles:
Owner, Contributor, Reader

Authority level
Management group, subscription, resource group

Managed Identity
System assigned managed identity
/User-assigned managed identity

Register app in the Azure directory. Can use AD for auth. Like IAP,
could define certain AD users or all entra ID users to use the app

---------------------------------------------
For OUTH2
the resource server must register itself in the authorization server

authorization callback URL is the callback address. After auth, a code will be returned with the callback url. with the code sent to token endpoint, token will be required.

The client ID and Client Secret will be used when asking auth server to identify from which application the auth requirement come.
client ID and client secret are used for token exchange.

After that, app use the access token to ask the auth organization to get user info

-----------------------------------------------------

Azure AD B2C (identity service)
Can use various 3 party identity provider
Can be used in business apps

sync Cloud AD and on-prem AD

1 password Hash Sync: password are copied to Azure AD
2 Pass-through: password still in on-prem, Azure AD pass data to on-prem to validate

-----------------------------------------------

Monitor:
Action group: notification + actions(service bus, event grid)

Export logs to the log analytics workspace. Then could use query language in the log analytics

Diagnostic setting (log sink to the analytics workspace)

insights
Application insight -> availability test
resource tag(key:val)

Label in GCP: key:val
Tags in GCP: string

AZure KeyVault,
store secret, certification, or key. Should enable identity management for using the keys

-----------------------------------------------

AZure policy

definition, Initiative(group of definition),  assignment(assign initiative to certain scope).