# Personal DynDNS Service

This project implements a personal dynamic DNS web service using a paired down, extended, and combined version of the legacy v2 and v3 dyndns protocols. It was created to allow easy dynamic dns updates to self hosted domains by standard dyndns clients, e.g. home routers. Extensions added help to enable letsencrypt validation of certificates for internal devices that are not accessible outside a home firewall.

## Usage

You can use one of the popular dynamic DNS clients, e.g. [ddclient](https://github.com/ddclient/ddclient), or configure your router with its built in options.

### Configuring a Router

Some routers support a generic dynamic dns provider. You may be able to select one of the services that support the dyndns protocol, such as dyndns or noip.

You will need to enter the following information
- Hostname - the fully qualified name that you want to update
- Username - Your credentials for this service
- Password - Your credentials for this service
- Server - the server hosting the personal dynamic DNS web service 

### Using the API directly

You can update DNS manually or with custom scripting using the API by making GET requests to the web service URL:

```
https://username:password@dyndns.domain.com/nic/update?hostname=host.updatedomain.com&myip=1.2.3.4
```

Updates can be sent over HTTP or HTTPS, with preference to HTTPS since passwords are being used. HTTP on port 8245 is also available and may be used to bypass transparent HTTP proxies.

#### Update Parameters

The hostname parameter is required. Only one of the additional parameters may be used at the same time. An IP address update is the default if no additional parameter is included.

- **hostname** (required) - Comma separated list of hostnames to update (max 20)<br>
Each hostname specified will be updated with the same information, and the return codes will be given one per line, in the same order as given.
- **myip** - Comma separated list of IP addresses to set for the update (max 20)<br>
Supports both IPv6 and IPv4 addresses. If this parameter is not specified, the best IP address the server can determine will be used (some proxy configurations pass the IP in a header, and that is detected by the server). Any IP addresses passed to the system that are not properly formed will be ignored and the system’s best guess will be used.
- **txt** - TXT value to set for the update<br>
If this parameter is specified, a TXT record will be added/updated for the hostname
- **delete** - A comma separated list of record types to delete<br>
Will delete all the records of the specified type(s). The list can contain the following values: A AAAA TXT . ALL can be used to delete records for all of these types.

#### Return Codes

- **badauth** - The username and password pair do not match an existing user.
- **good** &lt;value&gt; - The update was successful, and the hostname is now updated.
- **nochg** &lt;value&gt; - The update changed no settings, and is considered abusive. Additional nochg updates will cause the hostname to become blocked. [^1]
- **notfqdn** - The hostname specified is not a fully-qualified domain name (not in the form hostname.example.org or example.com). If no hostnames were specified, notfqdn will be returned once.
- **nohost** - The hostname specified is not found for this user.
- **numhost** - Too many hosts (more than 20) specified in an update.
- **numip** - Too many IP addresses (more than 20) specified in an update.
- **abuse** [^1] - The hostname specified is blocked for update abuse.
- **badagent** [^1] - The user agent was not sent or HTTP method is not permitted (we recommend use of GET request method).
- **badparam** - The parameter combinations are not valid for the given operation.
- **badop** - The request is not a valid operation.
- **dnserr** - DNS error was encountered.
- **911** - There is a problem.

[^1]: Abuse monitoring is not implemented yet, so don't do it.

#### Check IP

You can check the best IP address the server can determine by using the `checkip` endpoint. The optional **myip** parameter can be used for troubleshooting to verify what address would be used.

```
https://dyndns.domain.com/checkip
```


## Installation

### Prerequisites

These items are required before continuing and are outside the scope of this document
- Web server installed and running. Code/tips in this document assume Apache.
    - Enable the rewrite and cgi Apache modules
        ```
        sudo a2enmod rewrite
        sudo a2enmod cgi
        ```
- DNS for the web service hostname pointing to the installed web server.
- Authoritative DNS server for the domain(s) that will be available for updates. Code/tips in this document assume Knot DNS.
- The `dnspython` python module is required.
    ```
    sudo pip3 install dnspython
    ```

This information will be needed during the installation
- Base directory where this project will be installed.
- Domain name for the web service<br>
(examples will use `dyndns.domain.com`)
- The group that the web server is running as<br>
  ```
  grep APACHE_RUN_GROUP /etc/apache2/envvars
  ```


### Installation

- Clone the repository<br>
    ```
    git clone git@github.com:danmalone326/dyndns.git
    ```
- Couldn't figure out how to get git to save this, so we need to secure the `secure` directory.<br>
    ```
    chmod -R o-rwx secure
    ```
- Copy the password and configuration file templates<br>
    ```
    cp -p data/authorizedHosts.csv.template data/authorizedHosts.csv
    cp -p data/zoneInfo.csv.template data/zoneInfo.csv
    cp -p secure/htpasswd.template secure/htpasswd
    ```
- Change group for files needed by the web service.<br>
    ```
    sudo chgrp -hR www-data data secure
    ```
- Create the Apache configuration.<br>
    ```
    etc/apache2/makeApacheConfig
    ```
    - Repo install directory - absolute path to the directory this repo was cloned to
    - Server full DNS name - the full DNS name of this web service
    - Admin email address - Displayed on error web page when thing go really wrong
- Copy the Apache configuration template.<br>
This filename is given in the output of the previous step
    ```
    sudo cp etc/apache2/dyndns.example.org.conf /etc/apache2/sites-available/
    ```
- Optional: Customize the Apache configuration.<br>
    - Add secondary domains with ServerAlias
    - Change log file location
    ```
    vi /etc/apache2/sites-available/dyndns.domain.com.conf
    ```
- Optional: Port 8245 may be used to bypass transparent HTTP proxies. To enable this, you will need to allow this port in your firewall.<br>
    ```
    sudo ufw allow 8245/tcp comment 'HTTP for DynDNS'
    ```
- Enable the new site and restart Apache.<br>
    ```
    sudo a2ensite dyndns.domain.com
    sudo systemctl reload apache2
    ```
- Verify the web server is responding.
    ```
    http://dyndns.domain.com/checkip
    ```
- (Optional/Recommended) Enabling SSL/TLS for the web service is highly recommended, but beyond the scope of this document. Assuming this site is accessible publicly, the above setup should enable letsencrypt's HTTP-01 domain validation.
    ```
    sudo certbot --apache --no-redirect
    ```

## Configuration
There are 4 steps required to allow DNS updates.
- A user account must be created.
- A user must be granted permission to update one or more names or group of names.
- The authoritative DNS server must be configured to securely receive DNS updates for the (sub)domains that will be updated.
- Then this service must be configured to send the updates.

### Creating a new user account
A user account will be granted access to update one or more names. 
We are using Apache basic authentication, so the following can be used to create a new user account and password.
```
htpasswd -B secure/htpasswd janeuser
```

For (slightly) better security, a user can create their own password entry and have it appended to the passwd file.
```
htpasswd -n janeuser
```

At this time, we do not support multiple keys for a single user, so, for now, to limit access, we need to create multiple users.<br>
e.g. janeuser-home, janeuser-work, janeuser-remote

### Granting access

Users are granted access to updating hosts through patterns. Patterns can match a single host or multiple with a wildcard. These are simple patterns, not regex, where a * represents a wildcard. 
Patterns must also reference the zone (see below) where the update will be sent. 
When users are granted access to multiple patterns, the first match of username and hostname to pattern found will be used.

Add patterns to the file `data/authorizedHosts.csv`
```
authorizedUser,pattern,zoneName
janedoe,site1.example.com,example.com
janedoe,site2.example.com,example.com
johndoe,*.updatedomain.com,updatedomain.com
```

### Configuring a new zone for Dynamic DNS updates
Each (sub)domain that will be allowed for updates must first be configured on an authoritative DNS server. In this document, we configure DNS to allow updates to the entire zone, even though we may limit the update using the authorizations above. You may want to consider additional security to limit what hostnames can be updated. Another option is to create a subdomain zone that allows updates only. 

1. Configure Authoritative DNS Server for Secure Dynamic Updates. These guides should help but additional configuration may be necessary for your setup.
    - <details>
        <summary>Configure Bind9 for Dynamic DNS</summary>

        - We are assuming the (sub)domain is already setup and responding to queries.
        - On the DNS server, generate a new shared TSIG key. The key name used here must be unique within the scope of the DNS server. Save this output in a secure location, it will be used in multiple steps below.
            ```
            tsig-keygen myUniqueKeyName
            ```    
        - Insert the output into the bind config `/etc/named.conf.local`. 
            ```
            key "myUniqueKeyName" {
                algorithm hmac-sha256;
                secret "84bZnP+IDhtz++++raqEk+q5grmji5RqNNEQr+sNb+k=";
            };
            ```
        - Add an `acl` section to `/etc/named.conf.local`. 
        At a minimum, the acl should contain the IP address of the web service that will be submitting the updates, or localhost IP if running on the same server. Multiple addresses can be included. The acl name used here must be unique within the scope of the DNS server.
            ```
            acl "myUniqueAclID" {
                127.0.0.1;
                172.20.3.1;
                key myUniqueKeyName;
            };
            ```
        - Add an `allow-update` option to the existing `zone` section using the new `acl`.
            ```
            zone "updatedomain.com" IN {
                type master;
                file "/var/lib/bind/zones/updatedomain.com";
                allow-update {myUniqueAclID;};
            };

            ```
        - Check the configuration.
            ```
            sudo named-checkconf -p /etc/named/named.conf
            ```
        - Restart bind9.
        </details>

    - <details>
        <summary>Configure Knot for Dynamic DNS</summary>

        - We are assuming the (sub)domain is already setup and responding to queries.
        - On the DNS server, generate a new shared TSIG key. The ID used here must be unique within the scope of the DNS server. Save this output in a secure location, it will be used in multiple steps below.
            ```
            keymgr -t myUniqueKeyName
            ```    
        - Add this to the `key` section of `/etc/knot/knot.conf`. 
        If another key already exists in this file, be sure to add the new key to the existing section.
        This is a YAML file, so be careful of the indentation.
            ```
            key:
            - id: myUniqueKeyName
                algorithm: hmac-sha256
                secret: oZYLbK4wx0dhCcv++HIsBIeQDK0=
            ```
        - Add to the `acl` section of `/etc/knot/knot.conf`. 
        At a minimum, the acl should contain the IP address of the web service that will be submitting the updates, or localhost IP if running on the same server. Multiple address keys can be included.
            ```
            acl:
            - id: myUniqueAclID
                address: 127.0.0.1
                address: 172.20.3.1
                action: update
                key: myUniqueKeyName
            ```
        - Add the acl to the existing `zone` section for the zone. Be sure to add this new acl as there may be others already existing, e.g. for zone transfers.
            ```
            zone:
            - domain: updatedomain.com
                acl: transfer_to_ns3
                acl: myUniqueAclID
                file: "updatedomain.com.zone"
            ```
        - Restart the knot service.
        </details>

    - <details>
        <summary>Test from the dns server</summary>

        - This can be difficult to troubleshoot. Check `/var/log/syslog`
        - Testing updates from the DNS server may require different IP addresses in the acl. 
        - Individual zone files must be in a location the DNS server has permisison to update. File system permissions and AppArmor can get in the way, e.g. zone files should not be in `/etc/...` directories. 
        - Here is an example dynamic DNS update using `nsupdate`.
            ```
            $ nsupdate
            > server 127.0.0.1
            > zone updatedomain.com
            > key hmac-sha256:myUniqueKeyID oZYLbK4wx0dhCcv++HIsBIeQDK0=
            > add jenny.updatedomain.com 30 A 86.75.30.9
            > send
            (pause here and verify the update)
            > delete jenny.updatedomain.com
            > send
            > quit
            ```
        </details>

2. Test from the server where the dyndns web service is running
    - The included script `bin/nsupdate-check` can be used to verify dynamic updates are configured correctly
    - The script will ask for 6 values
        1. the dns zone to test
        2. the name or IP address of the DNS server to send updates for the zone
        3. the port to use for the DNS updates
        4. the tsig key algorithm
        5. the tsig key name/id
        6. the tsig secret key
    - There are 2 modes of operation
        1. If the zone has already configured in the dyndns web service, you will be asked if you want to verify the current configuration. If yes, the configuration will be read and then the tests will be performed.
        2. If the zone is not configured in the dyndns web service, or you choose not to test the current configuration, you will be prompted for each of the values. A series of tests will be performed to validate the given values and verify the DNS server is accepting updates. When complete, updates to the dyndns web server configuration will be provided. 

3. Make the zone available in the dyndns web service
    - Create a key file in the `secure` directory. This key file needs a unique name and should have a `.key` extension, e.g. `updatedomain.com.key`. This key file contains a single line with values for algorithm, keyname, and secretkey separated by colons.
        ```
        hmac-sha256:myUniqueKeyName:oZYLbK4wx0dhCcv++HIsBIeQDK0=
        ```
    - Set the key file to have the same permissions and ownership as the htpasswd file.
        ```
        sudo chown --reference secure/htpasswd secure/updatedomain.com.key
        sudo chmod --reference secure/htpasswd secure/updatedomain.com.key
        ```
    - Add an entry in the `data/zoneInfo.csv` configuration. The name server and port are of the authoritative DNS server that will accept the dynamic updates. The IP address may be localhost if running on the same server.
        ```
        name,nameServer,nameServerPort,keyFile
        updatedomain.com,127.0.0.1,53,updatedomain.com.key
        ```

# License
MIT License

Copyright (c) 2023 Dan Malone

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.