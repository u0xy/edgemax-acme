# ACME DNS-01 for Ubiquiti EdgeRouter

This repository is heavily based on
[https://github.com/j-c-m/ubnt-letsencrypt/](https://github.com/j-c-m/ubnt-letsencrypt/)
and https://github.com/hungnguyenm/edgemax-acme. It's a simpler version to
generate and automatically renew SSL certificate from [Let's
Encrypt](https://letsencrypt.org/) without reconfiguring firewall and exposing
any port to the internet.

This is beneficial especially in restricted network (behind firewall or double
NAT) or non-available required ports (i.e., 80, 443 used by other services).

It does require DNS API access from the DNS provider. The list of supported DNS
provider can be found from [acme.sh wiki
page](https://github.com/acmesh-official/acme.sh#8-automatic-dns-api-integration).


## Requirements

1. Determine required scripts

    First, you need to validate if your DNS provider is supported by [acme.sh
    dnsapi](https://github.com/acmesh-official/acme.sh/tree/master/dnsapi). To
    minimize the space needed, you only need to install the corresponding API
    script to your Router. For example, GoDaddy only needs `dns_gd.sh`,
    Namecheap only needs `dns_namecheap.sh`.

2. Obtain API login information from DNS provider

    Follow the instruction from [acme.sh
    dnsapi](https://github.com/acmesh-official/acme.sh/tree/master/dnsapi) to
    get your API login information. Also take note the required tags. For instance

    - GoDaddy needs `GD_Key` and `GD_Secret`.
    - Namecheap needs `NAMECHEAP_API_KEY`, `NAMECHEAP_USERNAME`,
      `NAMECHEAP_SOURCEIP`.


## Installation

You need to install `acme.sh`, `renew.acme.sh`, `reload.acme.sh`, and the
corresponding DNS API script. The scripts assume that `acme.sh` is put in
`/config/scripts/acme`. If you decide to use different folder, you'll need to
modify the `renew.acme.sh` to reflect the change.

As the `ubnt` user on your Router:

    mkdir -p /config/scripts/acme/dnsapi
    curl -o /config/scripts/acme/acme.sh https://raw.githubusercontent.com/acmesh-official/acme.sh/master/acme.sh
    curl -o /config/scripts/renew.acme.sh https://raw.githubusercontent.com/hungnguyenm/edgemax-acme/master/renew.acme.sh
    curl -o /config/scripts/reload.acme.sh https://raw.githubusercontent.com/u0xy/edgemax-acme/master/reload.acme.sh
    curl -o /config/scripts/acme/dnsapi/[yourdnsapi].sh https://raw.githubusercontent.com/acmesh-official/acme.sh/master/dnsapi/[yourdnsapi].sh
    chmod 755 /config/scripts/acme/acme.sh /config/scripts/renew.acme.sh /config/scripts/reload.acme.sh /config/scripts/acme/dnsapi/[yourdnsapi].sh

Remember to replace `[yourdnsapi]` with your DNS provider script file name from above.

Create a credentials file `/config/scripts/account.conf` with the required tags. For instance for Namecheap, it is:

    export NAMECHEAP_API_KEY='xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    export NAMECHEAP_USERNAME='xxxx'
    export NAMECHEAP_SOURCEIP='xxx.xxx.xxx.xxx'

After these steps, your `/config/scripts` folder should have the following structure:

    scripts
    ├── account.conf
    ├── acme
    │   ├── acme.sh
    │   └── dnsapi
    │       └── dns_namecheap.sh
    ├── reload.acme.sh
    └── renew.acme.sh


## Request certificate the first time

The script `renew.acme.sh` has the following options:

- `-d` (required) is the domain to issue certificate. You can add multiple domains by repeating this option.
- `-n` (required) is the DNS provider id. It is the same with your DNS API script from [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/tree/master/dnsapi).
- `-c` (required) path to the credentials file (probably `/config/scripts/account.conf`)
- `-i` (optional) flag to enable insecure mode.
- `-v` (optional) flag to enable acme verbose.

As ACME now prevents `acme.sh` to be called with sudo, we'd need to switch to root user before running the script the first time:

    sudo su

With the root shell, the command below works for Namecheap DNS:

    /config/scripts/renew.acme.sh -d subdomain.example.com -n dns_namecheap -c /config/scripts/account.conf

If you need extra arguments to acme.sh (perhaps for a [challenge
alias](https://github.com/acmesh-official/acme.sh/wiki/DNS-alias-mode)) specify
them at the end after a ```--```:

    /config/scripts/renew.acme.sh -d subdomain.example.com -n dns_namecheap -c /config/scripts/account.conf -- --challenge-alias challenge-domain.example.com

This `renew.acme.sh` does the heavy lifting, interacting with your DNS provider
and LetsEncrypt. As a last step, it also call the `reload.acme.sh` script,
which concatenates the key and cer in `/config/ssl/server.pem`.


## Configure your Router to use the generated `server.pem` file

Ensure the domain points to the Router's internal IP address.

You can configure in two ways (assuming internal IP address is 192.168.1.1):

- Router static host mapping: `set system static-host-mapping host-name subdomain.example.com inet 192.168.1.1`
- domain A record: depends on DNS provider, you can add an A record to the DNS database

Then, as root

	configure
	set service gui cert-file /config/ssl/server.pem
	commit
	save
	exit

You should now be able to access your Router at
[https://subdomain.example.com](https://subdomain.example.com). Verify if the
certificate is trusted.


## Configure automatic renew

If the above steps succeeded, the management UI is accessible with the new
valid certificate, and you're now ready to schedule task for automatic renewing
certificate. The following commands create a cronjob to execute `renew.acme.sh`
every day, with the same arguments that we run earlier. Since `acme.sh` script
only renews cert every 60 days, this task will just quit within the first 60
days. At the time this guide is written, all Let's Encrypt certificates expire
after 90 days.

As root,

    configure
    set system task-scheduler task renew.acme executable path /config/scripts/renew.acme.sh
    set system task-scheduler task renew.acme interval 7d
    set system task-scheduler task renew.acme executable arguments '-d subdomain.example.com -n dns_namecheap -c /config/scripts/account.conf'
    commit
    save
    exit

## Changelog

    2020-02-02: use account.conf instead of passing arguments
    2020-01-17: Update the first-time command to fix sudo error from acme.sh
    2018-09-14: Add an option for providing arbitrary arguments to acme.sh
    2018-04-22: Change RSA certificate to ECDSA P-384; Set default log to /var/log/acme.log
    2017-12-21: Add -i and -v options in renew.acme.sh
    2017-12-02: Remove &quot; in task-scheduler arguments
