---
title: "Automatically deployed feature branch previews"
slug: "feature-branch-previews"
date: 2020-01-13T20:54:44+01:00
showDate: true
draft: true
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

The solution I came up with fulfills these criteria with a number of clever/imbecile (your pick) tricks!

### Choosing a domain schema
