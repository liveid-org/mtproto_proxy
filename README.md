Erlang mtproto proxy
====================

This part of code was extracted from [@socksy_bot](https://t.me/socksy_bot).

Features
--------

* Promoted channels! See `mtproto_proxy_app.src` `tag` option.
* "secure" randomized-packet-size protocol (34-symbol secrets starting with 'dd')
  to prevent detection by DPI
* Secure-only mode (only allow connections with 'dd'-secrets). See `allowed_protocols` option.
* Multiple ports with unique secret and promo tag for each port
* Automatic configuration reload (no need for restarts once per day)
* Most of the configuration options can be updated without service restart
* Very high performance - can handle tens of thousands connections! Scales to all CPU cores.
* Supports multiplexing (Many connections Client -> Proxy are wrapped to small amount of
  connections Proxy -> Telegram Server)
* Small codebase compared to official one
* A lots of metrics could be exported (optional)

How to start - docker
---------------------

### To run with default settings

```bash
docker run -d --network=host seriyps/mtproto-proxy
```

### To run on single port with custom port, secret and ad-tag

```bash
docker run -d --network=host seriyps/mtproto-proxy -p 443 -s d0d6e111bada5511fcce9584deadbeef -t dcbe8f1493fa4cd9ab300891c0b5b326
```

or via environmet variables

```bash
docker run -d --network=host -e MTP_PORT=443 -e MTP_SECRET=d0d6e111bada5511fcce9584deadbeef -e MTP_TAG=dcbe8f1493fa4cd9ab300891c0b5b326 seriyps/mtproto-proxy
```

Where

* `-p 443` / `MTP_PORT` proxy port
* `-s d0d6e111bada5511fcce9584deadbeef` / `MTP_SECRET` proxy secret (don't append `dd`! it should be 32 chars long!)
* `-t dcbe8f1493fa4cd9ab300891c0b5b326` / `MTP_TAG` ad-tag that you get from [@MTProxybot](https://t.me/MTProxybot)

### To run with custom config-file

1. Get the code `git clone https://github.com/seriyps/mtproto_proxy.git && cd mtproto_proxy/`
2. Copy config templates `cp config/{vm.args.example,prod-vm.args}; cp config/{sys.config.example,prod-sys.config}`
3. Edit configs. See [Settings](#settings).
4. Build `docker build -t mtproto-proxy-erl .`
5. Start `docker run -d --network=host mtproto-proxy-erl`

Installation via docker can work well for small setups (10-20k connections), but
for more heavily-loaded setups it's recommended to install proxy directly into
your server's OS (see below).

How to start OS-install - quick
-----------------------------------

```bash
wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb && dpkg -i erlang-solutions_1.0_all.deb
apt-get update && apt-get upgrade -y && apt dist-upgrade -y
apt install erlang-nox erlang-dev build-essential git curl htop sudo nginx -y
git clone https://github.com/hookzof/mtproto_proxy && cd mtproto_proxy
cp config/vm.args.example config/prod-vm.args && cp config/sys.config.example config/prod-sys.config
nano config/prod-sys.config
make && make install && systemctl enable mtproto-proxy && systemctl start mtproto-proxy
```

How to start OS-install - detailed
--------------------------------------


### Install deps (ubuntu 18.04)

```bash
sudo apt install erlang-nox erlang-dev build-essential
```

You need Erlang version 20 or higher! If your version is older, please, check
[Erlang solutions esl-erlang package](https://www.erlang-solutions.com/resources/download.html)
or use [kerl](https://github.com/kerl/kerl).

### Get the code:

```bash
git clone https://github.com/seriyps/mtproto_proxy.git
cd mtproto_proxy/
```

### Create config file

see [Settings](#settings).

### Build and install

```bash
make && sudo make install
```

This will:
* install proxy into `/opt/mtp_proxy`
* create a system user
* install systemd service
* create a directory for logs in `/var/log/mtproto-proxy`
* Configure ulimit of max open files and `CAP_NET_BIND_SERVICE` by systemd

### Try to start in foreground mode

This step is optional, but it can be usefull to test if everything works as expected

```bash
./start.sh
```

try to run `./start.sh -h` to learn some useful options.

### Start in background and enable start on system start-up

```bash
sudo systemctl enable mtproto-proxy
sudo systemctl start mtproto-proxy
```

Done! Proxy is up and ready to serve now!

### Stop / uninstall

Stop:

```bash
sudo systemctl stop mtproto-proxy
```

Uninstall:

```bash
sudo systemctl stop mtproto-proxy
sudo systemctl disable mtproto-proxy
sudo make uninstall
```

Logs can be found at

```
/var/log/mtproto-proxy/application.log
```

Settings
--------

All possible documanted configuration options could be found
in `src/mtproto_proxy.app.src`. Do not edit this file!

To change configuration, edit `config/prod-sys.config`:

Comments in this file start from `%%`.
Default port is 1443 and default secret is `d0d6e111bada5511fcce9584deadbeef`.

Secret key and proxy URL will be printed on start.

The easiest way to update config right now is to edit `config/prod-sys.config`
and then re-install proxy by

```bash
sudo make uninstall && make && sudo make install
```

There are other ways as well. It's even possible to update configuration options
without service restart / without downtime, but it's a bit trickier.

### Change default port / secret / ad tag

To change default settings, change `mtproto_proxy` section of `prod-sys.config` as:

```erlang
 {mtproto_proxy,
  %% see src/mtproto_proxy.app.src for examples.
  %% DO NOT EDIT src/mtproto_proxy.app.src!!!
  [
   {ports,
    [#{name => mtp_handler1,
       listen_ip => "0.0.0.0",
       port => 1443,
       secret => <<"d0d6e111bada5511fcce9584deadbeef">>,
       tag => <<"dcbe8f1493fa4cd9ab300891c0b5b326">>}
    ]}
   ]},

 {lager,
<...>
```
(so, remove `%%`s) and replace `port` / `secret` / `tag` with yours.

### Listen on multiple ports / IPs

You can start proxy on many IP addresses or ports with different secrets/ad tags.
To do so, just add more configs to `ports` section, separated by comma, eg:

```erlang
 {mtproto_proxy,
  %% see src/mtproto_proxy.app.src for examples.
  %% DO NOT EDIT src/mtproto_proxy.app.src!!!
  [
   {ports,
    [#{name => mtp_handler_1,
       listen_ip => "0.0.0.0",
       port => 1443,
       secret => <<"d0d6e111bada5511fcce9584deadbeef">>,
       tag => <<"dcbe8f1493fa4cd9ab300891c0b5b326">>},
     #{name => mtp_handler_2,
       listen_ip => "0.0.0.0",
       port => 2443,
       secret => <<"100000000000000000000000000000001">>,
       tag => <<"cf8e6baff125ed5f661a761e69567711">>}
    ]}
   ]},

 {lager,
<...>
```

Each section should have unique `name`!

### Only allow connections with 'dd'-secrets

It might be useful in Iran, where proxies are detected by DPI.
You should disable all protocols other than `mtp_secure` by providing `allowed_protocols` option:

```erlang
  {mtproto_proxy,
   [
    {allowed_protocols, [mtp_secure]},
    {ports,
     [#{name => mtp_handler_1,
      <..>
```

### Swap file (recommended)

Example: 4G = 4 * 1024 = 4096M

```bash
fallocate -l 4096M /root/swapfile && chmod 600 /root/swapfile && mkswap /root/swapfile && swapon /root/swapfile

echo "/root/swapfile none swap sw 0 0" >> /etc/fstab && reboot
```

Helpers
-------

Number of connections

```erlang
/opt/mtp_proxy/bin/mtp_proxy eval 'lists:sum([proplists:get_value(all_connections, L) || {_, L} <- ranch:info()]).'
```
