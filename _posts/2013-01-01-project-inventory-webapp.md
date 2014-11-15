---
layout: project_page
title: "Inventory PHP app"
date: 2013-10-01 20:36:58
permalink: /projects/inventory-webapp/
description: "A simple general purpose inventory system web application written in PHP"
tags: web
---

<table id="captionedpicture" style="max-width: 660px;">
	<tr><td><img src="{{ site.url }}/img/projects/inventory-webapp/electronics-inventory-screenshot.jpg" alt="Screenshot of the inventory in a browser"/></td></tr>
</table>

At university, on the Computing for Real Time Systems course at UWE, the "Web Technologies" module stood out like a sore thumb amongst all the low level embedded software development and OS theory. During the webby activities, one of the assignments was to produce an administration back end for an imaginary classified adverts website in PHP.

Some years later I decided I needed something to keep a record of what electronics components I had stored around, so I dug out this code and re-purposed it as a general purpose inventory. You may find it useful for a similar purpose.

<!--more-->

I have my own instance of it running online, not just on a web host somewhere mind you, but in the cloud! I'm using Red Hat's rather excellent [OpenShift](https://www.openshift.com/). Not that I need any of the scaling or elasticity that such an infrastructure provides, but just to try it out. Plus a good cloud is almost obligatory for any self-respecting web app in recent years.


Find the code with instructions in the readme on [Github](https://github.com/edlangley/inventory-webapp).


Currently the interface has a few usability problems that I found irritating recently whilst doing a lot of updates to my inventory which have been queued up for some years. After adding/removing an item or changing quantities the entire inventory contents gets re-loaded which is heavy going. Also there's no way to edit the description of an item.

It would be good to add some Javascript with a good RESTful API and some JSON at some point, but there are too many other projects for me to concentrate right now. Alternatively, fork the github project see what you can do and send me a pull request!
