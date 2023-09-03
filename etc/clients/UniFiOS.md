# Configuring UniFi OS for a Custom Dynamic DNS Service
You must switch to the legacy interface to see the dynamic dns server configuration.

- Network
- Settings
- Services
- Dynamic DNS
- Create New Dynamic DNS
    - Service: dyndns
    - Hostname: host.dynzone.tld
    - Username: username
    - Password: password
    - Server: `ddns.server.com/nic/update?hostname=%h&myip=%i`