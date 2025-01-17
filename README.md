# acme-hooked

This is a tiny, auditable script that you can throw on your server to issue and
renew TLS certificates via ACME (for example at [Let's
Encrypt](https://letsencrypt.org/) or [Buypass Go
SSL](https://www.buypass.com/ssl/products/acme)) based on the superb
[acme-tiny](https://github.com/diafygi/acme-tiny) by Daniel Roesler, and
sharing similar goals. Since it has to be run on your server and have access to
your private ACSD account key, the goal is to make it as short and auditable as
possible (currently less than 200 lines). The only prerequisites are python and
openssl, plus anything you need to run your hook scripts.

**PLEASE READ THE SOURCE CODE! YOU MUST TRUST IT WITH YOUR ACCOUNT KEY!**

## What's Different Compared to `acme-tiny`

First off, acme-hooked offers a compatibility wrapper that can act as a drop-in
replacement of acme-tiny. So switching is easy: there is no need to change your
existing setup. However, if you want to switch over to acme-hooked, it offers
the following improvements compared to the original acme-tiny:

- acme-hooked clearly separates the conerns of (1) interacting with the ACME
  server, and (2) modifying your local system. Where acme-tiny directly writes
  to your filesystem, acme-hooked separates this aspect out into hook scripts
  that have a clearly defined interface.

- acme-hooked supports processing of more than one CSR at once, without the need
  to register your account every time, like when running acme-tiny in a loop.

- acme-hooked is able to handle both DNS-01 and HTTP-01 type ACME challenges
  via its hook scripts. acme-tiny was built for the HTTP-01 challenge only.

- acme-hooked still follows the acme-tiny core idea of having tiny, auditable
  code. It still contains well under 200 lines of actual code that has been
  thoroughly re-checked, linted, and tidied up.

## How to Use This Script

If you already have an ACME issued certificate and just want to renew, you
should only have to do Steps 3 and 6.

### Step 1: Create an Account Key

You must have a public key registered with the ACME service and sign your
requests with the corresponding private key. If you don't understand what this
means, this script likely isn't for you! Please use alternatives like
[certbot](https://certbot.eff.org/). To create a new key, which can be used by
acme-hooked to register an account for you and sign all following requests, run
the following command:

```sh
openssl genrsa 4096 > account.key
```

### Step 2: Create a Certificate Signing Request (CSR) for Your Domains

The [ACME protocol](https://datatracker.ietf.org/doc/html/rfc8555) requires a
CSR file to be submitted to it, even for renewals. You can use the same CSR for
multiple renewals. NOTE: You can't use your account private key as your domain
private key!


```sh
# generate a domain private key (if you haven't already)
openssl genrsa 4096 > domain.key
```

```sh
# For a single domain
openssl req -new -sha256 -key domain.key -subj "/CN=yoursite.com" > domain.csr

# For multiple domains (use this one if you want both www.yoursite.com and yoursite.com)
openssl req -new -sha256 -key domain.key -subj "/" -addext "subjectAltName = DNS:yoursite.com, DNS:www.yoursite.com" > domain.csr

# For multiple domains (same as above but works with openssl < 1.1.1)
openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:yoursite.com,DNS:www.yoursite.com")) > domain.csr
```

### Step 3: Set Up a Hook Script to Provide ACSD Challenges

You must prove you own the domains you want a certificate for. ACSD requires
that you either provide a TXT record in your domain's DNS information, or that
you host a challenge file on your webserver. The challenge information is
generated by acme-hooked. A hook script, supplied by you, sets it up so that it
can be accessed over the internet. Several template hook scripts are available
for different purposes. Hook scripts are called by acme-hooked with the
following parameters (see the [Hook Scripts README](hooks/README.md) for details):

```sh
# set up a challenge
/path/to/hookscript setup <domain> <token> <content>

# make the challenge accessible to the internet
/path/to/hookscript activate

# check that the challenge works correctly
/path/to/hookscript check <domain> <token> <content>

# remove a challenge once it is completed
/path/to/hookscript remove <domain> <token> <content>

# do any remaining cleanup work
/path/to/hookscript finish
```

#### HTTP Challenge: Hosting Challenge Files

Challenge files need to be made accessible under ".well-known/acme-challenge/"
url path of your domain. NOTE: ACME servers will perform a plain HTTP request
to port 80 on your server, so you must serve the challenge files via HTTP (a
redirect to HTTPS is fine too).

```sh
# Make some challenge folder
mkdir -p /var/www/challenges/
```

Adapt the hook script so that it places the challenge files into this folder.

```nginx
# Example for nginx
server {
    listen 80;
    server_name example.org www.example.org;

    location /.well-known/acme-challenge/ {
        alias /var/www/challenges/;
        try_files $uri =404;
    }

    ...the rest of your config
}
```

#### DNS Challenge: A TXT Record Provisioned to Your DNS

To complete the DNS challenge, your nameserver must provide a TXT record
containing the challenge content under the "\_acme-challenge" subdomain of the
domain to be validated, like so (e.g. for BIND or NSD zonefiles):

```
_acme-challenge.www.example.org. 300 IN TXT "<content>"
```

### Step 4: Get a signed certificate!

With a hook script set up to provision one of the challenges above, you now just
need to run acme-hooked with enough permissions to read your private account key
and CSR.

```sh
# Run the script on your server for the HTTP challenge
python acme_hooked.py --account-key ./account.key --csr ./domain.csr --http-hook /path/to/hookscript

# Or, for the DNS challenge
python acme_hooked.py --account-key ./account.key --csr ./domain.csr --dns-hook /path/to/hookscript
```

Finally, the certificate itself will be given to your hook script to store somewhere, like so:

```sh
# Certificate is passed in via stdin by acme-hooked, equivalent to this:
echo "<certificate_content>" | /path/to/hookscript write <csrfile>
```

Alternatively, you can also use the acme-tiny compatibility wrapper as a drop-in replacement for acme-tiny:

```sh
# Internally, the compatibility layer calls acme-hooked and a comptability hook script
python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /var/www/challenges/ > ./output.crt
```

### Step 5: Install the certificate

The signed TLS certificate chain created by this script can be used along with
your private key to run an HTTPS server. You need to include them in the HTTPS
settings in your web server's configuration. Here's an example on how to
configure an nginx server:

```nginx
server {
    listen 443 ssl;
    server_name example.org www.example.org;

    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/domain.key;

    ...the rest of your config
}

server {
    listen 80;
    server_name example.org www.example.org;

    location /.well-known/acme-challenge/ {
        alias /var/www/challenges/;
        try_files $uri =404;
    }

    ...the rest of your config
}
```

### Step 6: Setup an auto-renew cronjob

Congrats! Your website is now using HTTPS! Unfortunately, most free ACME
certificates only last for 90 days, so you need to renew them often. No worries!
It's easy to automate! Just make a bash script and add it to your crontab (see
below for example script).

Example of a `renew_cert.sh`:
```sh
#!/usr/bin/sh
python /path/to/acme_hooked.py --account-key /path/to/account.key --csr /path/to/domain.csr --<http|dns>-hook /path/to/hookscript || exit
mv /path/to/newlycreated.crt /path/to/certificate.crt
service nginx reload
```

```
# Example line in your crontab (runs once per month)
0 0 1 * * /path/to/renew_cert.sh 2>> /var/log/acme_hooked.log
```

## Permissions

The biggest problem you'll likely come across while setting up and running this
script is permissions. You want to limit access to your account private key and
challenge (i.e. web folder or DNS server) as much as possible. It is recommended
to create a user specifically for handling this script, the account private key,
setting up the challenges, moving the certificates into place, and reloading the
HTTPS webserver.

**BE SURE TO:**
* Backup your account private key (e.g. `account.key`)
* Don't allow this script to be able to read your domain private key!
* Don't allow this script to be run as root!

## Feedback/Contributing

This project has a limited scope and codebase. It serves one purpose only: to
issue certificates via ACME. All other aspects are delegated to hook scripts,
via a clean interface. Bug reports and pull requests are welcome. In
particular, a large collection of ready-to-use hook scripts are an explicit
goal of the project.

Since the acme-hooked script itself handles users' private keys, in line with
the goals of acme-tiny, it must stay easy to audit by anyone who wants to run
it. Hence, pull requests that significantly increase the size of its codebase
will probably not be merged.

If you want to add features for your own setup to make things easier for you,
please do! acme-hooked is open source, so feel free to fork and modify as
necessary.
