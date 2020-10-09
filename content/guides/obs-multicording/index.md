---
title: "OBS Multicording Setup for Advanced Post-Production"
date: 2020-10-09T21:00:00-00:00
tags: [twitch, obs]
---

There's a video version of this available in two places. Feel free to watch/read along!

- Twitch: PSYCHE Twitch doesn't allow non-affiliates to upload videos anymore. I don't want to give them any opportunity to make money off of me, so no twitch link :(
- YouTube: [https://youtu.be/O9mi0amdMRc](https://youtu.be/O9mi0amdMRc)

## Introduction

Multicording is a term to describe recording/streaming multiple video files at once. In the context of live broadcasts (ie Twitch, YouTube, Facebook Live, etc), an advanced streamer may want to record multiple elements separately to enable some post-production workflows. A great example would be separately recording the different elements of a live broadcast (ie microphone + camera, gameplay, etc) to separate files so that they can be edited separately after the live broadcast is over and uploaded to another service.

> A quick word of caution: I have a pretty powerful machine (Ryzen 9 3900X, 32GB of RAM, and a RTX 2070 GPU). I've had no issues of latency/performance, but if you're pushing the limits of your current setup, this may not work well for you. Feel free to drop me a line via e-mail/discord/comments at the bottom of the page and let me know your feedback!

## High-level plan

For the more technically-inclined, feel free to roll your own solution :) What I'm recommeding is:

- Using Newtek NDI, run multiple OBS instances and output the camera + microphone on one NDI Output, and the gameplay on a separate NDI Output.
- Use a third instance of OBS (or another streaming solution like XSplit or SLOBS, or even a second computer) to mux the two NDI Outputs and add any remaining on-screen elements (like chat replay, follow/raid/sub alerts, etc). Apply filters to your sources (ie Chroma Keying) in *this* OBS instance, not on the source OBS instances.
- Use a single hotkey to record the sources from OBS to separate video files

## OBS Setup

First, set up OBS to not warn you about running multiple instances. Right-click your OBS Shortcut, go to Properties, and add ` --multi warning` to the end of the launch path. For me, the full launch path is:

```
"C:\Program Files\obs-studio\bin\64bit\obs64.exe" --multi warning
```

> IMPORTANT: Notice the space between the final quotemark after `obs64.exe` and `--multi`, as well as the space between `--multi` and `warning`. These are required!

The next thing on the list is to set up the NDI Plugin. NDI allows you to output what you see in OBS over your network (with extremely low latency) so that it can be captured on another machine/program.

Head to the developer's latest release for the NDI plugin (at time of writing, it's 4.9.0 for Windows): [https://github.com/Palakis/obs-ndi/releases/latest](https://github.com/Palakis/obs-ndi/releases/latest)

Download and install the appropriate version for your platform. On Windows, the easiest way is to run the `-Windows-Installer.exe` file and reboot OBS afterwards.

## OBS Profiles and Scenes

With OBS set to launch multiple instances and the NDI plugin installed, open three instances of OBS. Open your first instance normally, then hold your Shift key and click the OBS icon in your taskbar to open another instance. Do this another time to get three instances running.

Using **Profiles**, we'll separate the hotkeys, audio settings, and output settings per OBS instance for optimal multi-cording. I recommend setting up three profiles (but feel free to adjust for your use case):

1. Record - Gameplay
2. Record - Webcam
3. Livestream

For each of these, click "Profile" in the OBS main toolbar, then click "New". Enter a name (you can use the three above) and click OK.

Repeat this for **Scene Collections**. Scene Collections are a way to keep different sets of scenes and sources grouped so you don't have a massive library to choose from in a single OBS instance.

For each profile, also create a corresponding Scene Collection using Scene Colletion > New.

## Batch file

To make this really easy to launch in the future, create a batch file on your desktop to launch all of your OBS instances. You can download the attached file here: [OBS-Multi.bat](OBS-Multi.bat), or create your own using Notepad (or your favorite text editor).

> NOTE: If you download the file, just save it as "OBS-Multi.bat" on your desktop. If you're creating it yourself, make sure to save it as a `.bat` file! If you use Notepad, and your file saves as a text file (double-clicking it just opens in Notepad), open Windows Explorer and on the View tab, check "File name extensions", and rename the file from `<filename>.txt` to `<filename>.bat`.

To create the file yourself, just create a new blank document, set the working directory (`pushd`) into your OBS installation directory, and add a line for each of the profiles/scene collections you set up earlier. Each line needs to follow this format:

```
start "" "<Full path to obs[64].exe>" --multi --collection [collection] --profile [profile]
```

If you use spaces in the collection/profile names, escape them with quotes. The file you can download from my site looks like this:

```bat
pushd "C:\Program Files\obs-studio\bin\64bit"
start "" "C:\Program Files\obs-studio\bin\64bit\obs64.exe" --multi --collection "Record - Webcam" --profile "Record - Webcam"
start "" "C:\Program Files\obs-studio\bin\64bit\obs64.exe" --multi --collection "Record - Gameplay" --profile "Record - Gameplay"
start "" "C:\Program Files\obs-studio\bin\64bit\obs64.exe" --multi --collection "Livestream" --profile "Livestream"
```

Once you've got this file, you should be able to double-click it to launch all of your OBS instances. MAGIC!

> Note: if you've got a streamdeck or similar product, you could at this point set a key to launch this batch file so that you can open all of your OBS instances with just the push of a key!

## Profile and Scene Setup - Webcam + Microphone

Click into the OBS instance that's set up for recording your webcam. Go to Tools > NDI Output Settings and make "Main Output" is unchecked. Click into "Settings" in the controls pane, and make sure to set a few things:

- Stream tab: disconnect from a streaming service if it's showing connected
- Output tab - Recording (set Output mode to Simple)
  - Recording Path: set to a dedicated folder for your camera. Ideally, this will be on a solid state drive.
  - Recording Quality: High Quality, Medium File Size
  - Recording Format: mkv
  - Encoder: Hardware (NVENC) or Software (x264) depending on your computer. If you have a powerful CPU, use Software. If not (or you find your CPU usage is too high), use Hardware.
- Audio tab: Disable all Desktop Audio sources, and only set your Mic up as your normal microphone.
- Video tab: Set Output resolution to 1280x720 if you're going to use full-screen video in your streams, otherwise just 640x360 will be fine. Set your FPS to match your main stream output (I use 60, but many people stream at 30fps).
- Hotkeys tab: Set a hotkey for Start Recording and Stop Recording. In the video, I use Ctrl + Alt + R for Recording, but you can set this to anything.

Click OK on the settings window, and add your camera as a video source in the only scene in the collection. At this point, your camera profile/scene should have a single scene called "Scene", a video capture device for your camera, and your mic audio. You can press your hotkey for recording or click "Start Recording" and make sure a video file is generated in the recording location you specified earlier.

If it's good, to to Tools > NDI Output and check the "Main Output" checkbox, and set a name other than "OBS". In the video, I set it as "WEBCAM+MIC".

## Profile and Scene Setup - Gameplay

Click into the OBS instance that's set up for recording your gameplay. Go to Tools > NDI Output Settings and make "Main Output" is unchecked. Click into "Settings" in the controls pane, and make sure to set a few things:

- Stream tab: disconnect from a streaming service if it's showing connected
- Output tab - Recording (set Output mode to Simple)
  - Recording Path: set to a dedicated folder for your camera. Ideally, this will be on a solid state drive.
  - Recording Quality: High Quality, Medium File Size
  - Recording Format: mkv
  - Encoder: Hardware (NVENC) or Software (x264) depending on your computer. If you have a powerful CPU, use Software. If not (or you find your CPU usage is too high), use Hardware.
- Audio tab: Enable your normal desktop audio, and disable any mic/aux devices.
- Video tab: Set Output resolution to 1920x1080, 1600x900, 1280x720, or whatever your desired video output setting is. This should match your game's resolution or your stream output. If your stream layout doesn't have "full-screen" video, set this to your game's resoltion. A few examples for this setting:

1. Playing retro games in a 4:3 aspect ratio, but streaming in 16:9. You'll want to set the output to a 4:3 resolution here.
2. Playing something at a higher resolution than your stream (ie playing on 1440p but streaming at 720p). Set the output to 720p.
3. Playing at 1080p and having your game show up on the livestream as a 1600x900 window (due to scene layout). Set the output to 1600x900 in this case.

Basically, you want to set this as high as you can without exceeding the size that's needed for stream.

- Hotkeys tab: Set the same hotkey for Start Recording and Stop Recording as you did for the webcam. In the video, I use Ctrl + Alt + R for Recording, but you can set this to anything.

Click OK on the settings window, and add your gameplay as a video source in the only scene in the collection. At this point, your gameplay profile/scene should have a single scene called "Scene", a video capture device or window capture/game capture for your game, and your desktop audio (no mic). You can press your hotkey for recording or click "Start Recording" and make sure a video file is generated in the recording location you specified earlier.

If it's good, to to Tools > NDI Output and check the "Main Output" checkbox, and set a name other than "OBS". In the video, I set it as "GAMEPLAY".

## Profile and Scene Setup - Livestream

You're in the home stretch!

Click into the OBS instance that's set up for live streaming. Go to Tools > NDI Output Settings and make "Main Output" is unchecked. Click into "Settings" in the controls pane, and set your settings as you normally do for stream. My recommendations:

- Stream tab: Connect this to your streaming account (Twitch, Youtube, etc).
- Output tab: Video bitrate, encoder, and audio bitrate set as per your streaming service's guidelines.
- Audio tab: Desktop audio should your normal desktop audio, and no mic should be enabled.
- Video tab: Canvas will ideally be 1920x1080, and the output resolution will be whatever resolution you normally stream at (1080p, 900p, or 720p are all commonly used settings).
- Hotkeys: __**Don't**__ set a hotkey for recording in this profile, unless you *also* want to capture the final composited output. Since you're recording each source element separately, there's no need to do this.

For the scenes, you can set up your normal "Starting Soon", "Intermission", etc scenes. The only thing you'll want to change from normal are your camera and gameplay sources. Instead of using a video capture device or window capture, use a new source: the NDI Source. You can't use the normal capture devices/methods as the other OBS instances are capturing your game/camera/mic with them.

Add a new source for NDI Stream, and in the top dropdown, pick the camera NDI stream. Repeat the process, adding an NDI Stream source for the gamplay. Position your elements as usual, within your scene/layout as you normally would.

## Putting it all together

At this point, you should have three instances of OBS running:

1. Capturing, recording, and rebroadcasting your webcam + microphone
2. Capturing, recording, and rebroadcasting your gameplay
3. Capturing the two above instances, adding extra on-stream elements, and broadcasting to your live streaming service of choice.

You can run a quick test by pressing your hotkey and checking that multiple files are generated at the locations you specified earlier. If everything works, you're good to go!

Stream as usual, just remember to use the batch file you set up to launch all of the OBS instances, and press your hotkey to record all of your sources separately when you start a stream.

As always, I'm available for more personal help if it's needed. The best ways to get a hold of me are: Discord, E-mail, and comments on this page (in that order). You can find my Discord invite and e-mail address on my [About]({{< ref "/about" >}} "About") page, or drop a comment below.

I hope this helps someone out there. Catch you later.

`-Tom`