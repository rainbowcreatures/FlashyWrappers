# FW SDK

5/6/2017

The repository has been moved to a private repo at bitbucket.org. You can still get access to sources in case you are a former customer.
This repo will stay open only for issues of the free version. If you'd like to post a high priority issue, or a feature request, please post it to BitBucket.

Note: FW is resuming being both free and commercial, which also means being supported / updated again starting today.

Building
--------

*Windows / Android / Flash*
The .bat files are used to build FW on Windows.

*Windows*

Windows came in 2 flavors. Like for all encoders, FFmpeg based encoder came first. This is the only platform where I still kept it. The reason is, when debugging FFmpeg based encoder for Flash, 
it was often useful to make a build for Windows with identical code - debugging FlasCC / Crossbridge can be pretty horrible. The FFmpeg flavor of FlashyWrappers Windows was built with either MingW or MSVC(lately only MSVC) - the define USE_MEDIACODEC determines if the encoder is FFmpeg or MediaCodec based.

*OS X / iOS*

Those platforms use identical source code file, luckily AVFoundation is almost identical on OS X and iOS. There is an unfinished OS X path which would use GLES capture like iOS does, so for now OS X only accepts bitmap frames passed from AIR.

*iOS: experimental ReplayKit support*

Before this makes its way into PDF, here's a short intro about how to use this new feature. 

ReplayKit for iOS allows you to capture all audio and video from your AIR application. There is no need for FWSoundMixer or other tricks to capture all playing audio. You can also capture microphone together with the audio being recorded.

The disadvantage is, this API is pretty sandboxed by Apple, so for example it doesn't allow direct access to your video file and displays some standard pop ups, such as an alert asking the user if they want to allow video recording, or displaying a standard preview popup after the video is recorded. This popup allows users to edit the video and save it to camera roll, or share it. 

To use ReplayKit, start by calling myEncoder.load as usual, then after 'ready' event, do NOT use the usual myEncoder.start, and do NOT use myEncoder.capture on every frame. 

Instead, to start recording with ReplayKit, call:

```javascript
myEncoder.iOS_ReplayKitStart(0);   (0 without microphone, 1 with microphone)
```

This will popup a permission asking to record your screen. Then, wait for the 'replaykit_started' status event to know you can present whatever you need to record.

```javascript
			if (event.code == 'replaykit_started') {
				// encoder started, start capturing frames as soon as possible if (e.code == “started”) {
				trace("ReplayKit recording started...");
```

If you want to stop the recording, call:

myEncoder.iOS_ReplayKitStop();

You could again wait for a 'replaykit_stopped' event, but in the current implementation this is a bit useless, because a video preview automatically pops up after the recording is stopped, so you'll get 'replaykit_preview_started' event shortly after the 'replaykit_stopped' event. 

After the user is finished with editing and saving (or sharing) the video, you'll recieve 'replaykit_preview_finished' event.

```javascript
			if (event.code == 'replaykit_stopped') {
				trace('ReplayKit recording stopped');
			}
			if (event.code == 'replaykit_preview_started') {
				trace('ReplayKit preview started');
			}
			if (event.code == 'replaykit_preview_finished') {
				trace('ReplayKit preview finished');
			
			}
```

Keep in mind that ReplayKit API by Apple is its own separate beast and has nothing to do with the rest of low level FlashyWrappers methods it is using to capture from GLES context and record videos. FW merely allows you to access this new Apple API via the new ANE methods.

*Hello world*

This source code was taken from the Hello World example.

```javascript
package  {
	
	import flash.display.MovieClip;
	import com.rainbowcreatures.*;
	import com.rainbowcreatures.swf.*;
	import flash.events.StatusEvent; // in case you're not importing this already
	import flash.events.Event; // in case you're not importing this already
	import flash.events.IOErrorEvent;
	import flash.utils.ByteArray;
	import flash.net.FileReference;
	import flash.text.TextField;
	import flash.text.TextFormat;

	[SWF(width=600,height=400, frameRate='24')]		
	public class Helloworld extends MovieClip {
		
		var myEncoder:FWVideoEncoder;
		var frameIndex:Number = 0;
		var maxFrames:Number = 50;
		var txt:TextField = new TextField();
		
		public function Helloworld() {
			// constructor code
			var tf:TextFormat = new TextFormat("Arial", 30, 0XAA5050);
			txt.text = "Hello world!";			
			txt.setTextFormat(tf);
			txt.width = 200;
			txt.y = 180;
			addChild(txt);			
			myEncoder = FWVideoEncoder.getInstance(this);			
			myEncoder.addEventListener(StatusEvent.STATUS, onStatus);
			myEncoder.load("../../lib/FlashPlayer/mp4/");			
		}
		
		private function onStatus(e:StatusEvent):void {
			trace("Got status");
			if (e.code == "ready") {
				myEncoder.setDimensions(stage.stageWidth, stage.stageHeight);
				myEncoder.start(24);
				trace("FlashyWrappers ready! Init...");
				addEventListener(Event.ENTER_FRAME, onFrame);
			}
			if (e.code == "encoded") {
				var bo:ByteArray = myEncoder.getVideo();
				trace("Recording finished, video length: " + bo.length);								
				
				// save the file for check
				var saveFile:FileReference = new FileReference();
				saveFile.addEventListener(Event.COMPLETE, saveCompleteHandler);
				saveFile.addEventListener(IOErrorEvent.IO_ERROR, saveIOErrorHandler);
				saveFile.save(bo, "video.mp4");								
			}
		}
		
		private function onFrame(e:Event):void {
			// animate the rectangle
			txt.x++;

			// clear the background, otherwise all transparency will be black by default in the video
			graphics.beginFill(0XFFFFFF, 1);
			graphics.drawRect(0, 0, stage.stageWidth, stage.stageHeight);
			graphics.endFill();
			
			// render one frame of your stuff into someMovieClip, then capture and
			// add one frame into the FlashyWrappers video encoder like this:
			myEncoder.capture(this);
			frameIndex++;
			if (frameIndex >= maxFrames - 1) {
				removeEventListener(Event.ENTER_FRAME, onFrame);				
				myEncoder.finish();
			}
		}							

		// file was saved
		private function saveCompleteHandler(e:Event):void {
			trace("Video saved!");
		}
		
		// some error happened
		private function saveIOErrorHandler(e:IOErrorEvent):void {
			trace("Video NOT saved:(");
		}

	}	
}

```


