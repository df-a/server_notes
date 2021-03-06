Here are various notes I've made for creating a server. These rules are for Ubuntu running
nginx. I have some notes for Apache that I may add at another time.




## SECURING SERVER


1. Create Droplet 

2. ```SSH into server - ssh root@{ip-address}```

3. ```sudo apt-get update; sudo apt-get upgrade -y```

4. ```sudo apt-get install fail2ban -y```

5. ```sudo adduser {username}```
    - fill in user info.

6. ```sudo visudo```
    - under ```"root    ALL=(ALL:ALL) ALL"``` add ```"{username}    ALL=(ALL:ALL) ALL"``` - don't include quotations
        - can remove root, but from what I've found, it doesn't matter - but it may, IDK.
        
7. exit server

8. create new ssh key
    ```ssh-keygen -t rsa -b 4096```

    - FWIW, RSA can be cracked by a quantum computer, if you worry about that ish.

9. Copy key to server.
    - ```ssh-copy-id -i ~/.ssh/{name of key}.pub “username@ip_address”```

10. SSH into your server (not root)

11. ```(nano | vim | vi) /etc/ssh/sshd_config```
    - change ssh ```port``` from 22 (this will just keep the logs a little cleaner)
    - change ```PermitRootLogin``` to 'no' - don't include quotations
    - change ```PasswordAuthentication``` to 'no' - don't include quotations
    - exit/save file and run: ```sudo service ssh restart```

12. Add fire wall rules - with ex. rules
    - ```sudo ufw allow 80``` - http
    - ```sudo ufw allow 443``` - https
    - ```sudo ufw allow 25``` -smtp
    - ```sudo ufw allow 22/{your new ssh port}``` -ssh
    - ```sudo ufw allow from 15.15.15.0/24 to any port 5432``` - postgres from remote server, change subnet
    - ```sudo ufw enable```


13. Set up pre firewall rules (this will ghost the server - ping packets will be dropped)
    - ```(nano | vim | vi) /etc/ufw/before.rules```
    - DROP everything related to all ICMP/Pinging - I believe there are 8-10 of these in total
        
        - these are usually the fourth and fifth blocks.

        - these two blocks have ```ICMP``` mentioned in the comments above them.

    - exit/save file and run: ```sudo ufw reload ```



## Add SSL to site - letsencrypt  

1. ```sudo apt-get install letsencrypt``` 

2. ```sudo letsencrypt certonly --standalone --rsa-key-size 4096 --force-renew -d example.com -d www.example.com```
    - nginx needs to be turned-off

        -```sudo service nginx stop```

3. ```sudo crontab -e```
    - may need to choose editor

4. at the bottom of the file add: ```30 2 * * 1 sudo /usr/bin/letsencrypt renew --rsa-key-size 4096 >> /home/${username}/le-renew.log```
    - this will regen a key every monday at 2:30 AM.
    - you can try /var/log, but I've run into permission issues when using sudo
        - you could set this in the root cron though, just remove sudo

5. create DH key:
    - ```sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 3072```

## SECURE SSL

I may upload the actual nginx file later, but for now, I'll just add the neccessary parts.

```
 Vary Upgrade-Insecure-Requests;
 add_header X-Content-Type-Options nosniff;
 add_header X-Xss-Protection "1; mode=block";
 add_header Content-Security-Policy "default-src https: data: 'unsafe-inline' 'unsafe-eval'";
 add_header X-Frame-Options "SAMEORIGIN";
 server_tokens off;

 ssl on;
 ssl_certificate /etc/letsencrypt/live/{your site}/fullchain.pem;
 ssl_certificate_key /etc/letsencrypt/live/{your site}/privkey.pem;
 ssl_dhparam /etc/ssl/certs/dhparam.pem;

 ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

 ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';

 ssl_prefer_server_ciphers on;
 ssl_ecdh_curve secp384r1;
 ssl_session_cache shared:SSL:10m;
 ssl_session_tickets off;
 add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; preload" ;
 ssl_session_timeout 2h;
 ssl_stapling on;
 ssl_stapling_verify on;
```

exit and restart nginx - `sudo service nginx restart`

Double check everything on [ssllabs.com](https://ssllabs.com) and [securityheaders.io](https://securityheaders.io)

## Tripwire
This is honestly a bit of a lengthy process. However, Justin Ellingwood has written a [great piece over at DO on it.](https://www.digitalocean.com/community/tutorials/how-to-use-tripwire-to-detect-server-intrusions-on-an-ubuntu-vps)

## Configure postfix
To change postfix from sending from user@domain-localhost, I found the best way is to change the hostname for Ubuntu

1. hostname yourdomain.com

2. ``sudo vi /etc/hostname`` and change it to yourdomain.com

3. You will have to exit your droplet and ssh back into it.

Your hostname will have changed

*** Note
With the above you will receive an email with your super user account. This isn't ideal. However, you can set up postfix to use an email relay to send stuff from Google instead, see the linode link at the bottom for instructions on how to do so.

## Get email notifications when a user/root is logged into
0. This is assuming that the tripwire has been set, forcing you to have downloaded a mail client - in the case above, it would be postfix.

1. Let's start with root

    - ```sudo su```

    - ```vi /root/.bashrc```

    - At the bottom of the file add: ```mail -s "Alert: Root Access from `who | cut -d'(' -f2 | cut -d')' -f1`" your_email@domain.com```

2. Your user

    - `vi /home/${username}/.bashrc`

    - At the bottom of the file add: ```echo 'ALERT - Root Shell Access (ServerName) on:' `date` `who` | mail -s "Alert: Root Access from `who | cut -d'(' -f2 | cut -d')' -f1`" your_email@domain.com```

*** Note
With the above you will receive an email with your super user account. This isn't ideal. However, you can set up postfix to use an email relay to send stuff from Google instead, see the linode link at the bottom for instructions on how to do so.



## Create an SSH config file - OSX

`touch ~/.ssh/config`

```sudo (nano | vim | vi) ~/.ssh/config```


Host {name} - ex. Personal

HostName {your ip address} - ex. 104.159.103.195 

Port {your ssh port} - ex. 22

User {your username} - ex. root (don't use root!)

IdentityFile ~/.ssh/example.key - the private key for the server

Example: 
```
Host Personal
HostName 104.159.103.195 
Port 2222
User Jack
IdentityFile ~/.ssh/personal
```

## Add SWAP
* Note SWAPs can be harmful to older SSDs...

1. ```sudo fallocate -l 4G /swapfile```
2. ```ls -lh /swapfile```
3. ```sudo chmod 600 /swapfile```
4. ```ls -lh /swapfile```
5. ```sudo mkswap /swapfile```
6. ```sudo swapon /swapfile```
7. ```sudo nano /etc/fstab```
8. Add to the bottom of the file:
   ``` - /swapfile   none    swap    sw    0   0```
9. ```sudo sysctl vm.swappiness=10```
10. ```sudo sysctl vm.vfs_cache_pressure=50```
11. ```sudo nano /etc/sysctl.conf```
12. Add to the bottom:
    - ```vm.swappiness=10```
    - ```vm.vfs_cache_pressure = 50```


## Increase PHP and Nginx memory sizes:

php5 location = /etc/php5/fpm/php.ini

1. sudo (nano | vim | vi ) /etc/php7.0/fpm/php.ini
    - ```upload_max_filesize = 50M```
    - ```post_max_size = 50M```
    - ```max_execution_time = 120```
    - ```max_input_time = 120```
    - ```memory_limit = 64M```

2. sudo (nano | vim | vi) /etc/nginx/nginx.conf
    - ```client_max_body_size 100M;```

## Secure PHP 
1. Open php.ini  

    - ```sudo (nano | vim | vi) /etc/php/7.0/fpm/php.ini``` 

2. uncomment ```cgi.fix_pathinfo=1``` - remove the semi-colon in front of it, and change the value to 0

    - ```cgi.fix_pathinfo=0```

3. Remove X-Powered-by
    
    - ```expose_php = 0```

4. Restart php: ```sudo systemctl restart php7.0-fpm```


## Cloudflare with S3
 1. Create a bucket with the name of your domain, with a subdomain of cdn - cdn.example.com
    - configure the bucket policy as well as the cors to allow the domain to access

 2. Assuming you have already pointed your dns HostName at cloudflare, add the cname cdn pointing to cdn.example.com.s3.amazonaws.com

 3. Change the ssl on cloudflare to flexible

 4. change the cloud to orange for the cdn cname

 ** MAY NEED TO DO

 5. You get a redirect loop:
     - this is due to the flexible ssl. Go to page rules on cloudflare and add a 'always use https' rule for your domain
     - in your /etc/nginx/sites-availble/default change the redirect look like:
        ```
        server_name    example.com www.example.com;

        if ($http_x_forwarded_proto = "http") {
     
        return 301 https://$server_name$request_uri;
        
        }
        ```
 6. If you get a 500 from the server:
    - you need to change the try_files in the /etc/nginx/sites-availble/default as it may be due to a redirect:
       -change ```try_files $uri $uri/ index.html``` to  ```try_files $uri $uri/ =404```

## Closing notes:
There are other practices that need to be followed to ensure security - always using sftp, using different keys for different servers, always running `sudo apt-get update; sudo apt-get upgrade -y` when logging into the server, not reusing the same passwords - but in the end, if someone finds a zero day in the hypervisor - in this instance DO's - none of this really matters.

Also, in the above, having the email sent showing the super user's account name isn't ideal. I currently haven't figured out a way to 'spoof' the name to just show "mail" or something. However, relaying can be done to use something like gmail - you could then use an email alias through gmail.

[https://www.linode.com/docs/email/postfix/postfix-smtp-debian7](https://www.linode.com/docs/email/postfix/postfix-smtp-debian7)

## Extras
Here's a great video that goes into some of the stuff listed above while setting up a DO server: 

[https://www.youtube.com/watch?v=YZzhIIJmlE0](https://www.youtube.com/watch?v=YZzhIIJmlE0)

## License 
    These notes are released under MIT.