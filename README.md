# Expose ports to clearnet with Cloudflare without exposing your IP to the whole world

## Disclaimer
This is my first written guide, so I hope it is it's written in a way everybody can learn from this. 
If you want to improve this guide, please make a pull request or an issue.
For other questions: DM [@bufo24](t.me/bufo24) on Telegram

# Requirements
- Domain in [Cloudflare](cloudflare.com)
- Router with port forwarding (option without port forwarding coming soon)

# What can you do after this tutorial?
- Connect Zeus (or any other mobile wallet) to your node within seconds
- View your own mempool over clearnet, like mempool.mydomain.com
- Seemsless integrations with LNBits, like Boltcard

# Schema
```mermaid
sequenceDiagram
Client ->> Cloudflare: [GET] mempool.mydomain.com
Cloudflare ->> Nginx: [GET] mempool.mydomain.com
Nginx->>Node: [GET] localhost:3006
Node->>Nginx: mempool.space webpage
Nginx->>Cloudflare: mempool.space webpage
Cloudflare->>Client: mempool.space webpage
Client ->> Cloudflare: [GET] lnd.mydomain.com
Cloudflare ->> Nginx: [GET] lnd.mydomain.com
Nginx->>Node: [GET] localhost:8080
Node->>Nginx: LND response
Nginx->>Cloudflare: LND response
Cloudflare->>Client: LND response
```

# Steps
- Install Nginx on your node.
	- `apt install nginx -y`
- Get SSL certificates from Cloudflare to Nginx.
	- Log into Cloudflare, select your domain and select SSL/TLS > Origin Server
	- Click "Create Certificate"
	- Keep "Generate private key and CSR with Cloudflare" selected
	- Make sure domain.com and *.domain.com are both includes in "Hostnames".
	- Set "Certificate Validity" to your own liking.
	- Click "Create".
	- Create and open a file in your Nginx directory to store the SSL key and certificate: 
		`sudo nano /etc/nginx/domain.com.crt` 
		and paste the SSL certificate from Cloudflare into this file. It should start with:
		`-----BEGIN CERTIFICATE-----`
		to save and exit: `CTRL + X` and `Y` to confirm.
		The same thing goes for the SSL key: 
		`sudo nano /etc/nginx/domain.com.key`
		paste the SSL key from Cloudflare, this one should start with:
		`-----BEGIN PRIVATE KEY-----`
		And again, do `CTRL + X` and `Y` to confirm.
- Edit Nginx config
	- Open the Nginx config with: `sudo nano /etc/nginx/nginx.conf`
	- Scroll down to the bottom of the `html {} ` and add `server {}` within the html brackets.
	- Add the following lines within the server brackets:
```
		listen 443 ssl http2;
		listen [::]:443 ssl http2;
		server_name domain.com *.domain.com;
		ssl_certificate domain.com.crt;
		ssl_certificate_key domain.com.key;
		ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
		ssl_ciphers HIGH:!aNULL:!MD5;
```
- Add subdomains to Nginx config:
	- Underneath the lines describe above, we can specify our wanted subdomains and disered location:
```
location / {
	if ($http_host = 'mempool.domain.com') {
		proxy_pass http://localhost:3006;
	}
	if ($http_host = 'lnd.domain.com') {
		proxy_pass https://localhost:8080;
	}
	if ($http_host = 'lnbits.domain.com') {
		proxy_pass http://localhost:3007;
	}
	# needed for websocket:
	proxy_set_header Upgrade $http_upgrade;
	proxy_set_header Connection "Upgrade";
	proxy_set_header Host $host;
}
```
- Reload Nginx: `nginx -s reload`
	- If this gives an error, you are probably missing a semicolon or a bracket.
- Port forward your router.
	- Set 443 as internal and external port and provide the local IP of your node.
	- You can find your local IP of your node by `ifconfig eth0`  for ethernet or `ifconfig wlan0` for WiFi.
- Add DNS record to Cloudflare.
	- We have to provide your external address of your home network to Cloudflare so they can make requests to your node. You can find your external IP address by going to `myip.wtf` in your browser or doing `curl myip.wtf/json` in your terminal.
	- In Cloudflare, go to your domain and select DNS.
	- Add an A record: `*` as name and enter your external IP at IPv4 address, and make sure the Proxy is enabled.

Now your subdomains should be accessible from your browser.
To let Zeus work with your subdomain, just change your TOR address to your subdomain like lnd.domain.com and disable TOR.

If you think this guide could be better, please make a PR!
For questions: DM [@bufo24](t.me/bufo24) on Telegram
