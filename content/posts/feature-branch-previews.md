---
title: "Automatically deployed feature branch previews"
slug: "feature-branch-previews"
date: 2020-03-14T13:12:20+01:00
showDate: true
draft: false
tags: ["blog","story"]
---
Automatically deployed feature branch previews
==============================================
At [MessengerPeople](https://www.messengerpeople.dev), we have employed a pretty strict branching schema:

```
                      /───fix/foo──o──\
                     /                 \
                    /                   \
 master ━━┯━━━x━━━━x━━━━━━━━━━━━━━━━━━━━━x━━┯━━━━━━━━━━━━x━━┯━━━━━━━━━
          │    \                            │           /   │
          │     \                           │          /    │
          │      \───feature/bar─o─o─o────o─│───o──o──/     │
          │                                 │               │
       v1.24.8                           v1.24.9         v1.25.0

x: Merge/Branch
o: Commit
```

There's a _single_ branch and source of truth, which is `master`. Every Release is a tag on master, every new development 
takes place in a new branch. This makes version control incredibly easy to reason with, as there simply isn't too much 
complexity: Master is bleeding-edge, releases are fixed points in time and code.  
This has the additional advantage of master being a kind-of staging environment: Our CI pipeline builds after
any commit to master, which usually happens as soon as a branch is merged into it. The result is deployed to our staging 
server, ready for QA to review before creating the actual release.

What has caused a little headache, though, is how to properly test feature branches. Easy for developers! They simply run 
`docker-compose up` and the development stack is spun up.  
But what about testers, product managers, eager sales colleagues? They have no chance to review a feature in development 
until the dev deems it ready to merge (which it most definitely is not).

--------

This problem was bugging me more than I'd like to admit. After lots of coffee though, I came up with a plan! Our ideal 
solution should fulfill the following requirements:

 - **Feature branches should be automatically deployed whenever someone commits to them.** This is ideal for rapid 
   interaction testing (a fancy word I just made up for someone screaming "WTF nothing works!" across the hall and 
   someoneone shouting "I just fixed it, reload the page!" back at them).
 - **Deployments should have a unique hostname.** As we're working with web apps used to living on the domain root, we
   can't just put them in a subfolder without breaking things. Therefore, each branch deployment must be available in their
   own subdomain (which brings additional pitfalls, more on that below).
 - **Deployments must have a valid SSL certificate and ask for credentials.** Feature branches could contain sensitive 
   details, personal data or inappropriate jokes (ever sent _"penis"_ to 120.000 people by accident? Yup, witnessed it), so
   we must ensure only employees are able to view them.
 - **After merging branches, the server should remove their remnants.** In a fast-paced environment, we create lots of 
   branches, most of them pretty ephemeral. To avoid cluttering the server with hundreds of obsolete versions, it should
   remove stale deployments after they have been merged into master.

The solution we came up with fulfills these criteria with a number of clever/imbecile (your pick) tricks!

### Deployment strategy
As it often goes, your initial assumptions about a problem turn out to have a small mistake somewhere, causing everything 
relying on them to crumble down. With this project, deployment was the part that I had to rethink the most. 

#### Iteration 1: Let the CI deploy to Firebase
As we run the production and staging environments on [Firebase Hosting](https://firebase.google.com/docs/hosting), I though
it'd make the most sense to deploy branch previews to new Firebase sites, too. You can actually host multiple sites in the 
sample Google Cloud project, but there's an upper limit of 36 websites per project. As we currently have 6 production sites up,
this leaves us with 30 available previews&mdash;not enough. I shortly looked into creating new Google Cloud projects
programmatically before deciding to abandon this approach.

#### Iteration 2: Listen to GitHub webhooks on a preview server
A dedicated server it was, then. I experimented with a simple "controller" application written in Symfony that would listen to
GitHub webhooks, create a preview directory, check out or pull the code change and build it according to some simple rules,
maybe even a configuration file in the repository.  
As I figured production builds would require more time than GitHub was willing to wait for the webhook response, I addded a job
queue to the application and set up a separate worker process to process queued build jobs. The worker would scan the project
structure, look for build instructions and carry it out. I even thought about dynamically running docker-compose setups to
support backend applications, too.

It took me longer than I'd like to admit that I've started to recreate our CI server.

![Genius.](https://user-images.githubusercontent.com/6115429/76680124-5fa8c180-65e6-11ea-9ef0-c399117b9a9c.jpeg)

While the application would indeed build as expected, it took ages and as several builds piled up, the server ran out of file
descriptors. After the sudden realisation that we have complex build pipelines for a reason and my whacky little 
`if (fileExists('package.json'))` would _not_ be enough, I scrapped the idea of the controller.

#### Iteration 3: 
I turned back to our [Buddy CI server](https://buddy.works/), which I had foolishly ignored in the previous iteration. Buddy is
an _amazing_ product: What previously took me hours in Jenkins (don't even get me started on the BlueOcean pipeline syntax) is
done in a matter of minutes on Buddy.  
A separate pipeline was set up, configured to be limited to branches matching `^refs\/head\/(feature|fix)-(.+)$`. I decided to
have the preview server do as little CGI work as possible to reduce complexity. This demanded deploying branch build artefacts
to the correct location, within the web server root directory, up front. Therefore, I added a shell script to Buddy to build a
slug of the project and branch name:
```bash
export REPOSITORY=$(echo $BUDDY_REPO_SLUG | cut -d'/' -f2)
export BRANCH=$(echo "$BUDDY_EXECUTION_BRANCH" | iconv -t ascii//TRANSLIT | sed -r s/[^a-zA-Z0-9]+/-/g | sed -r s/^-+\|-+$//g | tr A-Z a-z)

export PREVIEW_DEPLOYMENT_TARGET=$REPOSITORY-$BRANCH
```
Thankfully, Buddy already provides a slug version of the repository name (think `acme-inc/my-repo`). We split it on the slash
and take the second part of the result to get the repository name slug. Then, we "slugify" the current branch and append that
to the repository name.  
This leaves us with identifiers like `my-repo-feature-add-foo`, which are ideal for URLs! After the build process is done, we deploy the content of our `dist/` directory to the remote path `/var/www/previews/$PREVIEW_DEPLOYMENT_TARGET`, as Buddy supports using environment variables in deployment actions.

To sum up: Instead of wasting time needlessly managing Google Cloud API complexity or badly recreating CI servers, we take what
we have and deploy the existing artefacts to a static file server. Deployments: Check!

### Choosing an URL schema
To serve those files, we have nginx set up with the web root pointing to `/var/www/previews`. There's two things left to take
care of, though: Serving previews using HTTPS and limit the audience to our own employees. HTTPS is essential for CORS and
authorization&mdash;also, it's 2020. We have free and ubiquituous SSL thanks to Let's Encrypt. There's no excuse, not even for
internal projects.  
Additionally, we don't want to expose new features to the public _in an uncontrolled manner_. This can lead to all kinds of
trouble, from exposing sensitive details to raising expectations for new stuff that is ultimately scrapped.

Now, there are several options when it comes to the schema for URLs to your previews. For example, you could place all previews 
in sub-directories of your preview domain: That would look like `https://previews.example.org/{preview-slug}/`. This requires 
your application to be tolerant to path changes, though.  
Instead, I settled on second-level subdomains: `https://{preview-slug}.previews.example.org/`. While this might sound more
complex at first, it prevents the path issue and allows us to choose different webserver or CDN rules for individual previews,
if need be.

The following nginx configuration handles this setup:
```nginx
server {
    server_name ~^(?<subdomain>[^.]+).preview.messengerpeople.dev;
    root /var/www/previews/$subdomain;
    index index.html;

    access_log /var/log/nginx/$host.access.log preview;
    error_log  /var/log/nginx/preview.error.log;

    location / {
        try_files /__PREVIEW_BUILD__/index.html /dist/$uri /dist/index.html /build/$uri /build/index.html /public/$uri /public/index.html /$uri /index.html =404;
    }

    error_page 404 /404.html;

    location = /404.html {
        root /var/www/previews;
        internal;
    }

    listen              [::]:443 ssl http2;                                               # managed by Certbot
    listen                   443 ssl http2;                                               # managed by Certbot
    ssl_certificate     /etc/letsencrypt/live/preview.messengerpeople.dev/fullchain.pem;  # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/preview.messengerpeople.dev/privkey.pem;    # managed by Certbot
    include             /etc/letsencrypt/options-ssl-nginx.conf;                          # managed by Certbot
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;                                # managed by Certbot
}

```

This routes all requests for a subdomain into a directory named accordingly below `/var/www/previews`. If it doesn't exist, the
friendly 404 page at `/var/www/previews/404.html` will be served instead. If it does, we have a matching preview! To support 
multiple environments, the `try_files` directive simply probes multiple common directory schemas: `dist/`, `build/` or 
`public/` (more on `__PREVIEW_BUILD__` later).  

### SSL shenanigans
I generated the SSL certificate referenced in the nginx configuration using the following command:   
```bash
sudo letsencrypt certonly \
    -d \*.preview.messengerpeople.dev \
    -d preview.messengerpeople.dev \
    --dns-cloudflare \
    --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini
```

Wildcard domains as `*.preview.messengerpeople.dev` require a DNS verification. Certbot supports several plugins to perform DNS
verifications, one of them for Cloudflare. This will automatically create and delete a verification record for the domain, 
which is perfect: Setup-and-forget SSL for our preview domain.  
Now, in theory, the previews should be available on the internet already: That is, unless the domain is proxied through the
Cloudflare cache! While you can _create_ wildcard records in the Cloudflare DNS and even proxy them (if you're at least on a
Business plan), the domain certificate still won't cover it; this makes using Cloudflare for the previews mostly impossible. 

However, there's an escape hatch here! Cloudflare supports uploading custom certificates for websites on a Business plan.
We have an existing certificate from Let's Encrypt already, but it has a pretty low expiration time (2&ndash;3 months), so
uploading the renewed certificate every other day would be both a hassle and too error-prone.  

**Lucky for us, the Cloudflare API allows to upload custom certificates programmatically and Let's Encrypt has post-renewal
hooks!**

I've written [a post-renewal hook script](https://gist.github.com/Radiergummi/bd7a7298a6d07e76912daa06a72707c5) that will take
the existing credentials from the Cloudflare DNS verification plugin and update the custom certificate with the freshly created
file right after the unattended renewal on our preview server.  
All in all, this allows us to use a second-level subdomain, with a valid and auto-renewed SSL certificate, via the Cloudflare 
Proxy.

### Guarding access
We could simply set up a Basic Authorization here, share a set of credentials with the company and leave the prompt to the 
user browser. As we've gone through all the trouble to make Cloudflare work, why not go the extra mile and defer authentication
to [Cloudflare Access](https://teams.cloudflare.com/access/)? Access is an easy way to let users authenticate with their 
company accounts, be it GitHub for developers or Microsoft for everyone else.  
Set-up is pretty straight-forward once you've understood the security model. After creating an access policy, you'll need to
restrict access to nginx to Cloudflare IP addresses, which are available from `https://www.cloudflare.com/ips-v4` and
`https://www.cloudflare.com/ips-v6`. We have a script in place that polls the list every other day and updates an nginx snippet
included in the server configuration.
Bam! Previews are now restricted to employees.

### Finishing touches
Almost there! As we're working on a tool intended to be used by non-engineers too, it should have more UX than usual. Not that
developers do not deserve great UX&mdash;they do!&mdash;but a generic 404 page from nginx looks not that scary to us.  
To make this whole thing neat and approachable, I decided to render a placeholder page as soon as the build starts and remove
it after the build completes. As Buddy also supports actions on pipeline failures, we can even override it with an error page
in case the build fails!

This merely involves another build step that writes a HEREDOC string to `__PREVIEW_BUILD__/index.html`: 
```bash
mkdir -p /var/www/previews/$PREVIEW_DEPLOYMENT_TARGET/__PREVIEW_BUILD__
cat << EOF > /var/www/previews/$PREVIEW_DEPLOYMENT_TARGET/__PREVIEW_BUILD__/index.html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Preview of $BUDDY_EXECUTION_BRANCH</title>
    <link href="https://fonts.googleapis.com/css2?family=Open+Sans:wght@400&display=swap" rel="stylesheet">
	  <link href="https://preview.messengerpeople.dev/assets/style.css">
</head>
<body>
<img src="https://preview.messengerpeople.dev/assets/img/build.svg">
<h1>Preview Build is in progress</h1>
<p>
    Hey there! We're currently building a preview of the branch
    <code>$BUDDY_EXECUTION_BRANCH</code>.
</p>
<p>Please wait for a moment until the page is ready. It will refresh automatically.</p>
<button type="button" id="refresh">Refresh now</button>
<script async>
  setTimeout(() => window.location.reload(), 30000);
  document.addEventListener('DOMContentLoaded', () => document
    .getElementById('refresh')
    .addEventListener('click', () => window.location.reload()),
  );
</script>
</body>
</html>
EOF
```

This renders the following, auto-reloading page:
![Screenshot](https://user-images.githubusercontent.com/6115429/76681509-e4e6a300-65f3-11ea-9f8a-a8aaa0c988ee.png)
_By the way, the excellent illustration has been taken from [undraw.co](https://undraw.co/)!_

This allows engineers to send a link to product managers, testers or 
[random employees](https://www.joelonsoftware.com/2000/08/09/the-joel-test-12-steps-to-better-code/) throughout the company,
all of which can simply leave the tab open until the actual application runs. 

### Future plans
There are so many ways to expand on this system! We have just included an index page, displaying all available previews. In the
future, we'd like to add a feedback overlay, allowing testers to directly add notes and screenshots as issues to the 
repository.  
I would also love to include support for other types of applications, for example whole docker-compose stacks.

That's a story for another day, though :)
