---
layout: project_page
title: "Streaming textures with OpenGL, Qt and Gstreamer"
date: 2014-10-15 17:43:22
permalink: /projects/opengl-streaming-textures/
description: "A demonstration of streaming video pipelines into OpenGL textures in order to process and combine them."
tags: linux opengl qt
---

I once worked with someone who proposed an idea of combining the IR video feed from their product with the feed from a regular visible light camera, onto a single video output, in some sort of useful way. The general consensus in the team was that there would never be enough processor cycles left in the DSP to manage that, after all the other IR video processing going on.

That sparked this personal project of mine, to demonstrate that such video mixing could be accomplished using another processor within the SoC involved: not the DSP, but the Graphics Processing Unit. By streaming the video from the camera input ports to OpenGL textures, this opens up a whole extra opportunity to mix and process the video in a very efficient way using any combination of GL shaders desired.

<!--more-->

In what kind of useful ways could these video streams be processed/mixed together you may ask? Examples could include but are not limited to:
With one stream:

* Colour keying
* Real time colour/contrast correction
* HDR
* Lens warp correction (Important for mixing multiple video streams)
* Perform portions of image recognition algorithms on the GPU using the standard OpenGLSL framework, allowing use on SoCs with a graphics accelerator that lack general purpose (I.e. non-graphics related) usage of the GPU

With two streams (Perhaps one being an infra-red video source):

* Blend video using per-pixel alpha
  * The alpha mask could be generated dynamically as needed in another GL texture and utilised as needed by a texture shader, giving a lot of flexibility
* Drag a box over an area to see the other stream overlaid in the box
* Automatically highlight areas of relatively higher heat on a regular visible light video feed
  * Tint entire area with a certain colour
  * Draw an outline around the hot object

And so on.

<table id="captionedpicture">
	<tr><td>
		<img src="{{ site.url }}/img/projects/opengl-streaming-textures/tanks-ir.jpg" alt="" />
	</td></tr>
	<tr><td>
		<img src="{{ site.url }}/img/projects/opengl-streaming-textures/woodland-pink-highlight.jpg" alt="" />
	</td></tr>
	<tr><td>Primitive visualisations of some ideas which could be achieved using multiple video streams in OpenGL textures. Top: Are those tanks real or cardboard cutouts? Drag a box in the HMI to see the IR feed for that area of the video feed. Bottom: View a visible light camera feed, but automatically highlight any area over a certain threshold from the IR feed with a bright colour, for example to aid search and rescue activities.</td></tr>
</table>

As well as these ideas, I also wanted to see just how much the video feeds could be combined with UI elements in Qt, which I was also experimenting with at the time, purely for technical demonstration purposes. For example, one effect planned was a dynamic cube map texture on a 3D model, featuring reflections of the videos and Qt widgets in a scene, at the same time. 

Now, at this point I should make clear that at that stage, I certainly wasn't the first person to come up with putting video into an OpenGL texture. Others had produced demos, including [MDK talking about GL colorspace conversions](http://www.mdk.org.pl/2007/11/17/gl-colorspace-conversions), and a guy called MacSlow whose blog died.
Not to mention the desktop compositing window managers, which had appeared in recent years, iOS and later Android, in which the entire UI screen is rendered by the GPU, and each window is a seperate texture in the scene. On mobile devices this gave rise to such effects as flipping video orientation when the device is held in a different position, whilst on PCs, users could rotate the whole of a desktop around on a cube, wobble their windows about, or give everything a nice frosted glass effect which drained laptop batteries across the world.

However, as indicated by my above list of ideas, I am more interest in practical real world uses of the technique, more likely to be utilised in embedded single purpose devices.

##Proof of concept

So I set about sorting out a demonstration of such capabilities, initially running on my Linux desktop. I used Qt to abstract away any OpenGL/GLES differences down the line, as the eventual aim was to run on a lighter weight (ish) system on a chip. The result can be seen in the video below.

<table id="captionedpicture" style="max-width: 594px;">
	<tr><td>
		<iframe width="560" height="315" src="//www.youtube.com/embed/HJAvuDFREco" frameborder="0" allowfullscreen></iframe>
	</td></tr>
	<tr><td>A proof of concept demonstrating video mixing on a PC using Qt, OpenGL and Gstreamer</td></tr>
</table>

The videos are played by a Gstreamer pipeline using Fakesink as the video sink element. Fakesink has a callback into the application everytime a newly decoded video frame arrives, that buffer is placed in a queue, handled in the main Qt event thread, which places the video buffer into an OpenGL texture. The texture is then drawn in the scene using a shader which dynamically translates the YUV data into RGB during the render. That shader can be combined with one of several others to add video effects, utilise another texture as an alpha mask etc, again as shown in the video.

##Doin' it on an SoC

I didn't want to go much further adding more ideas from the list to the demo, before seeing it run on a potentially embedded device. I chose the TI OMAP3, because it has a Gstreamer plugin with various elements for video enc/decoding using the DSP, a PowerVR SGX 530 GPU supported by OpenGLES 2.0, and a Linux SDK provided by TI which has a build of Qt and ties all the software elements together.

##Porting

At that point, I thought "Ok, simply get the TI DVSDK setup, tweak the Gstreamer pipeline created in the code, re-build the demo with the Qt qmake in the SDK, and the job is done". 

As I discovered, when it comes to SDKs for embedded applications processors, and trying to join these disparate software elements together for use in a single application, there can be issues. A quick overview of the issues I had goes something like this:

* At first, I decided to build Arago (TI's overlay on OpenEmbedded at the time, before OE became Yocto).
  * I then switched to TI's DVSDK when none of the video demos in Arago would work, because the versions of CMEM and the other video components built into the image were not compatible

* Using the DVSDK the individual functionality needed was then checked
  * "encode" and "decode" video demos worked
  * Gstreamer sample pipelines worked
  * PowerVR OpenGLES2 demos worked
  * Qt demos worked
  * Qt with OpenGL|ES 2.0, did not work, just showed a green area in the GLWidget
* To get Qt to display OpenGL|ES output from the PowerVR within a GLWidget, I tried
  * Upgrading to version 1.6 of the TI Graphics SDK, no difference
  * Using the Qt build from the Arago image, that didn't work with OpenGL either
  * In the end based on intructions in the TI wiki, a particular commit from the Qt git repos was needed, with a patch from TI, to build a working set of Qt libs

After adding the specific Gstreamer pipeline setup to the demo code, and building it for the ARMv7 Linux target, I then ran into the following issues:

* The Codec Engine kept erroring out while loading the VIDDEC2 codec into the DSP. This turned out to be because I was creating an OMAP3 Gstreamer pipeline in the constructor of a derived video pipeline class, but also creating a generic Gstreamer pipeline in the base class constructor as well. The generic Gstreamer pipeline uses decodebin2 autoplugger element, which on an OMAP3 setup appears to use the TI video decoder element. So I was loading the video decoder into the DSP twice. In hindsight this was quite an obvious error, but I was too busy looking into the platform details and verbose Codec Engine debugging, instead of double checking my software design.
* I got the video pipeline running, but it would stall after 3 frames were received by Gstreamer into the queue for display. At least 6 frames were needed in the in and out queues in order to shift them through the application. After some checking I found that on a regular PC, Gstreamer could pass off ~1000 video frame buffers before running out. On a TI OMAP3 by default the number of frame buffers the video decoder allocates is .... 4.
* I finally got the video pipeline rendering into a texture, the frame rate for the application was 0.5 FPS.

After a lot of effort, a 0.5 frames per second render rate was rather disappointing. However at this point the cause of concern was known to me, which was that the texture data was being loaded from the Gstreamer video frame buffer using the standard OpenGL glTexImage2D call.

##Zero copy

You see, the trouble with using glTexImage2D to load the texture data, is that the whole buffer is then copied to another memory location again. On a PC, this is often reasonable, because a fancy 3D accelerator will have its own graphics memory, with a nice fast data bus to allow loading textures good 'n quick.

On an OMAP3 however, the memory used by the applications processor, the DSP, and the GPU are all the same, it's the DRAM. And the data rate for the DRAM is relatively low compared to the grunt of a modern PC. So the ideal situation would be to have the DSP decode a frame of video in YUV format into a buffer in DRAM. The application running on the ARM then passes a pointer to that buffer over to the GPU, which could access the data without having to re-load it through any of the conventional OpenGL calls. A custom fragment shader running on the GPU would then do the YUV to RGB conversion during the render.

<table id="captionedpicture">
	<tr><td>
		<img src="{{ site.url }}/img/projects/opengl-streaming-textures/buffer-zero-copy-diagram.jpg" alt="" />
	</td></tr>
	<tr><td>Diagram of a zero copy video to OpenGL pipeline. The video processor writes a decoded/captured frame to DRAM. The GPU renders it. Instead of copying the buffer at every step, let it sit in the same DRAM location all the time, and pass a pointer to it between the various processors in the system.</td></tr>
</table>

But how to pass the video buffer as texture data to the GPU without copying it through the OpenGL API as normal? Well thankfully, in the case of the OMAP3, TI have provided a kernel driver for exactly this problem, called bc-cat. bc-cat allows loading in the addresses of the desired texture data buffers via IOCTLs. TI seem to have collaborated with Imagination Technologies on the OpenGL side, who have provided some GL extensions for a special GL_TEXTURE_STREAM_IMG texture type, allowing the buffers loaded via the backdoor to be chosen when rendering.

The same buffers are used by Gstreamer repeatedly, so it's a case of tracking which buffer address has arrived with a new frame of video data this time, and specifying that as the texture ID to use when rendering next.

Unfortunately, with bc-cat, specifying the buffer addresses with the IOCTLs has to be done before setting any of the OpenGL stuff up (Apparently). Which lead to the conundrum of how to get all the possible video frame buffer addresses in advance? In the end, given that TI's codec only allocates 4 buffers, I went with the simple but effective approach of just waiting for all the possible buffers to get sent through from Gstreamer, building a map of addresses to texture IDs. Once no new buffer addresses are arriving, set up the OpenGL extensions. The result is that no video is shown for the first 8 frames or so, but with a 30FPS video, this is barely perceptible.

And the result? Well you can see for yourself in the video below:

<table id="captionedpicture" style="max-width: 594px;">
	<tr><td>
		<iframe width="560" height="315" src="//www.youtube.com/embed/10Mt_orGa1E" frameborder="0" allowfullscreen></iframe>
	</td></tr>
	<tr><td>Demonstration of OpenGLSL based video processing blending on a TI OMAP3 EVM using Qt, OpenGL and Gstreamer</td></tr>
</table>

##Now what

The OMAP3 is pretty old now, but then the whole point is that if you do things right, you can get away with doing awesome stuff on an old system. Still, I have my eye on a few rather more current System on Chips to port this to.

Also, more shaders, and demonstrating more of the original concept ideas. To that end, it might be good to move the code to a library for use in individual demos. We'll see.




