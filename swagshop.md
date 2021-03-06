---
layout: default
---

## Swagshop
###### Retired September 19
![](https://yaboygmoney.github.io/htb/images/swagshop/machineImage.png)

Pretty easy box. Web exploit for initial foothold and some classic binary ninja magic for privesc.

First things first is nmap of course:

```bash
nmap -sV -O -o nmap.results -p 1-65535 10.10.10.140
```

![](https://yaboygmoney.github.io/htb/images/swagshop/nmap.JPG)

SSH and HTTP. SSH is rarely ever the vector, so let's do some web exploitation. Can't exploit what you don't know, so further enumeration is required.

I ran a couple of gobusters (forgot something) first and got some results:

![](https://yaboygmoney.github.io/htb/images/swagshop/gobuster.JPG)

None of this looks particularly crazy, but we can look around. The ```/downloader``` one gives us the most useful direction.

![](https://yaboygmoney.github.io/htb/images/swagshop/version.JPG)

We can see it's running Magento Connect Manager version 1.9.0.0.

My second gobuster, I tacked on my usual ```-x php``` to also try to find links ending with a php extension. 

![](https://yaboygmoney.github.io/htb/images/swagshop/gobusterphp.JPG)

That gives us a little more, an index.php.

I ran yet another gobuster, with the index.php/ as the root URL and was met with wildcard responses, ending gobuster's effectiveness.

![](https://yaboygmoney.github.io/htb/images/swagshop/wildcard.JPG)

Reading up on Magento, I found that the admin panel is typically stored at /admin. I went to 10.10.10.140/admin (thinking somehow I would have better luck than gobuster) and was met with a 404. Then I remembered that there were a bunch of wildcard responses after index.php, so I tacked it on the end of there and found the admin panel.

![](https://yaboygmoney.github.io/htb/images/swagshop/admin.JPG)

At this point, I don't have any creds. I looked, didn't find anything. Proper Google-Fu pointed me to 
[this exploit](https://packetstormsecurity.com/files/133327/Magento-Add-Administrator-Account.html).

I'll be honest, the first time I got in through this web panel, I just used the default creds the script made because I figured someone had already compromised it.
```
username: forme
password: forme
```

That worked. Later on though, after one of the millions of box crashes, I switched it up and launched my own. We've got to point the ```target``` variable to the proper URL. Additionally, I change the creds to be:
```
username: yaboy
password: gmoney
```

![](https://yaboygmoney.github.io/htb/images/swagshop/exploitEdit.JPG)

```bash
chmod +x exploit.py
``` 
and let 'er rip.

![](https://yaboygmoney.github.io/htb/images/swagshop/exploit.JPG)

Head back to the admin panel and login. Easy mode. We land on the Magento Admin panel. I poke around and find a filesystem web GUI. Figured that was a good place to stick a reverse shell.

The path I went to on the admin panel was System > Filesystem > IDE > includes > config.php. I chose to edit that page because it's not going to be used, and will still eval my php reverse shell to get me access.

![](https://yaboygmoney.github.io/htb/images/swagshop/reverseShell.JPG)

I'm a big fan of the [Pentest Monkey PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php).
I axed the config.php, loaded the shell up, and set up my listener:
```bash 
nc -nvlp 1234
```

Then I navigated to http://10.10.10.140/includes/config.php and caught my shell and immediately got the user flag:

![](https://yaboygmoney.github.io/htb/images/swagshop/userFlag.jpg)

I start poking around doing my usual privesc checks. We can see in the sudoers file that the user www-data (our current user) can use vi with sudo privileges. 
These privileges are limited to the /usr/bin/vi binary and the files in /var/www/html/. Anywhere else, I cannot sudo.

![](https://yaboygmoney.github.io/htb/images/swagshop/sudoers.JPG)

Here, I learned a lesson in explicitness. From the /var/www/html directory, I try straight up
```bash
sudo vi junk.txt
```

![](https://yaboygmoney.github.io/htb/images/swagshop/sudoFail.JPG)

Asks for the password = did it wrong. The sudoers file says ```NOPASSWD``` so I shouldn't need it. I try specifying the user ```root```:

![](https://yaboygmoney.github.io/htb/images/swagshop/sudoFail2.JPG)

Still wants a password. So I get really explicit. [DMX-level](https://www.youtube.com/watch?v=6CqXgs-7ico) explicit. I use a literal path to the binary and the filename.

![](https://yaboygmoney.github.io/htb/images/swagshop/sudo.JPG)

And I finally run vi. From here, we can run commands from within this instance of vi which is running as root. As such, any command we run will also run under the context of root level permissions. So I call a shell by typing
```bash
:!/bin/sh
```

from vi.

![](https://yaboygmoney.github.io/htb/images/swagshop/escape.JPG)

I end up with a shell that has root skills. Cat the flag and dip.

![](https://yaboygmoney.github.io/htb/images/swagshop/rootFlag2.jpg)
