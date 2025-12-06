# Home_Assistant_Valetudo_SSL_Reverse_Proxy
How to access valetudo inside home assistant via https using self signed multi domain certificates
## Motivation
There are many different reasons why somebody wants to access home assistant via https/SSL e.g. if you want for whatever reason to access your mobile phones microphone inside homeassistant, it is mandatory to have SSL encryption activated.
When other entitys, such as Valetudo (which naturally run without SSL), are embedded inside home assistant (e.g. via a webpage card), those entities won't be accessible anymore, since they are not encrpyted. So in this scenario one could either use their microphone or access Valetudo but not both.
This guide describes all necessary steps on how to enable both, SSL encryption while also being able to access Valetudo. Also this method won't break unencrypted access, meaning home assistant and valuetudo will still be accessible without SSL/HTTPS.
On top, no external domain is needed - so from that perspective there wont be any additional costs since the whole system runs locally. If somebody wants to access their homeassistant system remotly there are alternatives, such as twingate, that will still work with this guide.

# SSL/HTTPS Show Valetudo in Homeassistant: Self signed multi domain certificate
## My setup
I have two Raspberry PIs, one running homeassitant OS and the other one was more or less lying around (besides running twingate, which isn't needed for this tutorial) and came in handy.
192.168.2.44 RaspberryPi running home assistant OS
192.168.2.51 Another RaspberryPi running Raspberry OS

## Preparation
First you will need to check your home assistant hostname.In Homeassistant go to settings->network->hostname: In my case thats "homeassistant"

<img width="340" height="326" alt="image" src="https://github.com/user-attachments/assets/72701781-b793-4ece-8f32-28727db80b08" />

Next check the hostname of the raspberry pi running Raspberry OS: 
~~~
sudo raspi-config
~~~
Then navigate to System Options and then hostname. In my case the hostname is "raspberrypi"

## Create Certificate
Connect via ssh to your second raspberry pi (or run on any other unix machine) and run the following commands:

Create Root Key:
~~~
sudo openssl genrsa -des3 -out rootCA.key 4096
~~~

Create Root Certificate:
~~~
sudo openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.pem
~~~
Add the following content, then exit via STRG+X -> y
~~~
Country Name (2 letter code) [AU]:DE
State or Province Name (full name) [Some-State]:BW
Locality Name (eg, city) []:YourCity
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Kueck
Organizational Unit Name (eg, section) []:--
Common Name (e.g. server FQDN or YOUR name) []:*.local
Email Address []:yourmail@provider.com
~~~

Now create rootCA.csr.cnf:
~~~
sudo nano rootCA.csr.cnf
~~~
~~~
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C=DE
ST=BW
L=YourCity
O=Kueck
OU=---
emailAddress=yourmail@provider.com
CN = *.local
~~~

Next create v3.ext:
~~~
sudo nano v3.ext
~~~
~~~
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
extendedKeyUsage=serverAuth

[alt_names]
DNS.1 = homeassistant.local
IP.1 = 192.168.2.44
DNS.2 = raspberrypi.local
IP.2 = 192.168.2.51
~~~

Create Certificate Signing Request
~~~
sudo su -c 'openssl req -new -sha256 -nodes -out hassio.csr -newkey rsa:2048 -keyout hassio.key -config <( cat rootCA.csr.cnf )'
~~~

Now sign the SSL certificate:
~~~
sudo openssl x509 -req -in hassio.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out hassio.crt -days 3650 -sha256 -extfile v3.ext 
~~~

The last steps are renaming & changing permissions:

Rename certificate:
~~~
sudo mv hassio.crt fullchain.pem
~~~
Rename private key:
~~~
sudo mv hassio.key privkey.pem
~~~
Change permissions:
~~~
sudo chmod 600 fullchain.pem privkey.pem
~~~

## Setup home assistant https access via reverse proxy NGINX (and install terminal)
On homeassistant install Advanced SSH & Web Terminal addon.
<img width="859" height="122" alt="image" src="https://github.com/user-attachments/assets/a6bf9614-f48c-4e26-a563-3326b16b5f44" />

In the plugin configure ssh access (user root and choose password). Activate sftp, save and start addon.

Next install the nginx addon.
Copy eg. via WINSCP certificates to the homeassistant host (folder may vary by homeassistant installation method, this is for home assistant supervised): 

/usr/share/hassio/ssl/fullchain.pem

/usr/share/hassio/ssl/privkey.pem

For Homeassistant OS installation method copy the files to:

/ssl/privkey.pem 

/ssl/fullchain.pem
~~~
sudo cp fullchain.pem ./ssl/fullchain.pem
sudo cp privkey.pem ./ssl/privkey.pem
~~~

Reminder: If WINSCP won't let you copy files to the home assistant system, click on advanced settings

<img width="602" height="413" alt="image" src="https://github.com/user-attachments/assets/6077f270-6295-4b54-a5f6-d9270f297214" />

and select "sudo su -" in  SCP/Shell:

<img width="551" height="457" alt="image" src="https://github.com/user-attachments/assets/7ae238f4-d5e7-453f-82f7-602f87a59d1f" />

Configure nginx addon as follows:

<img width="469" height="520" alt="image" src="https://github.com/user-attachments/assets/47e54f50-2b8b-4c11-9726-23a82d73cc01" />

## Setup Valetudo https access via reverse proxy NGINX
Ideally take separate host (e.g. a raspberry pi) to host a second NGINX reverse proxy that will redirect all incoming https traffic towards the valetudo instance.
~~~
sudo apt-get install nginx 
sudo systemctl start nginx 
~~~

Now configure the reverse proxy:
~~~
cd /etc/nginx/
sudo nano ./sites-enabled/reverse-proxy.conf
~~~
add the following content and exit via STRG+X
~~~
server {
    listen 443 ssl http2;
    server_name homeassistant.local homeassistant*;
    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    location / {
        proxy_pass http://192.168.2.42;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
~~~
I did also remove all content from the default file in the same folder.

Next test your configuration
~~~
sudo nginx -t
~~~
The prompt should tell:
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

Now restart nginx service:
~~~
sudo systemctl restart nginx
~~~

Valetudo should now be accessible via the hostname https://raspberrypi.local

## Android: Homeassistant Companion App
Copy rootCA.pem (created earlier) to your phone-
On your phone go to settings -> data protection and security -> More data protection and security ->
Encryption and login data -> install certificate -> CA-Certificate -> choose the rootCA.pem

<img width="220" height="492" alt="image" src="https://github.com/user-attachments/assets/d44f9fe4-2eb7-4396-8800-7fcbf06a1188" />

<img width="199" height="446" alt="image" src="https://github.com/user-attachments/assets/a1ffbc1e-42af-44de-9f67-8a0e60f3c896" />

Next, configure Homeassistant Companion App as follows

<img width="241" height="539" alt="image" src="https://github.com/user-attachments/assets/289228d8-da31-491b-b031-5cf65e7902a0" />

## iOS: Homeassistant Companion App
Copy rootCA.pem to your phone
Go to files and click on rootCA.pem. Load profile. Go to settings. Right under the user select profile and install it. Search for certificates and enable the newly installed certificate
Next, configure Homeassistant Companion App as follows
<img width="241" height="539" alt="image" src="https://github.com/user-attachments/assets/289228d8-da31-491b-b031-5cf65e7902a0" />

## Windows: Configure Chrome
Configure Chrome (on Desktop)
Go to the certificate manager (chrome://certificate-manager/localcerts) and install your certificate “rootCA.pem” under the section user defined.
Now homeassistant should be accessible without a warning.

 <img width="945" height="483" alt="image" src="https://github.com/user-attachments/assets/49eb8ad0-40e1-4569-bc4c-ad122ca8dbbc" />

## Putting everything together
In homeassistant add a website tile, chose the separate NGINX server hostename: https://raspberrypi.local

<img width="945" height="363" alt="image" src="https://github.com/user-attachments/assets/e191269a-563f-4063-9d5e-2b824b9b8eac" />

Now Valetudo should be accessible inside homeassistant:

<img width="443" height="392" alt="image" src="https://github.com/user-attachments/assets/3fe54a18-f0b8-438f-9cf9-402b70a39d9d" />

And also via the companion app:

<img width="283" height="632" alt="image" src="https://github.com/user-attachments/assets/f3fa695b-f265-4783-9eee-6402a0a1b5a1" />

Home assistant and Valetudo will still be accessible via http.

## Bonus Twingate configuration
If you want to access this setup remotly (from outside your WLAN/LAN), one possibility is to install twingate on the second raspberry pi running Raspberry OS.
A tutorial on how to install twingate on a raspberry pi can be found here: https://youtu.be/IYmXPF3XUwo?si=4Y7z4IXgol_eoOkB

After setup, Twingate needs to be reconfigured in order to access homeassistant / valetudo remotely via SSL.

Add SSL for Homeassistant:

<img width="501" height="396" alt="image" src="https://github.com/user-attachments/assets/beb7edd6-5acf-4cba-97aa-c927773bafc6" />

Add SSL for Valetudo reverse proxy:

<img width="485" height="385" alt="image" src="https://github.com/user-attachments/assets/0ac5ff2e-3667-482e-9e3e-5b64cc7c6351" />


$ 

