# Configuring ddclient for a Custom Dynamic DNS Service

During install:
```
apt install ddclient
```
or reconfiguration:
```
dpkg-reconfigure ddclient
```

- Dynamic DNS service provider: `other`
- Dynamic DNS update protocol: `dyndns2`
- Dynamic DNS server (blank for default):  `dyndns.server.com`
- Optional HTTP proxy: 
- Username: `username`
- Password: `password`
- Re-enter password: `password`
- IP address discovery method:
    - <details>
        <summary>Configure to use public IP address</summary>

        - IP address discovery method: `Web-based IP discovery service`
        - IP discovery service: `other`
        - IP discovery service URL: `https://dyndns.server.com/checkip`
        </details>
    - <details>
        <summary>Configure to use (private) IP address of a specific interface</summary>

        - IP address discovery method: `Network interface`
        - Network interface: `eth0` [^1]
        - How to run ddclient: `As a daemon`
        </details>

- Time between address checks: `5m`
- Hosts to update (comma-separated): `host.dynzone.tld`

[^1]: You can run this command to get the interface of your default route:<br>`ip route get 1 | grep -oP '\K\S+ *(?=src)'`