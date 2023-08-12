# Personal DynDNS Service

This project implements a personal dynamic DNS web service using a paired down, extended, and combined version of the legacy v2 and v3 dyndns protocols. It was created to allow easy dynamic dns updates to self hosted domains by standard dyndns clients, e.g. home routers. Extensions added help to enable letsencrypt validation of certificates for internal devices that are not accessible outside a home firewall.


## Usage

You can use one of the popular dynamic DNS clients, e.g. [ddclient](https://github.com/ddclient/ddclient), or configure your router with its built in options.

### Configuring a Router

Most routers will support a generic dynamic dns provider. You may be able to select one of the services that support the dyndns protocol, such as dyndns or noip.

You will need to enter the following information
- Hostname - the fully qualified name that you want to update
- Username - Your credentials for this service
- Password - Your credentials for this service
- Server - the server hosting the personal dynamic DNS web service 

### Using the API directly

You can update DNS manually or with custom scripting using the API by making GET requests to the web service URL:

```
https://username:password@dyndns.domain.com/nic/update?hostname=host.example.com&myip=1.2.3.4
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
- DNS for the web service hostname pointing to the installed web server.
- Authoritative DNS server for the domain(s) that will be available for updates.
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
- (Optional/Recommended) Enabling SSL/TLS for the web service is highly recommended, but beyond the scope of this document. The above setup should allow letsencrypt's HTTP domain validation.
    - My notes from certbot
        - had to remove the UnDefines at the end of the conf file
        - had to comment out the ServerAlias line, certbot doesn't understand the IfDefine
        - actually, in the end, using variables in the config completely broke the certbot compatibility. I'll have to try something else for this later.
        - after certbot completed, commented out the http->https redirect, some clients only do http

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
johndoe,*.example.org,example.org
```

### Configuring a new zone for Dynamic DNS updates
Each (sub)domain that will be set updates must first be configured on an authoritative DNS server. In this document, we configure DNS to allow updates to the entire zone, even though we may limit the update using the authorizations above. You may want to consider additional security to limit what hostnames can be updated. Another option is to create a subdomain zone that allows updates only.

- Configure the DNS server
    - We are assuming the (sub)domain is already setup and responding to queries.
    - On the DNS server, generate a new shared TSIG key. The ID used here must be unique within the scope of the DNS server. Save this output in a secure location, it will be used in multiple steps below.
        ```
        keymgr -t myUniqueKeyID
        ```    
    - Add this to the `key` section of `/etc/knot/knot.conf`. 
    If another key already exists in this file, be sure to add the new key to the existing section.
    This is a YAML file, so be careful of the indentation.
        ```
        key:
        - id: myUniqueKeyID
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
            key: myUniqueKeyID
        ```
    - Add the acl to the existing `zone` section for the zone. Be sure to add this new acl as there may be others already existing, e.g. for zone transfers.
        ```
        zone:
        - domain: example.com
            acl: transfer_to_ns3
            acl: myUniqueAclID
            file: "example.com.zone"
        ```
    - Test an update via command line before continuing.
        ```
        nsupdate -y hmac-sha256:myUniqueKeyID:oZYLbK4wx0dhCcv++HIsBIeQDK0= <<EOF
        server 127.0.0.1
        zone example.com
        add test.example.com 30 A 10.9.8.7
        send
        EOF
        ```
- Make the zone available in the web service
    - Create a key file in the `secure` directory. This key file needs a unique name and should have a `.key` extension, e.g. `example.com.key`. This key file is a yaml file with a single `key` section with a single key that matches the key in `knot.conf`.
        ```
        key:
            - id: myUniqueKeyID
                algorithm: hmac-sha256
                secret: oZYLbK4wx0dhCcv++HIsBIeQDK0=
        ```
    - Set the key file to have the same permissions and ownership as the htpasswd file.
        ```
        sudo chown --reference secure/htpasswd secure/example.com.key
        sudo chmod --reference secure/htpasswd secure/example.com.key
        ```
    - Add an entry in the `data/zoneInfo.csv` configuration. The IP address and port are of the DNS server that will accept the updates. The IP address may be localhost if running on the same server.
        ```
        name,nameServerIP,nameServerPort,keyFile
        example.com,127.0.0.1,53,example.com.key
        ```
