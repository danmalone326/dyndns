# Personal DynDNS Service

This project implements a personal dynamic DNS web service using a paired down, extended, and combined version of the legacy v2 and v3 dyndns protocols. It was created to allow easy dynamic dns updates to self hosted domains by standard dyndns clients, e.g. home routers. Extensions added help to enable letsencrypt validation of certificates for internal devices that are not accessible outside a home firewall.


## Usage

There you can use one of the popular dynamic DNS clients, e.g. DDclient, or configure your router with its built in options.

### Configuring a Router

Most routers will support a generic dynamic dns provider. You may be able to select one of the services that support the dyndns protocol, such as dyndns or noip.

You will need to enter the following information
- Hostname - the fully qualified name that you want to update
- Username - Your credentials for this service
- Password - Your credentials for this service
- Server - the server hosting the personal dynamic DNS web service 

### Using the API directly

You can update DNS manually or with custom scripting using the API by making GET requests to the web service URL:

`https://username:password@dyndns.example.com/nic/update?hostname=subdomain.yourdomain.com&myip=1.2.3.4`

Updates can be sent over HTTP or HTTPS, with preference to HTTPS since passwords are being used. HTTP on port 8245 is also available and may be used to bypass transparent HTTP proxies.

#### Update Parameters

hostname (required)
: Comma separated list of hostnames to update (max 20)
: Each hostname specified will be updated with the same information, and the return codes will be given one per line, in the same order as given.

myip
: Comma separated list of IP addresses to set for the update (max 20)
: Supports both IPv6 and IPv4 addresses. If this parameter is not specified, the best IP address the server can determine will be used (some proxy configurations pass the IP in a header, and that is detected by the server). Any IP addresses passed to the system that are not properly formed will be ignored and the system’s best guess will be used.

txt
: TXT value to set for the update
: If this parameter is specified, a TXT record will be added/updated for the hostname

delete
: A comma separated list of record types to delete
: Will delete all the records of the specified type(s). The list can contain the following values: 
A,
AAAA,
TXT,
ALL. 
If ALL is used, all allowed values will be deleted.

#### Return Codes

badauth
: The username and password pair do not match a real user.

good &lt;value&gt;
: The update was successful, and the hostname is now updated.

nochg &lt;value&gt;
: The update changed no settings, and is considered abusive. Additional nochg updates will cause the hostname to become blocked. [^1]

notfqdn
: The hostname specified is not a fully-qualified domain name (not in the form hostname.example.org or example.com). If no hostnames were specified, notfqdn will be returned once.

nohost
: The hostname specified is not found for this user.

numhost
: Too many hosts (more than 20) specified in an update.

numip
: Too many IP addresses (more than 20) specified in an update.

abuse [^1]
: The hostname specified is blocked for update abuse.

badagent [^1]
: The user agent was not sent or HTTP method is not permitted (we recommend use of GET request method).

badparam
: The parameter combinations are not valid for the given operation.

badop
: The request is not a valid operation.

dnserr
: DNS error was encountered.

911
: There is a problem.

[^1]: Abuse monitoring is not implemented yet


## Installation

### Prerequisites

These items are required before continuing and are outside the scope of this document
- Web server installed and running. Code/tips in this document assume Apache.
- DNS for the web service hostname pointing to the installed web server.
- Authoritative DNS server for the domain(s) that will be available for updates.

This information will be needed during the installation
- Base directory where this project will be installed.
- Domain name for the web service<br>
(examples will use `dyndns.domain.com`)
- The group that the web server is running as<br>
  `grep APACHE_RUN_GROUP /etc/apache2/envvars`


### Installation

- Clone the repository<br>
`git clone git@github.com:danmalone326/dyndns.git`
- Couldn't figure out how to get git to save this, so we need to secure the `secure` directory.<br>
`chmod -R o-rwx secure`
- Copy the Apache configuration template.<br>
`sudo cp etc/apache2/dyndns.example.org.conf /etc/apache2/sites-available/dyndns.domain.com.conf`
- Customize the Apache configuration.<br>
`vi /etc/apache2/sites-available/dyndns.domain.com.conf`
    - ddBaseDirectory - absolute path to the directory the repos was cloned to
    - ddServerName - the full DNS name of the web service
    - Optional: ddServerAlias - any additional DNS names
- Copy the password and configuration file templates<br>
`cp -p data/authorizedHosts.csv.template data/authorizedHosts.csv`<br>
`cp -p data/zoneInfo.csv.template data/zoneInfo.csv`<br>
`cp -p secure/htpasswd.template secure/htpasswd`
- Change group for files needed by the web service.<br>
`sudo chgrp -hR www-data data secure`
- Optional: Port 8245 may be used to bypass transparent HTTP proxies. To enable this, you will need to allow this port in your firewall.<br>
`sudo ufw allow 8245/tcp comment 'HTTP for DynDNS'`
- Enable the new site and restart Apache.<br>
`sudo a2ensite dyndns.domain.com`<br>
`sudo systemctl reload apache2`

## Configuration
There are 4 steps required to allow DNS updates.
- A user account must be created.
- A user must be granted permission to update one or more names or group of names.
- The authoritative DNS server must be configured to securely receive DNS updates for the (sub)domains that will be updated.
- Then this service must be configured to send the updates.

### Creating a new user account
A user account will be granted access to update one or more names. 
We are using Apache basic authentication, so the following can be used to create a new user account and password.<br>
`htpasswd -B secure/htpasswd janeuser`

For (slightly) better security, a user can create their own password entry and have it appended to the passwd file<br>
`htpasswd -n janeuser`

At this time, we do not support multiple keys for a single user, so, for now, to limit access, we need to create multiple users.<br>
e.g. janeuser-home, janeuser-work, janeuser-remote

### Granting access

Users are granted access to updating hosts through patterns. Patterns can match a single host or multiple with a wildcard. These are simple patterns, not regex, where a * represents a wildcard. 
Patterns must also reference the zone (see below) where the update will be sent. 
When users are granted access to multiple patterns, the first match of username and hostname to pattern found will be used.

Add patterns to the file `ata/authorizedHosts.csv`
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
    - On the DNS server, generate a new shared TSIG key
        - The ID used here must be unique within the scope of the DNS server.
        - This is a pre-shared key, so must be saved in a secure location. <br>
    `keymgr -t myUniqueKeyID`
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
            file: "ddns.outatime.com.zone"
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

## Table of contents (optional)

- Requirements
- Recommended modules
- Installation
- Configuration
- Troubleshooting
- FAQ
- Maintainers


## Requirements (required)

This module requires the following modules:
- [Bad judgement](https://www.drupal.org/project/bad_judgement)

OR

This module requires no modules outside of Drupal core.


## Recommended modules (optional)


## Installation (required, unless a separate INSTALL.md is provided)

Install as you would normally install a contributed Drupal module. For further information, see [Installing Drupal Modules](https://www.drupal.org/docs/extending-drupal/installing-drupal-modules).


## Configuration (required)

1. Enable the module at Administration > Extend.
1. Profit.


## Troubleshooting (optional)


## FAQ (optional)

**Q: How do I write a module README?**

**A:** Follow this template. It's fun and easy!


## Maintainers (optional)

- Dries Buytaert - [dries](https://www.drupal.org/u/dries)