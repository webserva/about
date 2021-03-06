
#### How to create a client cert

Our Telegram bot @WebServaBot will propose a bash script rendered with your account name, as per your Telegram.org username.

Click on the link given by the bot with your Telegram.org name substituted already, to review the script, and copy its URL.

![Cert script review](https://evanx.github.io/images/rquery/ws040-cert-script-curl.png)
<hr>
(incomplete script shown)

As per the second line of the custom script, you can curl and pipe into bash to execute the script on your local machine.

The custom script will execute the following:
- create the `~/.webserva/live` directory for private key material
- use `openssl` to create a private key, self-signed certificate, and P12 for your browser
- advise how to install the `wscurl` wrapper script
- advise you of the cert ID to `/grant` via WebServaBot

Having granted the cert via WebServaBot, invoke the endpoint https://secure.webserva.com/register-cert using the cert in one of these ways:
- load `~/.webserva/live/privcert.p12` into your browser
- try `curl -E ~/.webserva/live/privcert.pem` https://secure.webserva.com/register-cert
- equivalently, install our `wscurl` wrapper script and try `ws register-cert`

Optional query paramaters for `/cert-script` include:
- `archive` - archive `~/.webserva/live` to `~/webserva/archive/TIMESTAMP`
- `id` - the client cert id e.g. `admin`
- `role` - the client cert tole e.g. `admin` or `submitter`

However these are still experimental in this MVP, and we cannot advise their correct and safe use yet.

### Scripts

The content of this script should be as follows when run with a placeholder `ACCOUNT` account name:
```shell
# Curl this script and pipe it into bash for execution, as per the following line:
# curl -s 'https://open.webserva.com/cert-script/ACCOUNT' | bash
#
( # create subshell for local vars and to enable set -u -e
  set -u -e # error exit if any undeclared vars or unhandled errors
  account='ACCOUNT' # same as Telegram.org username
  role='admin' # role for the cert e.g. admin
  id='admin' # user/device id for the cert
  CN='ws:ACCOUNT:admin:admin' # unique cert name (certPrefix, account, role, id)
  OU='admin' # role for this cert
  O='ACCOUNT' # account name
  dir=~/.webserva/live # must not exist, or be archived
  # Note that the following files are created in this dir:
  # account privkey.pem cert.pem privcert.pem privcert.p12 x509.txt cert.extract.txt
  commandKey='cert-script'
  serviceUrl='https://secure.webserva.com' # for cert access control
  telegramBot='WebServaBot' # bot for granting cert access
  archive='~/.webserva/archive' # directory to archive existing live dir when ?archive
  certWebhook='https://secure.webserva.com/create-account-telegram/ACCOUNT'
  mkdir -p ~/.webserva # ensure default dir exists
  if [ -d ~/.webserva/live ] # directory already exists
  then # must be archived first
    echo "Directory ~/.webserva/live already exists. Try add '?archive' query to the URL."
  else # fetch, review and check SHA of static cert-script.sh for execution
    mkdir ~/.webserva/live && cd $_ # error exit if dir exists
    curl -s https://raw.githubusercontent.com/webserva/webserva/master/bin/cert-script.sh -O
    echo 'Please review and press Ctrl-C to abort within 8 seconds:'
    cat cert-script.sh # review the above fetched script, we intend to execute
    echo 'Double checking script integrity hashes:'
    shasum cert-script.sh # double check its SHA against another source below
    curl -s https://open.webserva.com/assets/cert-script.sh.shasum
    echo 'aee902de05dc8e85a7592c85fb074c65fb77043c' # hardcoded SHA of stable version
    echo 'Press Ctrl-C in the next 8 seconds to abort, and if any of the above hashes differ'
    sleep 8 # give time to abort if SHAs not consistent, or script review incomplete
    source <(cat cert-script.sh) # execute fetched script, hence the above review and SHA
  fi
)
```
where we fetch https://raw.githubusercontent.com/webserva/webserva/master/bin/cert-script.sh which should be:

```shell
  echo "${account}" > account
  if openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -subj "/CN=${CN}/OU=${OU}/O=${O}" \
    -keyout privkey.pem -out cert.pem
  then
    openssl x509 -text -in cert.pem > x509.txt
    grep 'CN=' x509.txt
    echo -n `cat cert.pem | head -n-1 | tail -n+2` | sed -e 's/\s//g' | shasum | cut -f1 -d' ' > cert.pem.shasum
    cat privkey.pem cert.pem > privcert.pem
    openssl x509 -text -in privcert.pem | grep 'CN='
    curl -s -E privcert.pem "$certWebhook" ||
      echo "Registered account ${account} ERROR $?"
    if ! openssl pkcs12 -export -out privcert.p12 -inkey privkey.pem -in cert.pem
    then
      echo "ERROR $?: openssl pkcs12 ($PWD)"
      false # error code 1
    else
      echo "Exported $PWD/privcert.p12 OK"
      pwd; ls -l
      sleep 2
      curl -s https://open.webserva.com/cert-script-help/${account}
      curl -s https://raw.githubusercontent.com/webserva/webserva/master/docs/install.wscurl.txt
      certSha=`cat cert.pem.shasum`
      echo "Try '/grant $certSha' via https://telegram.me/WebServaBot?start"
    fi
  fi
```
where the custom script will check its SHA:
```shell
curl -s https://raw.githubusercontent.com/webserva/webserva/master/bin/cert-script.sh | shasum
```
```shell
a48e967d4fc898e9bc0c3509931c936a81a40582
```

Please review these scripts, and raise any issues with us.

Note that the customised script will show the fetched script's content, with comparative hashes that must match.
It then sleeps for 8 seconds to give you a chance to press Ctrl-C to abort.

### How to register a client cert

```shell
curl -E ~/.webserva/live/privcert.pem https://secure.webserva.com/register-cert
```

Actually, we recommend that you install `wscurl,` our `curl` wrapper script to use that `~/.webserva/live/privcert.pem` by default.
It also offers some builtin help with hints.

### How to CLI

Read the following instructions to install `wscurl,` our `curl` wrapper script to use your `privcert.pem` in `~/.webserva/live.`

https://raw.githubusercontent.com/webserva/webserva/master/docs/install.wscurl.txt

This snippet of online CLI help advises the following:
```shell
Try the following:
  cd
  git clone https://github.com/webserva/webserva.git
  alias ws='~/webserva/bin/wscurl.sh' # try add this line to ~/.bashrc
  ws help
```

![wscurl](https://evanx.github.io/images/rquery/ws040-wscurl.png)

#### Dangerzone

You should be able to remove all vestiges of the above from your machine as follows:

```shell
rm -rf ~/.webserva # contains private key matter
rm -rf ~/webserva # clone of https://github.com/webserva/webserva
```

#### Troubleshooting

Check your cert details:
```shell
openssl x509 -text -in ~/.webserva/live/privcert.pem | grep 'CN='
```

To see the commands being executed including `openssl,` you can use `bash -x` as follows:
```shell
curl -s https://open.webserva.com/cert-script/ACCOUNT | bash -x
```

https://twitter.com/@evanxsummers
