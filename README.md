# Source of moritzfriedrich.com
This is the source code of my personal homepage.

## Tech Stack
The website is currently running as follows:

 - DNS managed by Cloudflare, `CNAME` points to `radiergummi.github.io`.
 - [Hugo](https://gohugo.io/) builds the source in this repository triggered by a GitHub Action as soon as I push to master.
 - The content of the `build` is actually a git submodule, pointing to my
   [`radiergummi.github.io`](https://github.com/Radiergummi/radiergummi.github.io) repository. After each build, the new version will be 
   committed to that repository automatically.
 - The Cloudflare cache is purged so the changes are visible immediately.

All in all, this is completely automated, absolutely free (I only pay the domain registration) and _blazingly_ fast. As soon as I edit or add
a markdown file in the content directory, the magic gets to work.

## Directory structure
The website currenty has a blog section and some static pages. All content lives in the [`content/`](./content) directory.

## Build and setup
To build the site, you'll need hugo extended and two node modules:
```bash
yarn install
hugo
```
