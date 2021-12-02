---
title: "Slash to Search is bad for Usability"
date: 2021-12-02T09:40:52+01:00  
showDate: true  
draft: true  
tags: ["blog","story"]
---
More and more web apps seem to provide a nifty little shortcut: By pressing the `/` key on the keyboard, the focus
instantly changes to the search field. Memorizing this shortcut allows to be more efficient - that is, if you happen to
use an english keyboard layout: The forward slash is located on the bottom left of the keyboard, making it easy to
reach.  
Users from Germany, Italy, Spain, and Russia, for example, will have to use a key combination (most
commonly `Shift + 7`) to insert a forward slash. Depending on how robust the implementation is, doing so will sometimes
focus the search field, or not have any visible effect.

This might seem like a very minor thing to complain about. However, for all the focus we collectively seem to put on
accessibility, this is a pretty basic flaw: Evidently, nobody put a second thought to users with keyboard layouts other
than the US default one when implementing this. I'd simply shrug it off for any small company, but
even [Google has implemented this](https://www.engadget.com/google-search-slash-keyboard-shortcut-034415301.html) in
their search results.

How could we improve this? I'd suggest moving away from the forward slash itself, to the reason it was chosen: Because
it's easy to reach. Have a tiny switch-case for the user locale, pick the rightmost, lowermost special character key on
their keyboard.  
That way, we all get to be more efficient.
