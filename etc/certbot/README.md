# How to use certbot-hook

This script can be used with certbot to request a certificate using the DNS-01 challenge. 
This can be especially useful when generating certificates on a server on a home network, behind a firewall, or otherwise not publicly accessible. 

## Usage

- Install this certificate on the server that will run certbot.
  - Set the permissions on the script to allow only root to read/write the file. It will contain a password.
  - Edit the script and set the following variables.
    - DYNDNS_URL
    - DYNDNS_USERNAME
    - DYNDNS_PASSWORD
    - DDNS_REGEX (optional)
- Run certbot
    ```
    sudo certbot \
     certonly \
     --domains host.example.com \
     --preferred-challenges dns \
     --authenticator manual \
     --manual-auth-hook /root/dyndns/certbot-hook \
     --manual-cleanup-hook /root/dyndns/certbot-hook \
     --non-interactive \
     --redirect \
     --manual-public-ip-logging-ok \
     --max-log-backups 20
    ```
- Note, you can not use the built in installer options with the manual authenticator, e.g. --installer apache
  - You may be able to create a --deploy-hook that restarts services after a renewal
