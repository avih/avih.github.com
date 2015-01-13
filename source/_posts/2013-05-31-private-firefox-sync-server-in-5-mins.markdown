---
layout: post
title: "Private Firefox Sync server in 5 minutes"
date: 2013-05-31 04:39
comments: true
categories: 
---

Well, theoretically in 5 minutes.

The official python server is [there](https://docs.services.mozilla.com/howtos/run-sync.html). The instructions on this page are for setting up a small 3rd party weave (Firefox sync) PHP server which is compatible with current Firefox 21 and nightlies as far as I can tell, and which I started using on my own systems as a replacement for the Mozilla sync server.

I'm not associated with this server in any way, and I'm *not* a security expert. **Use at your own risk**.

##The short version - with sqlite:

1. Have a web server with HTTPS and php5+sqlite (if self-signed cert, make sure to permanently accept it before Firefox sync setup).
2. Create a directory for weave (e.g. `/var/www/weave`), and put the files from [this repository](https://github.com/balu-/FSyncMS). Make sure the web server has write access to it (e.g. `sudo chown www-data /var/www/weave`).
3. Browse to `https://your.domain.com/weave/index.php` (sqlite selected by default), click `OK`.
4. **The server is now ready and operational**. Sync URL is `https://your.domain.com/weave/index.php/` (**note** the trailing '/').
5. Once an account was created after setting up sync from Firefox, you can disable further new accounts registrations at `settings.php` (new pairings with existing accounts will still work).
6. If using sqlite, make sure that `weave_db` is not accessible from outside (e.g. using `.htaccess`).
7. If required, to reset the server and delete the accounts: delete `weave_db` and `settings.php` at the weave directory (and go to step 3).
8. Migration to a new server: Unlink all devices (Tools>Options>Sync>Unlink), setup sync on one device with the new server, pair the other devices normally. If you don't wish to share all items (Addons/Passwords/etc), make sure to click "Sync Options" at the setup/pair dialogs since it resets to default more than expected.
<!-- more -->

##The long version
And other gotchas as encountered with Ubuntu 12.04 LTS. All notes on weave security were gathered from irc.mozilla.org#sync, and I hope I got them right. Thanks to *telliott*, *rnewman* and *witheld* for their help and patience.

1. Have a web server with HTTPS and php5+sqlite support.
- HTTPS is not mandatory for Firefox sync, but highly recommended. Sync sends the account password in plain text to the server. This password can NOT be used to decrypt the synced data at the DB, but it does allow an attacker to login to this sync account and make modifications such as delete elements at the DB for this account. HTTPS prevents password snooping and further attacks on the server/DB.
- Using self-signed certificate (snakeoil) will work for sync. However:
  - Any Firefox client which uses this server for sync should first permanently approve the certificate (e.g. by browsing to `https://your.domain.com` and adding the exception), preferably also verifying that the signature matches the server's.
  - Instructions for setup of [Apache2 + HTTPS with self-signed certificate](http://d.klwe.info/ubuntu-12-04-setting-up-apache2-and-ssl-with-self-signed-certificate/).
  - Self-signed certificate for sync apparently [doesn't always work on Firefox for Android](https://github.com/balu-/FSyncMS/issues/3).
- If required, add sqlite support to php. E.g. `sudo apt-get install php5-sqlite`.

2. Create a directory for weave (e.g. `/var/www/weave`), and put the files from [this repo](https://github.com/balu-/FSyncMS)
 - By default, if using sqlite, the DB will be created at the same directory with file name `weave_db`, and therefore the web-server will need write access to that dir. Also, the automatic setup will create a `settings.php` file at the same directory, which also requires write access (e.g. `sudo chown www-data /var/www/weave`).
 - If using manual setup (manually creating `settings.php`): If setting the sqlite DB file path elsewhere, or if using mysql, there's no need for that dir to have write access (but still need write access to the sqlite DB file if using sqlite).
 
3. Browse to `https://your.domain.com/weave/index.php` , select sqlite and press `OK` (the other option is mysql)
- Once OK is clicked, a new file `settings.php` will be created which allows some configuration (disable new accounts registration, sqlite db file path, mysql credentials).
- If sqlite was selected, the DB file will be created at the same directory as `weave_db`.

4. You can now setup Sync in firefox: choose a custom server and enter the URL (note the trailing '/'): `https://your.domain.com/weave/index.php/`
- To pair new devices, follow the normal procedure (no need to manually enter the custom URL on new devices - just use the 12 chars code and it'll work).

5. Note that the server is now public and anyone can create a new account on it by setting up Firefox sync with this server.
- Edit `settings.php` if you wish to disable further new accounts registration (this doesn't prevent pairing new devices on existing accounts).

6. If using sqlite with default DB file path, then the DB file will be accessible via the web (`https://your.domain.com/weave/weave_db`). While the data at the DB is encrypted (and the account login password cannot be used to decrypt the data), it's still highly recommended to make it inaccessible from the outside.
- I used `.htaccess` to prevent access to everything but `index.php` at that directory, which works:

``` c .htaccess to block everything except index.php
    Order Deny,Allow
    Deny from all
    Allow from 127.0.0.1
    <Files index.php>
        Order Allow,Deny
        Allow from all
    </Files>
``` 
- If your Apache server doesn't seem to read the `.htaccess` file, you might need to enable it, e.g. by adding to `/etc/apache2/httpd.conf` (empty by default):

``` c Enable .htaccess with Apache2
<Directory /var/www/weave>
AllowOverride All
</Directory>
``` 
And restarting apache. E.g. `sudo service apache2 restart`

- Alternatively (haven't tried myself): move `weave_db` to a dir outside of wwwroot and edit `settings.php` accordingly.
- I haven't tried using mysql with this server.

7. If required, to reset the server and delete the accounts
- Delete `weave_db` and `settings.php` at the weave directory (and go to step 3).

8. Migration to a new server
- Unlink all devices (Tools>Options>Sync>Unlink).
- Setup sync on one device with the new server.
- Pair the other devices normally.
- **If you don't wish to share all items** (Addons/Passwords/etc), make sure to click "Sync Options" at the setup/pair dialogs since it resets to default more than expected.

Good luck!
