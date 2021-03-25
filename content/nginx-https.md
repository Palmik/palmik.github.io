+++
title = "How to handle HTTPS with Nginx"
date = 2021-03-26T15:30:00Z

[taxonomies]
categories = []
tags = ["SSL", "Nginx"]
+++

How to use Nginx to handle HTTPS and follow all the best practices?
<!-- more -->
<hr/>

What do we want to handle?
* Redirect `http://example.com` to `https://example.com`.
* Redirect `https://www.example.com` to `https://example.com`.
* Use only modern protocols.
* Use only strong ciphers.
* Use [content security policy (CSP)](https://en.wikipedia.org/wiki/Content_Security_Policy). 
* Use [HTTP strict transport policy (HSTS)](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security) which can prevent downgrade (`https` --> `http`) attacks.

Here's what the config would look like, we will unpack it section-by-section below.

```config
events {
}

http {
  # Handle https://example.com
  server {
    listen 443 ssl;
    server_name example.com;

    # Assuming you use Let's Encrypt for your SSL certificate.
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_session_timeout 1d;
    ssl_session_cache builtin:100 shared:SSL:10m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
    ssl_prefer_server_ciphers on;
    # curl https://ssl-config.mozilla.org/ffdhe2048.txt > /path/to/dhparam
    ssl_dhparam /path/to/dhparam;

    location / {
      add_header Content-Security-Policy "default-src 'none';connect-src 'self' www.google-analytics.com;manifest-src 'self';style-src 'self';script-src 'self' www.googletagmanager.com;img-src 'self' www.googletagmanager.com;frame-ancestors 'self';base-uri 'none';";
      add_header Strict-Transport-Security "max-age=15768000; includeSubdomains; preload";
      add_header X-Content-Type-Options nosniff;
      add_header X-Frame-Options SAMEORIGIN;
      add_header X-XSS-Protection "1; mode=block";

      proxy_set_header        Host $host;
      proxy_set_header        X-Real-IP $remote_addr;
      proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header        X-Forwarded-Proto $scheme;

      # Pass the request to your server.
      proxy_pass http://server:10080;
    }
  }
  
  # Redirect http://example.com and http://www.example.com to https://example.com
  server {
    listen 80;
    server_name www.example.com example.com;
    return 301 https://example.com$request_uri;
  }

  # Redirect https://www.example.com to https://example.com
  server {
    listen 443;
    server_name www.example.com;
   
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_session_timeout 1d;
    ssl_session_cache builtin:100 shared:SSL:10m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
    ssl_prefer_server_ciphers on;
    ssl_dhparam /path/to/dhparam;
    
    return 301 https://example.com$request_uri;
  }
}
```

### SSL Config

[The Nginx documentation](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_prefer_server_ciphers) is a great resource to underatand the different SSL config options.

* `ssl_session_cache`: Allows the resue of TLS sessions. 
* `ssl_session_timeout 1d`: Maximum recommended value from TLS RFC and from Nginx documentation. Higher value means potentially higher server-side resource use, higher chance that the same TLS session can be reused by the client (lower latency / resource use), very high value might affect forward secrecy. 
* `ssl_ciphers`: Sets the allowlist of ciphers to only "strong" options, very old clients that do not support any of the strong ciphers might not be able to connect.
* `ssl_prefer_server_ciphers off`: Lets the client choose the cipher from the allowlist. The client will likely choose the fastest cipher from the allowlist above.
* `ssl_dhparam`: Overrides the default Diffie-Helman parameters which would by default use 1024 bit key (which is within the realm of possibility of being cracked).
  ```
  curl https://ssl-config.mozilla.org/ffdhe2048.txt > /path/to/dhparam
  ```

### Content-Security-Policy header
      
The header value we use in the config is very conservative, essentially only allowing the site's origin and Google Analytics to do anything. Let's unpact it piece-by-piece, as you will likely need to modify it yourself:

```
default-src 'none';
```
* Don't allow anything by default. 

```
connect-src 'self' www.google-analytics.com;
```
* Allow requests only to the website's origin / domain and to Google Analytics. [Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/connect-src).

```
manifest-src 'self';
```
* Allow manifest only for the website's origin / domain. [Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/manifest-src).

```
style-src 'self';
```
* Allow styles only for the website's origin / domain. [Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/style-src).

```
script-src 'self' www.googletagmanager.com;
```
* Allow script sources only from the website's origin / domain and from Google Analytics. [Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src).

```
img-src 'self' www.googletagmanager.com;
```
* Allow image sources only from the website's origin / domain and from Google Analytics. [Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/img-src).

```
frame-ancestors 'self';
```
* Allow only the website's domain / origin itself to embed it in iframe. [Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/frame-ancestors).

```
base-uri 'none';
```
* Do not allow setting any URI in the `<base>` element. The `<base>` element allows you to set the URI relative to which sources like `/foo/bar.js` are resolved. Setting it to `none` prevents attackers from changing the URI, which would otherwise allow them to load e.g. any JavaScript they want. [Documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/base-uri).

### X-Content-Type-Options header

Prevents browsers from trying to "guess" the [MIME type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types) of a resource and makes it instead rely on the explicitly specified type. This makes sure that the browser does not turn non-execuable resource into executable resource.

### Strict-Transport-Security

This makes sure that future requests to the domain (or its subdomains) will be done exclusivery over HTTPS and never over HTTP. The `max-age` value determines for how long, since the last request, this should be the case. This means that if you stop serving the website over HTTPS, users will not be able to connect to it (at least not without ignoring a big fat warning first). 
