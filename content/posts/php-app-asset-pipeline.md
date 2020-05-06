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

What does this mean, though? When building what I call a "traditional" web application, you receive pre-rendered templates from
the server, display data on individual pages and submit forms with modified data back to the server. Interactions between
the data layer and the GUI can trigger a page refresh or use AJAX in the background, or both.  
This approach allows developers to move incredibly quickly, getting a working application up and running in a matter of days.

That is because lots of layers of complexity are left out: A fully-functional REST or GraphQL API with an authorization layer requires a more formal workflow and definitely a sober approach to security. The client app will require a substantial mount of time to build after each modification. On top of that, modern frontend development relies on TypeScript, which is great--except for rapid prototyping.

Creating a simple asset pipeline
--------------------------------
Still, it's 2020, and you probably don't want to omit frontend assets alltogether: We definitely need a way to bring CSS, JS and images into our site in an optimized manner. So it's time to throw webpack into the mix!  
At the most basic, our configuration might look like so:
```js
const { resolve } = require('path');

module.exports = {
  entry: './resources/js/index.js',
  output: {
    filename: 'main.js',
    path: resolve(__dirname, 'public'),
  },
  rules: [
    {
      test: /\.js$/,
      loader: 'babel-loader',
      exclude: /node_modules/
    },
    {
      test: /\.css$/,
      use: ['style-loader', 'css-loader']
    }
  ]
};
```
What's it do? Take `./resources/js/index.js` and generate `public/main.js` from it, processing any imported CSS in the process. 
This allows for a basic asset pipeline.


This is very much a story of 
[progressive enhancement](https://developer.mozilla.org/en-US/docs/Glossary/Progressive_Enhancement), using AJAX only where 
available and still having a functional application without. Depending on your audience, this might be relevant.  
