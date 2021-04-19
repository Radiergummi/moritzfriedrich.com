---
title: "Forwarding Some URLs in Nginx"
date: 2021-04-19T12:53:41+02:00 showDate: true draft: true tags: ["blog","story", "nginx", "devops"]
---
Today we faced an interesting problem with the deployment of our new website to [Webflow](https://webflow.com/). Currently, we have lots of SEO-relevant
aggregation pages and public profiles we're not immediately ready to move to the new application. These pages are automatically generated from our old
application server, receive serious traffic (40k views per day), and cannot be replicated in Webflow, our new website provider.  
Unfortunately, those pages are all placed on the root domain (`matchory.com/offending-pages`), and by "lots", I meant several hundreds of them. This means we
can't simply add redirects in Webflow, but have to keep pointing the DNS for `matchory.com` to our own web servers. There, we take it the other way around and
redirect the traffic for all non-SEO pages to Webflow. But how ([TL;DR](#full-nginx-config))?

-----

Luckily, we use nginx on our web servers, so we could use the proxy module to forward requests to another upstream server. Normally this is used to pass
requests to an internal application server that should not be exposed to the internet directly, but of course nothing's stopping you from proxying to a public
IP address instead! We took this to our advantage by forwarding requests for our website to the main webflow host at `https://proxy-ssl.webflow.com`, _if they
match certain conditions_.

> **How did we determine the target host?**  
> Webflow provides directives for using a custom domain that include adding a `CNAME` record that points to this host:
> ```dns
> www.matchory.com. IN CNAME proxy-ssl.webflow.com. 3600
> ```
> This means whatever server serves this SSL proxy at webflow probably knows how to terminate SSL for our custom domain. If it can do that, we should be equally
> able to perform HTTP requests with a matching host header against it!

Configuring the upstream
------------------------
We start by defining an upstream we want our requests to proxy to:
```nginx
upstream webflow {
    server proxy-ssl.webflow.com:443;
}
```
We only add a single server here; the upstream is simply responsible for having a single target to route to in the rest of the config file. Note the port at the
end of the host name: The default is port 80, even if we use the `https` protocol later on.

Adding a resolver
-----------------
To make sure nginx is able resolve the hostname in the upstream block, we will need to configure a resolver. If you haven't already, you can put it into
your `http` block in the main config file (probably `/etc/nginx/nginx.conf`), but it may equally well be added to your site config, or server block.
```nginx
resolver 8.8.8.8 8.8.4.4;
```
This uses the [public Google resolvers](https://developers.google.com/speed/public-dns), but you can use whichever public DNS you fancy.

Routing a single location to Webflow
------------------------------------
Now that we have a working upstream, we can forward locations to it:
```nginx
location = /foo {
    proxy_pass https://webflow;
}
```
This would proxy any request to `/foo` to the Webflow server, using the https protocol. At least it should: We miss lots of configuration flags, so the above
would fail miserably. To make it work, we'll have to understand what is actually happening in the background if a request is forwarded!

### Excourse: Request proxying, SNI, and SSL
Imagine executing a simple HTTP request against our website. It will look like this:
```http
GET / HTTP/1.1
Host: www.matchory.com
```
That instruct whoever is receiving it to deliver the index page of the host `www.matchory.com`.

Now suppose we'd set up nginx to proxy `/foo` to another server, let's say, `foo.matchory.net`:
```nginx
location = /foo {
    proxy_pass http://foo.matchory.net;
}
```
In that case, nginx would pick up our request, and perform _another_ HTTP requeston its own - think using curl on the server. This request will look something
like this:
```http
GET /foo HTTP/1.1
Host: foo.matchory.net
X-Real-Ip: 1.2.3.4
X-Forwarded-Proto: http
```
As you can see, the host header is being set to the name of the upstream server. This actually makes sense, because we're performing a plain old HTTP request to
the upstream here!
Then again, most of the time, the application running on the upstream server identifies itself as the web server: Our hypothetical `foo` server probably thinks
its name was
`www.matchory.com`. This is important so it can create fully qualified links to other resources it provides, for example.  
So to convey the host header to the primary domain, we can instruct nginx to set the `Host` header:
```nginx
location = /foo {
    proxy_pass http://foo.matchory.net;
    proxy_set_header Host $host;
}
```
This will ensure the host header received by nginx will be passed on to the proxy, which can now successfully respond to the request. Nice!

_What happens, though, if we throw HTTPS into the mix?_

-----

Nothing extraordinary. Instead of sending a plain HTTP request to the upstream server, nginx will re-encrypt the request received before passing it on. While
HTTPS requests are encrypted with the public key of the server, they _do_ in fact expose the target hostname. Otherwise, we couldn't host multiple domains on
the same server with different certificates, because the web server simply wouldn't know which private key to use for decrypting the request.  
Therefore, web servers support a procedure called _**S**erver **N**ame **I**dentification_, or SNI for short. If enabled, a client will send the hostname it
attempts to connect to during the SSL handshake, in plain text. This allows the server to retrieve the appropriate encryption configuration before starting the
actual handshake (Note that I'm grossly over-simplifying, but the details really aren't that important here). After the secure connection was negotiated
successfully, the client sends _the actual, encrypted request_.

If you've been following closely, you might have noticed that there's a difference between the `Host` header we've used in the previous example, and the SNI
name sent during the SSL handshake: The header isn't even sent during the initial connection! If you simply attempted to swap "http" for "https" in the above
example, your request would fail, because nginx would request to connect to `foo.matchory.net` instead of `www.matchory.com`, before sending the request with
the proper hostname.  
This might work for a backend application you control, but not for a SaaS platform like Webflow, that uses a webserver that dynamically resolves hosts.

Setting the SNI name to proxy to
--------------------------------
Everytime I wonder whether nginx can so something, I learn that _of course_ there's a flag for that--you just need to know which. In this case, it's two of
them:
[`proxy_ssl_server_name`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_server_name) and
[`proxy_ssl_name`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_name). The former enables SNI for the upstream connection, the latter
sets the _name_ to use during SNI. This yields the following location:
```nginx
location = /foo {
    proxy_pass              https://webflow;    # Pass the request to the webflow upstream
    proxy_ssl_server_name   on;                 # Enable SNI
    proxy_ssl_name          $host;              # Set the SNI name to the current host
    proxy_set_header        Host $host;         # Set the host header in the request
}
```
This ensures the request will be directed at the correct server, by properly conveying the target server during the SSL handshake (via SNI) and HTTP request
parsing (via header).

Performance optimizations
-------------------------
The last part missing here is the upstream certificate validation: nginx will attempt to verify the certificate of the upstream server, expecting that we
control it and need to make sure the server actually carries one of our certificates. As we're connecting to an externally managed service, this isn't
necessary, and we can simply turn validation off. Additionally, using
the [`proxy_ssl_session_reuse`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_session_reuse) flag, we can configure nginx to reuse SSL
sessions so it doesn't have to perform a full SSL handshake on every request. Update the location block to its full form:
```nginx
location = /foo {
    proxy_pass              https://webflow;
    proxy_ssl_verify        off;
    proxy_ssl_session_reuse on;
    proxy_ssl_server_name   on;
    proxy_ssl_name          $host;

    proxy_set_header Host               $host;
    proxy_set_header X-Real-Ip          $remote_addr;
    proxy_set_header X-Forwarded-For    $remote_addr;
    proxy_set_header X-Forwarded-Proto  https;
}
````

Routing multiple locations to Webflow
-------------------------------------
If you wanted to proxy multiple locations with different URIs to Webflow, copy-pasting this would become tedious soon. By using another cool nginx feature, we
can keep our configuration file short and concise: [Named locations](https://nginx.org/en/docs/http/ngx_http_core_module.html#location). Named locations are
location blocks which cannot be reached by a request automatically, but referred to in other directives instead. Specifically, you may only use them
in `error_page`, `post_action` or `try_files`, which is exactly what we are going to do. Using `try_files`, we can direct nginx to test multiple resources and
use the first that exist. This is often useful to attempt serving static files before falling back to a web app:
```nginx
location / {
    try_files $uri $uri/ /index.php;
}
```
Instead of a PHP file, we can also use a named location as the fallback file:
```nginx
location @fallback {
    # ...
}

location / {
    try_files $uri $uri/ @fallback;
}
```
As we don't want to serve anything from the proxy, we just need a dummy to try before passing the request to our named location (nginx requires at least two
different arguments to `try_files`). This actually took me a bit, until I stumbled upon [this post on ServerFault](https://serverfault.com/a/965779/217063)
which suggests using `/dev/null` as the first argument. I agree with the author that this is a terrible, terrible but also beautifully simple workaround! This
allows the following construct, _with no performance hit_:
```nginx
location @fallback {
    # ...
}

location / {
    try_files /dev/null @fallback;
}
```
This will send all requests to the fallback location immediately. Applying this to our situation, we end up with the following (shortened for clarity):
```nginx
upstream webflow {
    server proxy-ssl.webflow.com:443;
}

server {
    location @webflow {
        proxy_pass https://webflow;
        # ...
    }

    location = / {
        try_files /dev/null @webflow;
    }

    location ^~ /([a-z]{2})/.*$ {
        try_files /dev/null @webflow;
    }
}
```
With just a single line, we can proxy a select URI pattern to our Webflow site, without any changes to the existing site, the SSL certificates, or Webflow
settings. Any performance hit by the doubled requests is levelled out by Cloudflare caching in front of nginx.

As is often the case, I would have taken _considerably_ longer if it wasn't for the excellent people posting on StackOverflow and ServerFault. Thank you all!

Full nginx config
-----------------
The final nginx configuration looks like the following:
```nginx
# Start by defining an upstream to send requests to. This helps organization.
upstream webflow {

    # Make sure to add the port after the server name, because nginx won't do this
    # automatically, even if proxying to a https target.
    server proxy-ssl.webflow.com:443;
}

# Make sure nginx can resolve the webflow domain correctly
resolver 8.8.8.8 8.8.4.4;

server {

    # The server name of our primary domain. Note that this configuration will be used for
    # both the old _and_ the new website.
    server_name www.domain.com;

    # This named location is only routed internally, and will proxy any requests to the 
    # webflow upstream with the proper configuration set.
    location @webflow {

        # Note the "https" here, making sure the request is re-encrypted
        proxy_pass                          https://webflow;

        # Set the host header of our current request (probably www.domain.com). We want to
        # forward that to the upstream, so they can properly route the request.
        proxy_set_header Host               $host;

        # Setting these is good faith
        proxy_set_header X-Forwarded-For    $remote_addr;
        proxy_set_header X_FORWARDED_PROTO  https;

        # Disable SSL verification: As we don't own the Webflow upstream, we can't keep 
        # track of their certificate. We trust their upstream anyway, so if they went rogue,
        # we couldn't do anything about it otherwise, either.
        proxy_ssl_verify        off;

        # Reuse SSL sessions: This prevents having to perform a separate SSL handshake for
        # every single request, but pool connections instead.
        proxy_ssl_session_reuse on;

        # SSL server name: This is the crucial bit. During SNI (server name identification),
        # nginx uses the name of the upstream by default, which would be Webflow's
        # proxy-ssl.webflow.com in our case. Here, we enable manual ambiguation.
        proxy_ssl_server_name   on;

        # Set the actual SNI name to send: Here, we use our Host header value instead, so 
        # the request can be identified on the Webflow origin server.
        proxy_ssl_name          $host;
    }

    # This block matches any URLs that start with a language code, and forwards them to the
    # webflow named location.
    location ~^ /([a-z]{2})/.*$ {
        try_files /dev/null @webflow;
    }

    # This block matches all requests for the homepage (eg. a single "/"), and forwards them
    # to the webflow named location.
    location = / {
        try_files /dev/null @webflow;
    }

    # This is the default location for all other requests, routed to our legacy application.
    location / {
        try_files $uri $uri/ @app;
    }

    # ...
}
```
