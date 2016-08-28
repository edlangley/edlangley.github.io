---
layout: project_page
title: "Java Virtual Puzzle and some other old Java stuff"
date: 2015-03-04 18:28:36
permalink: /projects/java-virtualpuzzle-and-other-stuff/
description: "A collection of Java apps and applets I wrote many years ago that still work happily"
tags: java
---

Whilst preparing to do some Android development some time soon, I thought I'd brush up on the old Java skills. Very old skills in fact, as one way I decided to do this was to unearth my old school project, written in Java, from about 14 years ago.

<!--more-->

It's a virtual sliding puzzle game, imaginitively titled "Virtual Puzzle".

<table id="captionedpicture">
	<tr><td>
		<img src="{{ site.url }}/img/projects/forgotten-java/the-park-jumbled.jpg" alt="Virtual puzzle screenshot showing a photograph of a park in 3x4 tiled pieces jumbled up on a yellow background in an application window. One of the pieces is a blank blue space." />
	</td></tr>
	<tr><td>
		<img src="{{ site.url }}/img/projects/forgotten-java/the-park-completed.jpg" alt="The same park image puzzle arranged correctly with a dialog congratulating the user, 376 moves in 223 seconds." />
	</td></tr>
</table>

I thought the application was finished all that time ago, turns out I must have been in the middle of some pretty hefty alterations when the project was left to sit. When the code was unearthed on a backup drive recently, it didn't compile at all, due to all sorts of mid-fiddling type syntax errors. Unfortunately version control usage was not de facto at secondary school in 2001 :-(

Anyway I have stuck it in a Github repo and finished it off. The puzzle "engine" class was all working, just the dialogues for the rest of the application that needed implementing. Like a fine antique, I've tried to keep it in original condition, mostly AWT based, as per Java 1.1/1.2 at the time, I forget. A bit of Swing has slipped in mind you, mainly the JTable to show the best times for the current user alongside the puzzle names, when choosing a puzzle to play.

<table id="captionedpicture" style="max-width: 394px;">
	<tr><td>
		<img src="{{ site.url }}/img/projects/forgotten-java/choose-puzzle-dialog.jpg" alt="Dialog containing a 3 column table with 1 row per puzzle, columns are puzzle name, best time and lowest move count for the current user on that puzzle." />
	</td></tr>
	<tr><td>The look and feel of the Swing based Jtable doesn't quite match the rest of the AWT components, but never mind.</td></tr>
</table>

If you fancy a quick go at it:

<div class="preformatted_console"><pre>
$ git clone https://github.com/edlangley/virtual-puzzle.git
$ cd virtual-puzzle/
$ javac src/*.java
$ java -cp src/ VirtualPuzzleApp
</pre></div>

First thing to do once the app has loaded is add a user, then select "Manage Puzzles". Add a puzzle, you'll need an image file in any of the common formats (JPG, PNG etc). Choose the number of pieces depending how hard you want to make it. Then go back and select "Do a Puzzle", choose it, and off you go.

Once you've done a 3x4 like the one shown above, you can try your hand at something like this:

<table id="captionedpicture">
	<tr><td>
		<img src="{{ site.url }}/img/projects/forgotten-java/aquarium-jumbled.jpg" alt="A puzzle with 26x20 pieces." />
	</td></tr>
</table>

Or this:

<table id="captionedpicture">
	<tr><td>
		<img src="{{ site.url }}/img/projects/forgotten-java/buildings-jumbled.jpg" alt="A puzzle with 100x80 pieces." />
	</td></tr>
</table>

One possible tweak to the gameplay (As much as a sliding puzzle game can be considered to have gameplay), may be to have a set time limit for each puzzle, with the timer counting down instead of up. The time text could turn red as it approaches 0, just to add a bit more tension to the proceedings perhaps.

## Other old stuff

Here's an applet featuring a first pass at porting a simple 2D tile scrolling engine from C++ to Java.

<table id="captionedpicture" style="max-width: 674px;">
	<tr><td>
		<applet code="PlatformGame.class" width="640" height="480" codebase="../../img/projects/forgotten-java/platformgame/"></applet>
	</td></tr>
	<tr><td>Click on the applet to give it focus, then use the arrow keys to scroll the map.</td></tr>
</table>



Back in 2002 this ran like a slug on an average desktop machine (about 20 seconds to scroll from one side of the map to the other), on todays machines it is pretty nippy, although the fan in my laptop does seem to ramp up to full speed. Still, who needs optimisation when you can just wait 13 years instead (I hope I don't need to tell you that I'm joking of course).


## Getting up to date

One thing I did do during this re-cap was abandon my old library of out-dated Java reading material, and just go with one good up to date book instead. On that basis I would recommend [Java in a Nutshell, 6th Edition](http://shop.oreilly.com/product/0636920030775.do) By Benjamin J Evans & David Flanagan. Feel free to make your own recommendation (Only one!) in the comments.

