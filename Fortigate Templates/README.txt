README - FortiGate Combined HTTP SNMP Custom Template for Zabbix 7.2
Created By Roee Golan

Privacy note
------------
All IP addresses, usernames, object names, passwords, and tokens in this README are fictional examples only.
Replace them with your real production values only inside your FortiGate and Zabbix systems.
Do not store real API tokens, SNMP passwords, or production IP addresses in this README.

Final YAML file
---------------
Use this final import file:

FortiGate_Combined_HTTP_SNMP_Custom_Zabbix_7_2_Created_by_Roee_Golan.yaml

Final Zabbix template technical name:
FortiGate Combined HTTP SNMP Custom

Reason:
Zabbix template host names cannot safely use special characters such as "+" during import.

Purpose
-------
This guide explains how to prepare a FortiGate firewall and a Zabbix 7.2 host for a combined monitoring setup based on:
1. FortiGate by HTTP / REST API - primary monitoring method.
2. FortiGate by SNMPv3 - supplemental monitoring method, mainly for interfaces and traffic counters.

Recommended design
------------------
HTTP/API should be the primary source for:
- System status
- CPU
- Memory
- Firmware / version
- Serial / model
- HA status
- License information
- FortiGuard / service status
- VPN / API-based monitoring data, where available

SNMPv3 should be used mainly for:
- Interfaces
- Traffic in/out
- Interface operational status
- Interface errors/discards
- Counters that are better exposed through SNMP

Important note
--------------
Do not link the original "FortiGate by HTTP" and "FortiGate by SNMP" templates together on the same host without adjustment.
They may contain duplicate graph, item, trigger, and discovery names.
The combined custom template should remove or rename duplicates, with priority given to "FortiGate by HTTP".

====================================================================
PART 1 - FortiGate REST API configuration
====================================================================

Step 1 - Create or verify an Admin Profile
------------------------------------------
In FortiGate GUI, go to:

System > Admin Profiles

Create a dedicated read-only profile, for example:

demo_monitor_profile

Recommended permissions:
- System: Read
- Network: Read
- Log & Report: Read
- Security Fabric: Read
- VPN: Read, if VPN monitoring is needed
- FortiView / Monitor sections: Read, if available in your FortiOS version

If a built-in read-only profile already exists, such as:

super_admin_readonly

it can be used, as long as it allows the required API endpoints.

Step 2 - Create REST API Admin
------------------------------
Go to:

System > Administrators > Create New > REST API Admin

Recommended values:

Username:
demo_api_user

Comments:
Zabbix FortiGate by HTTP monitoring

Administrator profile:
super_admin_readonly
or:
demo_monitor_profile

PKI Group:
Disabled, unless your environment requires certificate-based authentication.

CORS Allow Origin:
Disabled.

Step 3 - Restrict API login to trusted hosts
--------------------------------------------
Enable:

Trusted Hosts

Add the Zabbix server IP address in CIDR format.

Examples:

10.10.10.50/32
10.10.10.50/32

Use only the real source IP addresses from which Zabbix connects to the FortiGate management/API interface.

Step 4 - Save and copy the API Token
------------------------------------
Click OK/Create.

FortiGate will display an API Token.

Copy the token immediately and store it securely.
Usually, the full token cannot be viewed again after closing the window.

Step 5 - Test API access from the Zabbix server
-----------------------------------------------
Run this command from the Zabbix server CLI:

curl -k -s \
  -H "Authorization: Bearer EXAMPLE_API_TOKEN_1234567890" \
  "https://FORTIGATE_IP/api/v2/monitor/system/status"

Example:

curl -k -s \
  -H "Authorization: Bearer EXAMPLE_API_TOKEN" \
  "https://10.20.30.1/api/v2/monitor/system/status"

Expected result:
A JSON response from the FortiGate, including system status information.

If the command hangs or fails, test HTTPS connectivity first:

curl -k -v --connect-timeout 5 https://FORTIGATE_IP/

If this fails, check:
- FortiGate administrative access on the interface
- HTTPS enabled on the FortiGate interface
- Routing between Zabbix and FortiGate
- Firewall policy / local-in policy
- Trusted Hosts configuration on the REST API Admin
- Whether the source IP is the expected Zabbix source IP

====================================================================
PART 2 - FortiGate SNMPv3 configuration
====================================================================

Step 1 - Enable SNMP on FortiGate
---------------------------------
In FortiGate GUI, go to:

System > SNMP

Enable SNMP Agent.

Recommended system information:
- Contact
- Location
- Description

Step 2 - Create SNMPv3 user
---------------------------
In:

System > SNMP > SNMP v3

Create a new SNMPv3 user for Zabbix.

Recommended values:

Username:
demo_snmp_user

Security level:
Authentication and Privacy

Authentication protocol:
SHA512

Authentication password:
Use a strong password.

Privacy protocol:
AES256

Privacy password:
Use a strong password.

Notification hosts:
Not required for polling-only monitoring.

Queries / Polling:
Allow SNMP queries from the Zabbix server.

Trusted host / source restriction:
Restrict access to the Zabbix server IP where possible.

Step 3 - Enable SNMP access on the FortiGate interface
------------------------------------------------------
On the FortiGate interface that receives SNMP requests from Zabbix:

Network > Interfaces > Edit interface

Enable administrative access:

SNMP

If Zabbix connects over the management interface, enable SNMP on the management interface.
If Zabbix connects over another routed interface, enable SNMP there.

Step 4 - Test SNMPv3 from the Zabbix server
-------------------------------------------
Run this from the Zabbix server:

snmpwalk -v3 -l authPriv \
  -u demo_snmp_user \
  -a SHA-512 -A 'ExampleAuthPass123!' \
  -x AES-256 -X 'ExamplePrivPass123!' \
  FORTIGATE_IP \
  10.10.10.50.2.1.1

If your net-snmp version does not accept SHA-512 or AES-256 syntax, try:

snmpwalk -v3 -l authPriv \
  -u demo_snmp_user \
  -a SHA512 -A 'ExampleAuthPass123!' \
  -x AES256 -X 'ExamplePrivPass123!' \
  FORTIGATE_IP \
  10.10.10.50.2.1.1

Expected result:
System OIDs such as sysDescr, sysObjectID, sysUpTime, sysContact, sysName, sysLocation.

Example OID for Fortinet enterprise tree:

snmpwalk -v3 -l authPriv \
  -u demo_snmp_user \
  -a SHA-512 -A 'ExampleAuthPass123!' \
  -x AES-256 -X 'ExamplePrivPass123!' \
  FORTIGATE_IP \
  10.10.10.50.4.1.12356

====================================================================
PART 3 - Zabbix 7.2 template import
====================================================================

Step 1 - Import the custom combined template
--------------------------------------------
In Zabbix 7.2 GUI, go to:

Data collection > Templates > Import

Upload:

FortiGate_Combined_HTTP_SNMP_Custom_Zabbix_7_2_Created_by_Roee_Golan.yaml

Recommended import options:
- Create new: enabled
- Update existing: optional, only if re-importing a newer version of the same custom template
- Delete missing: disabled, unless you fully understand the impact

Step 2 - Verify template name
-----------------------------
The imported template should have a custom name such as:

FortiGate Combined HTTP SNMP Custom

The YAML should include an identifying description/comment:

Created By Roee Golan

====================================================================
PART 4 - Zabbix host creation / configuration
====================================================================

Step 1 - Create or edit the FortiGate host
------------------------------------------
Go to:

Data collection > Hosts > Create host

Recommended values:

Host name:
Use a stable technical name, for example:
Demo-FortiGate-01

Visible name:
Use a friendly name, for example:
Demo FortiGate Firewall

Host groups:
Network devices
Firewalls
FortiGate
or your existing group structure.

Step 2 - Add interfaces
-----------------------
Add an Agent interface only if needed for other checks.
For this template, the important interfaces are:

1. SNMP interface
Type:
SNMP

IP address:
FORTIGATE_SNMP_IP

Port:
161

Example:
10.20.30.1
or the actual FortiGate IP used for SNMP polling.

2. HTTP/HTTPS endpoint
The HTTP template usually uses macros rather than an HTTP interface.
The FortiGate API URL is normally defined by macro.

Step 3 - Link the custom template
---------------------------------
In the host Templates field, link only the custom combined template:

FortiGate Combined HTTP SNMP Custom

Do not also link the original templates unless the custom template was explicitly built to avoid all duplicates.

====================================================================
PART 5 - Required Zabbix macros
====================================================================

The final exact macro names depend on the imported "FortiGate by HTTP" template version.
After importing, open:

Data collection > Templates > FortiGate Combined HTTP SNMP Custom > Macros

Then copy required macros to the host level when values differ per FortiGate.

Common HTTP/API macros
----------------------

{$FORTIGATE.API.TOKEN}
Value:
Example FortiGate REST API token generated for demo_api_user.

Example:
EXAMPLE_API_TOKEN_1234567890

{$FORTIGATE.API.URL}
or:
{$FG.API.URL}
or:
{$FORTIGATE.URL}

Value:
https://FORTIGATE_IP

Example:
https://10.20.30.1

If the template expects only host/IP and not full URL, use:

{$FORTIGATE.API.HOST}
Value:
10.20.30.1

Important:
Use the exact macro names from the imported template.
Do not invent macro names if the template already defines them differently.

Common SNMPv3 macros
--------------------

Depending on the Zabbix template, SNMPv3 credentials may be configured directly on the host SNMP interface or through macros.

Recommended host SNMPv3 settings:

Security name:
demo_snmp_user

Security level:
authPriv

Authentication protocol:
SHA512

Authentication passphrase:
ExampleAuthPass123!.

Privacy protocol:
AES256

Privacy passphrase:
ExamplePrivPass123!.

Context name:
Leave empty unless your FortiGate configuration requires a context.

If your Zabbix template uses macros, common examples may be:

{$SNMPV3.USER}
demo_snmp_user

{$SNMPV3.AUTHPASS}
ExampleAuthPass123!

{$SNMPV3.PRIVPASS}
ExamplePrivPass123!

{$SNMPV3.AUTHPROTOCOL}
SHA512

{$SNMPV3.PRIVPROTOCOL}
AES256

Use the exact macro names that exist in your template.

Step 4 - Recommended host-level macros example
----------------------------------------------

For the current process, the host-level values should be similar to:

{$FORTIGATE.API.TOKEN}
<FortiGate API token>

{$FORTIGATE.API.URL}
https://10.20.30.1

SNMP interface:
IP address: 10.20.30.1 or the SNMP polling IP
Port: 161
SNMP version: SNMPv3
Security name: demo_snmp_user
Security level: authPriv
Authentication: SHA512
Privacy: AES256

====================================================================
PART 6 - Validation in Zabbix
====================================================================

Step 1 - Check latest data
--------------------------
Go to:

Monitoring > Latest data

Filter by the FortiGate host.

Check that HTTP/API items receive values.

Step 2 - Check SNMP interface discovery
---------------------------------------
Look for discovered interface items such as:
- Interface operational status
- Interface traffic in
- Interface traffic out
- Interface errors
- Interface discards

Step 3 - Check unsupported items
--------------------------------
Go to:

Data collection > Hosts > FortiGate host > Items

Filter:
State = Not supported

Common causes:
- Wrong API token
- Wrong API URL
- HTTPS blocked
- Token Trusted Hosts mismatch
- SNMPv3 username/password mismatch
- Wrong SHA/AES protocol syntax
- SNMP not enabled on the FortiGate interface
- Duplicate items from old templates still linked to the host

Step 4 - Check graphs
---------------------
Go to:

Monitoring > Hosts > FortiGate host > Graphs

Verify that graphs exist and do not duplicate names from old templates.

If there is an import or host update error such as:

Graph "FortiGate: CPU utilization" already exists

then another linked template or inherited template still has a graph with the same name.
Remove the duplicate graph from the custom template, rename it, or unlink the conflicting old template.

====================================================================
PART 7 - Operational recommendation
====================================================================

Recommended final state:
- One custom combined template linked to the FortiGate host.
- HTTP/API data has priority.
- SNMPv3 is used mainly for interface counters.
- API token is restricted by Trusted Hosts.
- SNMPv3 uses authPriv with SHA512 and AES256.
- No duplicate original FortiGate templates linked to the same host.

End of README
