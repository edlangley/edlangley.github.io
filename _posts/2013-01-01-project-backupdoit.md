---
layout: project_page
title: "Backupdoit"
date: 2014-10-29 19:23:54
permalink: /projects/backupdoit/
description: "A backup utility for doit.im"
tags: qt web
---

Backupdoit is a simple utility to back up your actions from doit.im.

**Update:** Binary release of v1.0 for Windows now available! See below.

[doit.im](http://doit.im/) is a web service which provides a system for implementing the [GTD](http://en.wikipedia.org/wiki/Getting_Things_Done) method of time-management.

GTD requires that all your actions/to-dos be recorded in a trusted system. In order to trust doit.im (Or any other web based software for that matter) there must be a way to backup the data recorded in it to somewhere else. A backup feature is not provided by doit as is, so this is my attempt at just such a tool.

<!--more-->

<table id="captionedpicture">
	<tr><td><img src="{{ site.url }}/img/projects/backupdoit/backupdoit_screenshot_main.jpg" alt="Screenshot of backupdoit"/></td></tr>
</table>

The data from your doit.im account is retrieved and can be backed up to either a list in a text file or raw JSON format.

Backupdoit is written in QtQuick and Qt using C++. It was a convenient excuse for me to learn all about QML.

##Download

Get the source code on github [here](https://github.com/edlangley/backupdoit).

Get the version 1.0 binary release for Windows [here](https://dl.dropbox.com/s/nfmt44ua9gn5o6i/backupdoit-1.0-win32.zip). Just extract all files from the zip to somewhere, then run the EXE.

