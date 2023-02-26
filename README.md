# certbot-zimbra
Automated letsencrypt/certbot certificate deploy script for Zimbra hosts.

[![asciicast](https://asciinema.org/a/219713.svg)](https://asciinema.org/a/219713)

## Rewrite

Thanks to the awesome job of @jjakob the script has undergone a considerable rewrite. 
Some things changed, some parameters have been renamed, so **if you're upgrading please read the [WARNING chapter](#warning) below**.

We encourage you to test the script and report back any issues you might encounter. The latest version can be downloaded from the [Releases tab](https://github.com/YetOpen/certbot-zimbra/releases), or if you prefer bleeding edge (may be broken) from the [master branch directly](/../../raw/master/certbot_zimbra.sh).

If you encounter any problem please [open an issue](https://github.com/YetOpen/certbot-zimbra/issues/new).

Things explicitly not tested are in the [TESTING](TESTING) file.

USE AT YOUR OWN RISK.

## WARNING

The command line parameters were changed with v0.7. `-r/--renew-only` was renamed to `-d/--deploy-only`, and `-d` was changed to `-H`. This is a BREAKING change so please update your crontabs and any other places they are used. Some new parameters were added, though they won't break backwards-compatibility, they add new features. Refer to the usage and/or the changelog for more information.

# Installation

## Requirements

- bash, su, which, lsof or ss, openssl, grep, sed (GNU), gawk (GNU awk)
- ca-certificates (Debian/Ubuntu) or pki-base (RHEL/CentOS)
- Zimbra: zmhostname, zmcontrol, zmproxyctrl, zmprov, zmcertmgr
- zimbra-proxy installed and working or an alternate webserver configured for letsencrypt webroot
- Certbot >=1.12.0, either certbot or letsencrypt binary in PATH.

## Certbot installation

The preferred way is to install it is by using the wizard [at certbot's home](https://certbot.eff.org/). Select "Other" as software. This will allow you to install easily upgradable system packages.

By installing Certbot via packages it automatically creates a cron schedule and a systemd timer to renew certificates (at least on Ubuntu). 
We must **disable this schedule** because after the renew we must deploy it in Zimbra. Also certbot's timers will attempt to update the cert twice a day,
this means a Zimbra restart may happen during work hours.
So open `/etc/cron.d/certbot` with your favourite editor and **comment the last line**. To disable systemd timers run:

```
systemctl stop certbot.timer && systemctl disable certbot.timer
```

## certbot-zimbra installation

Download the latest release and install it (copy the latest URL from the Releases tab):

```
wget --content-disposition https://github.com/YetOpen/certbot-zimbra/archive/0.7.11.tar.gz
tar xzf certbot-zimbra-0.7.11.tar.gz certbot_zimbra.sh
chmod +x certbot_zimbra.sh
chown root: certbot_zimbra.sh
mv certbot_zimbra.sh /usr/local/bin/
```
Or from the master branch: [certbot_zimbra.sh](/../../raw/master/certbot_zimbra.sh)

# Usage

```bash
USAGE: certbot_zimbra.sh < -d | -n | -p > [-aNuzjxcq] [-H my.host.name] [-e extra.domain.tld] [-w /var/www] [-s <service_names>] [-P port] [-L "--extra-le-parameter"]...
  Only one option at a time can be supplied. Options cannot be chained.
  Mandatory options (only one can be specified):
	 -d | --deploy-only: Just deploys certificates. Can be run as --deploy-hook. If run standalone, assumes valid certificates are in /etc/letsencrypt/live. Incompatible with -n/--new, -p/--patch-only.
	 -n | --new: performs a request for a new certificate ("certonly"). Can be used to update the domains in an existing certificate. Incompatible with -d/--deploy-only, -p/--patch-only.
	 -p | --patch-only: does only nginx patching. Useful to be called before renew, in case nginx templates have been overwritten by an upgrade. Incompatible with -d/--deploy-only, -n/--new, -x/--no-nginx.

  Options only used with -n/--new:
	 -a | --agree-tos: agree with the Terms of Service of Let's Encrypt (avoids prompt)
	 -L | --letsencrypt-params "--extra-le-parameter": Additional parameter to pass to certbot/letsencrypt. Must be repeated for each parameter and argument, e.g. -L "--preferred-chain" -L "ISRG Root X1"
	 -N | --noninteractive: Pass --noninteractive to certbot/letsencrypt.
	 --no-override-key-type-rsa: if certbot >=v2.0.0 has been detected, do not override ECDSA to RSA with "--key-type rsa" (use this to get the default ECDSA key type, Zimbra does NOT support it!)

  Domain options:
	 -e | --extra-domain <extra.domain.tld>: additional domains being requested. Can be used multiple times. Implies -u/--no-public-hostname-detection.
	 -H | --hostname <my.host.name>: hostname being requested. If not passed it's automatically detected using "zmhostname".
	 -u | --no-public-hostname-detection: do not detect additional hostnames from domains' zimbraServicePublicHostname.

  Deploy options:
	 -s | --services <service_names>: the set of services to be used for a certificate. Valid services are 'all' or any of: ldap,mailboxd,mta,proxy. Default: 'all'
	 -z | --no-zimbra-restart: do not restart zimbra after a certificate deployment

  Port check:
	 -j | --no-port-check: disable port check. Incompatible with -P/--port.
	 -P | --port <port>: HTTP port the web server to use for letsencrypt authentication is listening on. Is detected from zimbraMailProxyPort. Mandatory with -x/--no-nginx.

  Nginx options:
	 -w | --webroot "/path/to/www": path to the webroot of alternate webserver. Valid only with -x/--no-nginx.
	 -x | --no-nginx: Alternate webserver mode. Don't check and patch zimbra-proxy's nginx. Must also specify -P/--port and -w/--webroot. Incompatible with -p/--patch-only.

  Output options:
	 -c | --prompt-confirm: ask for confirmation. Incompatible with -q/--quiet.
	 -q | --quiet: Do not output on stdout. Useful for scripts. Implies -N/--noninteractive, incompatible with -c/--prompt-confirm.
```


If no `-e` is given, the script will figure out the additional domain(s) to add to the certificate as SANs via `zmprov gd $domain zimbraPublicServiceHostname zimbraVirtualHostname`.
This can be skipped with `-u/--no-public-hostname-detection`, in which case only the CN from `zmhostname` or `-H/--hostname` will be used.

Only one certificate will be issued including all the found hostnames. The primary host will always be `zmhostname` or the one passed via `-H|--hostname`.


# Zimbra 8.6+ single server example

## Preparation

The script needs some prerequisites. They are listed under Installation/Requirements. The script will run a prerequisite check on startup and exit if anthing is missing.

In addition, there are different modes of operation, depending on your environment (proxy server):

### Zimbra-proxy mode (the default)

Uses zimbra-proxy for the letsencrypt authentication. Zimbra-proxy must be enabled and running. This is the preferred mode.

When starting, the script checks the status of zmproxyctl and checks if a process with the name "nginx" and user "zimbra" is listening on port zimbraMailProxyPort (obtained via zmprov).

The port can optionally be overridden with `-P/--port` or the port check skipped entirely with `-j/--no-port-check` if you are absolutely sure everything is set up correctly. The zmproxyctl status check can't be skipped.

Patches are applied to nginx's templates to pass .well-known to the webroot to make letsencrypt work, after which nginx is restarted.

Everything, including new certificate requests, can be done via certbot-zimbra in this mode.

### Alternate webserver mode

Is selected with `-x/--no-nginx`. Requires `-P/--port` and `-w/--webroot`. `--port` is checked for listening status. All zimbra-proxy checks are skipped.

Can be used in case you don't have zimbra-proxy enabled but have a different webserver as a reverse proxy in front of Zimbra. 

You'll have to configure the webserver for letsencrypt (to serve /.well-known from a webroot somewhere in the filesystem), some examples for this can be found [here.](https://www.hiawatha-webserver.org/forum/topic/2275)

Renewal can be done as per instructions below, but `--pre-hook` can be omitted.

## First run

If you don't yet have a letsencrypt certificate, you'll need to obtain one first. The script can do everything for you, including deploying the certificate and restarting zimbra.

Run
`./certbot_zimbra.sh -n -c`

This will do all pre-run checks, patch zimbra's nginx, run certbot to obtain the certificate, test it, deploy it and restart zimbra. Passing `-c|--prompt-confirm` means the script will prompt you for confirmation before restarting zimbra's nginx, running certbot/letsencrypt, deploying the certificate and restarting zimbra.

Certbot will also ask you some information about the certificate interactively, including an e-mail to use for expiry notifications. Please use a valid e-mail for this as should the automatic renewal fail for any reason, this is the way you'll get notified.

The domain of the certificate is obtained automatically using `zmhostname`. If you want to request a specific hostname use the `-H/--hostname` option. This domain will be the DN of the certificate.

The certificate can be requested with additional hostnames/SANs. By default the script fetches `zimbraPublicServiceHostname` and `zimbraVirtualHostname` attributes from all domains and if present, adds those to the certificate SANs to be requested. If you want to disable this behavior use the `-u/--no-public-hostname-detection` option.

**Note:** Let's Encrypt has a limit of a maximum of 100 domains per certificate at the time of this writing: [Rate Limits](https://letsencrypt.org/docs/rate-limits/)

To indicate additional domains explicitly use the `-e/--extra-domain` option (can be specified multiple times). Note that `-e` also disables additional hostname detection. 

Additional options can be passed directly to certbot/letsencrypt with `-L | --letsencrypt-params`. The option must be repeated for each letsencrypt option. For example, if you want 4096-bit certificates, add `-L "--rsa-key-size" -L "4096"`. Refer to certbot's documentation for more information.

## Running noninteractively

When retrieving a new certificate using `-n|--new`, certbot runs interactively. If you want to run it noninteractively, you can pass `-N/--noninteractive` which will be passed on to certbot. Also passing `-q/--quiet` will suppress the status output of the script.
Only do this if you're absolutely sure what you're doing, as this leaves you with no option to verify the detected hostnames, specify the certificate e-mail etc. `-N/--noninteractive` may be combined with `-q | --quiet` and/or `-L | --letsencrypt-params` to pass all the parameters to certbot directly, e.g. in scripts to do automated testing with staging certificates. 

## Renewal

### Renewal using crontab

EFF suggest to run *renew* twice a day. Since this would imply restarting zimbra, once a day outside workhours should be fine. So in your favourite place (like `/etc/cron.d/zimbracrontab` or with `sudo crontab -e`) schedule the command below, as suitable for your setup:

```
# certbot_zimbra.sh requires bash and a path with /usr/sbin
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Replace /usr/bin/certbot with the location of your certbot binary, use this to find it: which certbot letsencrypt
12 5 * * * root /usr/bin/certbot renew --pre-hook "/usr/local/bin/certbot_zimbra.sh -p" --deploy-hook "/usr/local/bin/certbot_zimbra.sh -d"
```

The `--pre-hook` ensures Zimbra's nginx is patched to allow certificate verification. You can omit it if you remember to manually execute that command after an upgrade or a reinstall which may restore nginx's templates to their default.

The `--deploy-hook` parameter is only run if a renewal was successful, this will run certbot-zimbra.sh with `-d|--deploy-only` to deploy the renewed certificates and restart zimbra.

The domain to renew is automatically obtained with `zmhostname`. If you need customized domain name pass the `-H|--hostname` parameter after `-d`.

If you want to suppress status output and only receive notifications on errors, you can add `--quiet` to certbot and both hooks.

**Make sure you have a working mail setup (valid aliases for root or similar) to get crontab failure notifications.**

### Renewal using Systemd

If you prefer systemd you can use these instructions.
The example below uses the deploy-hook which will only rerun the script if a renewal was successful and thus only reloading zimbra when needed.
Sadly, systemd doesn't have a built-in on-failure mail notification function like cron does so you won't be notified of failed renewals. One could write a service to do that via "OnFailure=".

Create a service file eg: /etc/systemd/system/renew-letsencrypt.service

```
[Unit]
Description=Renew Let's Encrypt certificates and deploy into Zimbra if successful
After=network-online.target

[Service]
Type=oneshot
# run certbot --renew with pre/post hooks. only deploys if renewal was successful.
# Replace /usr/bin/certbot with the location of your certbot binary, use this to find it: "which certbot letsencrypt"
ExecStart=/usr/bin/certbot renew --quiet --pre-hook "/usr/local/bin/certbot_zimbra.sh -p" --deploy-hook "/usr/local/bin/certbot_zimbra.sh -d"
```

Create a timer file to run the above once a day at 2am: /etc/systemd/system/renew-letsencrypt.timer

```
[Unit]
Description=Daily renewal of Let's Encrypt's certificates

[Timer]
# once a day, at 2AM
OnCalendar=*-*-* 02:00:00
# Be kind to the Let's Encrypt servers: add a random delay of 0–3600 seconds
RandomizedDelaySec=3600
Persistent=true

[Install]
WantedBy=timers.target
```

Then reload the unit file with
```
systemctl daemon-reload
systemctl start renew-letsencrypt.timer
systemctl enable renew-letsencrypt.timer
```

Check the timers status:
```
systemctl list-timers renew-letsencrypt.timer
```


## Alternate webserver mode

See [Preparation](#preparation): [Alternate webserver](#alternate-webserver)

### Alternate webserver, manual certbot new certificate request

As above, but the first certificate can be obtained manually with certbot outside of this script with the authenticator plugin of your choice. Refer to the letsencrypt documentation for first certificate request information.

After the certificate has been obtained, `-d/--deploy-only` can be used to deploy the certificate in Zimbra (to use it in services other than HTTP also) and renewal can be done as usual with `--deploy-hook`.

### No proxy server (manual certificate request with alternate authentication method)

Since the HTTP authentication method can't be used, an alternate method like DNS will have to be used. Refer to the letsencrypt documentation on obtaining certificates without HTTP.

Deployment and renewal can be done as in the [Alternate webserver manual mode](#alternate-webserver-manual-certbot-new-certificate-request).

### Manual certificate request example

Say you have apache in front of zimbra (or listening on port 80 only) just run certbot by hand with appropriate options to request the certificate for apache, and when done run
```
/usr/local/bin/certbot_zimbra.sh --deploy-only
```
so that it will deploy the certificate in zimbra.

Set up renewal as above, but without `--pre-hook`.

# Troubleshooting

## Error: port check failed

This usually means zimbra-proxy is misconfigured. In the default case (without port overrides) the script checks if zimbra-proxy's nginx is listening on "zimbraMailProxyPort" (can be read with zmprov, port 80 in most cases). If this check fails, zimbra-proxy is misconfigured, not enabled, not started or you have a custom port configuration and didn't tell the script via port override parameters.

Zimbra's proxy guide ([Zimbra Proxy Guide](https://wiki.zimbra.com/wiki/Zimbra_Proxy_Guide)) is usually quite confusing for a novice and may be difficult to learn. For this we have a quick [Zimbra proxy configuration for certbot-zimbra guide](https://github.com/YetOpen/certbot-zimbra/wiki/Zimbra-proxy-configuration-for-Certbot-Zimbra) to get you up and running quickly. Still, you should get to know zimbra-proxy and configure it according to your own needs.

## Error: unable to parse certbot version

This is caused by certbot expecting user input when the script tried to run it to detect its version. To fix this, run `certbot` on the command line manually and answer any questions it has or fix any errors. After this the script should work fine.

Newer versions of the script print a more descriptive error message if ran with `-c|--prompt-confirm`.

## certbot failures

## General certbot troubleshooting

Check that you have an updated version of certbot installed. If you have installed certbot from your operating system's repositories, they may be out of date, especially on non-rolling distributions. If your distribution's certbot is outdated, remove the system packages and install it the way that certbot recommends for your operating system on their installation page, or a different way that you prefer.

Check certificate statuses with `certbot certificates`. Remove any duplicate or outdated certificates for the same domain names.

Check that ports 80 and 443 are open and accessible from the outside and check that your domain points to the server's IP. Basically troubleshoot Letsencrypt as if you weren't using certbot-zimbra.

## `cat: /etc/ssl/certs/2e5ac55d.0: No such file or directory` OR `Can't find "DSTRootCAX3"` OR `Unable to validate certificate chain: O = Digital Signature Trust Co., CN = DST Root CA X3`

Letsencrypt's "DST Root CA X3" expired in September 2021. Already issued certificates were cross-signed with both the old "DST Root CA X3" and new "ISRG Root X1" chains. Due to the way certbot-zimbra parses certificate files, it may cause certbot-zimbra to use the wrong chain when deploying the certificate. See issue #140.

Procedure to fix it:

- make sure you have latest ca-certificates (Debian/Ubuntu) or pki-base (RHEL/CentOS) package (do a apt-get dist-upgrade/upgrade/install ca-certificates or equivalent yum/dnf command), this will make sure you have the "ISRG Root X1" CA in the system-wide CA store
- force a renewal with `certbot renew --force-renewal --preferred-chain "ISRG Root X1" --cert-name "zimbra-cert-name"` Replace zimbra-cert-name with the name of your existing cert, you can find it with `certbot certificates`.
- If the previous step is successful, run `/usr/local/bin/certbot_zimbra.sh -d` to deploy the new cert.

The fix for new certificate requests is included in certbot-zimbra >=0.7.13, it will by default request new certs with `--preferred-chain "ISRG Root X1"`. Just upgrading certbot-zimbra will not fix the problem as you need to manually trigger a renewal or new cert request in certbot.

## zmcertmgr certificate and private key do not match ("expecting an rsa key")

Certbot v2.0.0 switched to ECDSA private keys by default, which Zimbra's zmcertmgr doesn't support. See [certbot docs](https://github.com/certbot/certbot/blob/caad4d93d048d77ede6508dd42da1d23cde524eb/certbot/docs/using.rst#id34)

It may be possible to [patch zmcertmgr](https://forums.zimbra.org/viewtopic.php?f=15&t=69645&p=301580) to support ECDSA keys, but this is not officially supported or widely tested.

Certbot-zimbra >=0.7.13 will auto-detect if certbot is >=2.0.0 and apply options while requesting a new certificate to obtain a RSA key.

If you used certbot >=2 with certbot-zimbra <0.7.13, you might run into this issue. There are two options to fix it:

- `certbot renew --key-type rsa --rsa-key-size 4096 --cert-name "zimbra-cert-name" --force-renewal` replace zimbra-cert-name with the name of the existing certificate, you can find it with `certbot certificates`. You can also change the key size to one that you prefer. If renewal is successful, redeploy the certificate with `/usr/local/bin/certbot_zimbra.sh -d`.
- update to certbot-zimbra >=0.7.13 and rerequest the certificate with `certbot-zimbra --new`, and add all the options you used with the original `--new` invocation, else your certificate may get replaced with one with different CN and SANs.

# Notes

## Notes on zimbraReverseProxyMailMode 

Letsencrypt by default tries to verify a domain using http, so the script should work fine if [zimbraReverseProxyMailMode](https://wiki.zimbra.com/wiki/Enabling_Zimbra_Proxy_and_memcached#Protocol_Requirements_Including_HTTPS_Redirect) is set to http, both, redirect or mixed. It won't work if set to https only. This is due to certbot deprecating the tls-sni-01 authentication method and switching to HTTP-01. https://letsencrypt.org/docs/challenge-types/

## Limitations

The script doesn't handle multiple domains configured with SNI (see #8). You can still request a single certificate for multiple hostnames.

## Upgrade from v0.1

If you originally requested the certificate with the first version of the script, which used *standalone* method, newer version will fail to renew. This because it
now uses *webroot* mode by patching Zimbra's nginx, making it more simple to work and to mantain.

To check if you have the old method, run `grep authenticator /etc/letsencrypt/renewal/YOURDOMAIN.conf`. If it says *standalone* it uses the old method.

To update to the new "webroot" method you can simply run `certbot-zimbra.sh -n -c -L "--force-renewal"`. This will force renew your existing certificate and save the new authentication method. It'll also ask you for deploying the new certificate in Zimbra. You can also manually modify the config file in /etc/letsencrypt/renewal/, while not recommended, is detailed here: https://community.letsencrypt.org/t/how-to-change-certbot-verification-method/56735

## How it works
This script uses zimbra-proxy's nginx to intercept requests to `.well-known/acme-challenge` and pass them to a custom webroot folder. To do this, we patch the templates Zimbra uses to build nginx's configuration files.
The patch is simple, we add this new section to the end of the templates:
```
    # patched by certbot-zimbra.sh
    location ^~ /.well-known/acme-challenge {
        root $WEBROOT;
    }
```
`$WEBROOT` is either `/opt/zimbra/data/nginx/html` (default) or the path specified by the command line option.
After this we restart zmproxy to apply the patches.

We then pass this webroot to certbot with the webroot plugin to obtain the certificate.

After the certificate has been obtained successfully we stage the certificates in a temporary directory, find the correct CA certificates from the system's certificate store and build the certificate files in a way Zimbra expects them. If verification with zmcertmgr succeeds we deploy the new certificates, restart Zimbra and clean up the temporary files.

After the first patching the script will check if the templates have been already patched and if so, it skips the patching and zmproxy restart steps. This is useful in cron jobs where even if we upgrade Zimbra and wipe out the patched templates they'll be repatched automatically.

The use of `--deploy-only` from `--deploy-hook` in cron jobs will only deploy the certificates if a renewal was successful. Thus Zimbra won't be unnecessarily restarted if no renewal was done.

## Certbot certificate privacy/security notes

Certbot preserves the gid and the g:rwx and o:r permissions from old privkey files to the renewed ones. This is described in 
https://github.com/certbot/certbot/blob/8b684e9b9543c015669844222b8960e1b9a71e97/certbot/storage.py#L1107

If you have some old certificates you've been renewing for a long time, it may be possible your privkey is created with other read permissions. This may be bad if all the containing directories are also other-readable. In my case they were not (the archive dir was mode 700) so the contained private keys were also not readable. Still, you may consider checking your situation and chmod'ing the privkeys to something more sensible like 640:

`chmod 640 /etc/letsencrypt/archive/*/privkey*.pem`

The default for new privkeys is 600.

If you want the keys in /etc/letsencrypt to be readable by some other programs, adjust the folder and file permissions as necessary, for example:
```
addgroup --system ssl-cert
chmod g+rx /etc/letsencrypt/{live,archive}
chgrp -R ssl-cert /etc/letsencrypt
addgroup ssl-cert <user that needs key access>
```

# License

See [LICENSE](LICENSE).

### Disclaimer of Warranty

THERE IS NO WARRANTY FOR THE PROGRAM, TO THE EXTENT PERMITTED BY APPLICABLE LAW. EXCEPT WHEN OTHERWISE STATED IN WRITING THE COPYRIGHT HOLDERS AND/OR OTHER PARTIES PROVIDE THE PROGRAM “AS IS” WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE. THE ENTIRE RISK AS TO THE QUALITY AND PERFORMANCE OF THE PROGRAM IS WITH YOU. SHOULD THE PROGRAM PROVE DEFECTIVE, YOU ASSUME THE COST OF ALL NECESSARY SERVICING, REPAIR OR CORRECTION.

# Author

&copy; Lorenzo Milesi <maxxer@yetopen.com>

## Contributors
- Jernej Jakob @jjakob
- @eN0RM
- Pavel Pulec @pulecp
- Antonio Prado
- @afrimberger
- @mauriziomarini

*if you are a contributor, add yourself here (and in the code)*


Feedback, bugs, PR are welcome on [GitHub](https://github.com/yetopen/certbot-zimbra).
