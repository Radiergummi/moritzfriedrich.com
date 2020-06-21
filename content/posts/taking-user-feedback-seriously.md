---
title: "Taking user feedback-seriously"
slug: "user-feedback"
date: 2020-06-20T12:03:35+02:00
showDate: true
draft: true
tags: ["blog","story"]
---

Recently, I got annoyed with the way we handle support in some cases. Users write emails or send messages via instant messaging apps, which end up in some ticket
system, get assigned a priority and will be forwarded as a ticket to a support employee. Maybe they don't have time right away because they are busy or on vacation 
or don't like the customer or simply aren't too motivated that day.  
Later on, they discover there's a bug or a feature/change request in the ticket or they don't know the solution because the problem is too technical. What happens 
next is the ticket being assigned to a developer. Developers, by the nature of their role, don't interact with customers directly very often--so they will 
occasionally look into the ticket system, as their main tool is GitHub anyway, and notice a bunch of tickets.

In the best case, these tickets will be converted into issues or at least referred to by their IDs when creating pull requests. Usually, though, the developers will
fix issues in a newer version, fire up Slack and tell the support employees that their problem _"is fixed"_. The support employee, in turn, will reassign the ticket
to themselves and tell the user about the fix.  
Actual feedback is rarely, if ever, collected or mentioned to someone on the engineering team. In most cases, product management will read feedback and maybe 
incorporate it into product descisions.

-----

This is a workflow I have witnessed in this or a similar form a lot by now. It has several issues: most prominently, engineers being completely disconnected from 
the very customers for whom they build the product. Every problem in the product is merely a new ticket ID. Feedback is reduced to new features on the backlog. 
Syncing things between version control and the ticket system will invariably cause things to be forgotten or miscommunicated.  
I've been thinking about how to change this a lot and came up with an experiment I'm including into some smaller projects now:

**A feedback form that creates an issue on GitHub.**

_"What the hell?"_, I hear you say. _"How can this be secure? Won't devs be flooded by pointless complaints and random hate? Are engineers supposed to do support 
now?"_. We'll get to each of those points. I'd just like to detail the implementation before.

The form itself is designed to reduce friction as much as possible: It consists of a message field (a text area without any length restrictions) and an optional,
five-star rating field you know from online shops. The meta-data, like user and customer name, contact information and current application state are transparently
gathered in the background.  
As the user submits the message, it will be augmented with the application state and `POST`ed to the server. As the backend picks the message up, it again augments
it with information I'm not comfortable settable by the user: The user and customer identification details, the application the feedback was created for
