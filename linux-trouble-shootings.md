# Nginx troubleshootings - old web contents after aeployment & newly deployed web app (Nginx) now showing up
date : May 17th 2026
scenario: developer deployed new code but website still showings old page contents.

---

## The complaint
Developer says:
> "I just deployed new code but the website is showing an old version. I restarted my app but nginx is still showing the old page."

---

## whats learned about Nginx

Nginx serves files from a folder on disk called the web root. On RH the default location is:

```bash
/usr/share/nginx/html/
```

when someone visits the server IP in a browser, nginx goes to that folder and serve whatever index.html is in there.
the web root is controlled in the nginx config file:

```bash
/etc/nginx/nginx.conf
```

look for the root directive inside the server block:

```bash
server {
    listen       80;
    server_name  _;
    root         /usr/share/nginx/html;
}
```
the root line is what tells nginx where to find files.

---

## 1st back up before touching anything

```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```
---

## 2nd create the developers app folder

in real deployments developers put their code in their own folder, not the nginx default. let's simulated this:

```
sudo mkdir -p /var/www/myapp
sudo touch index.html
nano index.html
<h1>New version deployed by the dev!</h1>
```

verifying it's all there:

```
cat /var/www/myapp/index.html
```

---

## 3rd update Nginx config to point at the newly created folder

open the config:

```
sudo nano /etc/nginx/nginx.conf
```

find root directive and change:

```
root         /var/www/myapp;
```

note: every nginx directive must end with a semicolon. missing it will break.

---

## 4th test config before reloading

Note : never reload nginx without testing the config first:

```
sudo nginx -t
```

successful output looks like:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

if it fails it tells exactly which line has the problem. fix it and test again.
got this error:

```
nginx: [emerg] invalid number of arguments in "root" directive in /etc/nginx/nginx.conf:44
```

cause:  missing semicolon at the end of the root line. fixed and test pass.

---

## 5th reloading Nginx

reload to apply changes:

```
sudo systemctl reload nginx
```

note: reload applies new config without dropping existing connections. safer than restarting.

---

## 6th verifying in terminal with curl command

```
curl http://localhost
```

this method bypasses browser cache
if curl shows the old page, the problem is server side. If curl shows the new page but browser shows old, the problem was browser cache.

---

## problem 2 - SELinux keep blocking Nginx

after fixing the config, curl still showing the old default page. The config was correct but something else was blocking nginx.

in RH, SELinux adds a second layer of security on top of regular file permissions. even if chmod and chown are correct, SELinux could still block access.

check SELinux status:

```
sudo getenforce
```

if it says Enforcing, SELinux is actively blocking things.

checking the SELinux label on the folder:

```
ls -lZ /var/www/myapp/
```

the folder output showed:

```
unconfined_u:object_r:var_t:s0
```

the label was var_t. Nginx requires web content to have the label httpd_sys_content_t. without it, SELinux silently blocks nginx from readings the files.

---

## 7th fix SELinux context permanently

the wrong way (temporary fix, resets after reboot):

```
sudo chcon -R -t httpd_sys_content_t /var/www/myapp/
```

the right way (permanent, survives reboots):

```
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/myapp(/.*)?"
sudo restorecon -R -v /var/www/myapp/
```

what these do:
semanage fcontext: writes a permanent rule to the SELinux policy saying this path should always have httpd_sys_content_t label.
restorecon: applies the policy to the files right now without waiting for a reboot.

verifying the label changed:

```
ls -lZ /var/www/myapp/
```

now showng output:

```
unconfined_u:object_r:httpd_sys_content_t:s0
```

---

## 8th verifying everything working

test again with curl:

```
curl http://localhost
```

should return the new page contents.
test in browser and use hard refresh to ignore cache:

```
CTRL + SHIFT + R
```

note : normal refresh uses cached version. hard refresh forces browser to get fresh files from server.

---

## 9th report back to the Developer

message:
> "All fixes. Nginx was pointing at the wrong folder and SELinux needed to be configured to allow access to the new location. should be live now, give it a try and let me know if anything looks off."

---

## full command summaries

```
# back up config
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak

# edit config
sudo nano /etc/nginx/nginx.conf

# test config syntax
sudo nginx -t

# reload nginx
sudo systemctl reload nginx

# test in terminal
curl http://localhost

# check SELinux status
sudo getenforce

# check SELinux label on folder
ls -lZ /var/www/myapp/

# making SELinux permanently
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/myapp(/.*)?"
sudo restorecon -R -v /var/www/myapp/

# verifying label
ls -lZ /var/www/myapp/
```
flags
ls: List the files and folders.
-l: Use the long listing format. This shows permissions, owners, file size, and modification dates.
-Z: The star of the show. This tells ls to display the SELinux security labels (contexts) for every item.
---

## lesson notes

nginx -t is good tools to use. always test before reloading. it catches syntax errors without breaking anything yet.

SELinux is invisible until it blocks. In RHEL always check SELinux contexts when file permissions look correct but access is still being denied.

chcon is temporary quick fixs. semanage fcontext plus restorecon is permanent. always use the permanent fix on production servers so it will survive rebooting/restarting.
browser cache. always use curl to verify server side changes in terminal. hard refresh when browser shows old contents.
document what you changes and why/how. 
---

## each tools notes

```
nginx -t              test config syntax without restarting
systemctl reload      apply new config without dropping connections  
curl http://localhost  test server response without browser cache
getenforce            check if SELinux is enforcing
ls -lZ                show SELinux labels on files
semanage fcontext     write permanent SELinux policy for a path
restorecon            apply SELinux policy to files immediately
```

---

*Learned during on the job training — May 2026*