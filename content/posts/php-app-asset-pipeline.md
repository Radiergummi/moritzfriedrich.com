---
title: "Flexible Asset Pipeline for traditional PHP web apps"
slug: "php-app-asset-pipeline"
date: 2020-04-27T09:58:20+02:00
showDate: true
draft: true
tags: ["blog","story"]
---
Flexible Asset Pipeline for traditional PHP web apps
====================================================
Recently, I've turned from _"single page app everything!"_ toward a more conservative stance, which might very well just be me
getting older and grumpier. Anyway I grew ever more frustrated with the inevitable development time overhead that came with 
even the tiniest frontend change--the sad and ridiculous climax being half an hour to add an optional query parameter to a 
single API call.

I certainly acknowledge the power of React and Redux and all their wonderful features, enabling a whole new class of web
apps that were unthinkable before. Still, I regard them as incredibly specialized tools useful for a narrow set of use cases! 
Many applications can do great without a sophisticated frontend framework or state management library, despite what the voice
of the internet might make you think.  
Bisecting a common business application, loads of GUI stuff is just CRUD with a nice hat on--forms sent to a backend, to be 
processed there. If you find yourself in such a situation, I urge you to consider stripping out the "single page" from your 
app!

-----

What does this mean, though? When building what I call a "traditional" web application, you receive pre-rendered templates from
the server, display data on individual pages and submit forms with modified data back to the server. Interactions between
the data layer and the GUI can trigger a page refresh or use AJAX in the background, or both. This is very much a story of 
[progressive enhancement](https://developer.mozilla.org/en-US/docs/Glossary/Progressive_Enhancement), using AJAX only where 
available and still having a functional application without. Depending on your audience 
