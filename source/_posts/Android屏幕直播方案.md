---
title: Android屏幕直播方案
date: 2016-06-17
desc: Android，视频，直播，WebSocket，FFmpeg，H.264
---
项目需求是实时同步Android手机屏幕画面至浏览器。这里有两个挑战，一是Android如何在应用内获得屏幕实时视频流，另一个是如何在浏览器上做视频直播。经过一番折腾，确定了如下的实现方案。期间，我们也实现了手机摄像头的直播。

<!-- more -->

演示效果：

![演示](/img/android-live-demo.gif)

# Android获取实时屏幕画面
## 原理与基础设置
Android 5.0版本之后，支持使用`MediaProjection`的方式获取屏幕视频流。具体的使用方法和原理如下图所示：

![MediaProjection原理](/img/media-projection.png)

参考**ScreenRecorder项目<sup>3</sup>**的实现，我们了解到`VirtualDisplay`可以获取当前屏幕的视频流，创建`VirtualDisplay `只需通过`MediaProjectionManager`获取`MediaProjection`，然后通过`MediaProjection`创建`VirtualDisplay`即可。

那么视频数据的流向是怎样的呢？

- 首先，Display 会将画面投影到 VirtualDisplay中；
- 接着，VirtualDisplay 会将图像渲染到 Surface中，而这个Surface是由MediaCodec所创建的；
- 最后，用户可以通过MediaCodec获取特定编码的视频流数据。

经过我们的尝试发现，在这个场景下，MediaCodec只允许使用**video/avc**编码类型，也就是**RAW H.264**的视频编码，使用其他的编码会出现应用Crash的现象（不知是否与硬件有关？）。由于这个视频编码，后面我们与它“搏斗”了好一段时间。

以下是关键部分的代码（来自**ScreenRecorder项目<sup>3</sup>**）：

```java
codec = MediaCodec.createEncoderByType(MIME_TYPE);
mSurface = codec.createInputSurface();
mVirtualDisplay = mMediaProjection.createVirtualDisplay(
		name,
		mWidth,
		mHeight,
		mDpi,
		DisplayManager.VIRTUAL_DISPLAY_FLAG_PUBLIC,
		mSurface,    // 图像会渲染到Surface中
		null,
		null);
```

在编码之前，我们还需要设置视频编码的一些格式信息，这里我们通过`MediaFormat`进行编码格式设置，代码如下（来自**ScreenRecorder项目<sup>3</sup>**）。

```java
private static final String MIME_TYPE = "video/avc"; // H.264编码
private static final int FRAME_RATE = 30;            // 30 FPS
private static final int IFRAME_INTERVAL = 10;       // I-frames间隔时间
private static final int TIMEOUT_US = 10000;

private void prepareEncoder() throws IOException {
    MediaFormat format = MediaFormat.createVideoFormat(MIME_TYPE, mWidth, mHeight);
    format.setInteger(MediaFormat.KEY_COLOR_FORMAT,
            MediaCodecInfo.CodecCapabilities.COLOR_FormatSurface);
    format.setInteger(MediaFormat.KEY_BIT_RATE, mBitRate);
    format.setInteger(MediaFormat.KEY_FRAME_RATE, FRAME_RATE);
    format.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, IFRAME_INTERVAL);

    codec = MediaCodec.createEncoderByType(MIME_TYPE);
    codec.configure(format, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);
    codec.start();
}
```

## 数据获取
![MediaCodec](/img/media-codec.png)
> 图片来自Android官方文档

紧接着，我们需要实时获取视频流了，我们可以直接从`MediaCodec`中获取视频数据。

根据官方文档，获取视频流有两种做法。一种是通过**异步**的方式获取数据，使用回调来获取`OutputBuffer`，具体代码详见[Android文档](https://developer.android.com/reference/android/media/MediaCodec.html)。

这里我们了解一下**同步**获取的方式，由于是同步执行，为了不阻塞主线程，必然需要启动一个新线程来处理。首先，程序会进入一个循环（可以设置变量进行停止），我们通过`codec.dequeueOutputBuffer()`方法获取到`outputBufferId `，接着通过ID获取`buffer`。这个`buffer`即是我们需要用到的**实时视频帧数据**了。代码如下（来自Android官方文档）：

```java
 MediaFormat outputFormat = codec.getOutputFormat(); // 方式二
 codec.start();
 for (;;) {
   int outputBufferId = codec.dequeueOutputBuffer(mBufferInfo, 10000);
   if (outputBufferId >= 0) {
     ByteBuffer outputBuffer = codec.getOutputBuffer(outputBufferId);
     // 方式一
     MediaFormat bufferFormat = codec.getOutputFormat(outputBufferId);
     codec.releaseOutputBuffer(outputBufferId, …);
     
   } else if (outputBufferId == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
     outputFormat = codec.getOutputFormat(); // 方式二
   }
 }
 codec.stop();
 codec.release();
```

按照**ScreenRecorder项目<sup>3</sup>**的做法，接着他使用`MediaMuxer`的`Muxer.writeSampleData()`方法，直接将视频流`outputBuffer `写入了文件。

然而，我们需要的是实时推流至服务器。那么，接下去应该如何实现呢？

## 视频推流
这里有一个小插曲，为了完成这个项目，我和同学查阅了不少资料和源码。其中有一个**RtmpRecoder项目<sup>2</sup>**使用`FFmpeg`进行实时摄像头的`RTMP`推流，推流的原理如下图所示。

![FFmpeg推流](/img/ffmpeg-push.jpg)

[FFmpeg](https://ffmpeg.org/)是一个大名鼎鼎的音视频转码库，它由C语言实现，因此在Java中，我们需要通过JNI进行调用，这里，我们使用了**JavaCV<sup>1</sup>**的FFmpeg转码功能。

<div class="tip">
注意：如果使用JavaCV并采用`mpeg1video`格式推流至服务器，切记将声道调为0，`recorder.setAudioChannels(0)`，否则视频会残缺不全。
</div>

说到这里，不得不吐槽一下**JavaCV<sup>1</sup>**，它没有文档，没有文档是件很可怕的事情，编程基本靠猜。而且它也没有实现FFmpeg的全部功能！！！我们为了把获取到的**RAW H.264**流数据传给JavaCV费了好大功夫，曾经一度想通过调用C语言函数来完成这项工作，但没有成功！

到最后黔驴技穷，只好去项目中开Issue寻求帮助，然而作者表示尚未实现该功能，WTF。好吧，毕竟开源项目，别人也没有义务去做这件事。所以最后也只能自己来解决这个问题了。

![](/img/javacv-issue.png)

废话不多说，既然**JavaCV<sup>1</sup>**无法完成这项工作，那么我们只好另辟蹊径。

现在，有两种做法。

- 自己编写FFmpeg类库。我尝试直接使用CLI接入stream的方式实现实时推流。方法也很简单，只需要在Java中启动`FFmpeg`进程，然后pipe输入流，再由`FFmpeg`推流至服务器。但实践之后发现一些奇怪的问题，只好作罢。
- 另一个方案就是徒手来处理**RAW H.264**流，将转码的工作放到服务器端去实现，最后我们使用这个方案成功完成了任务。下面来看看H.264编码：

# H.264编码
众所周知，视频编码格式种类繁多，H.264也是其中一种编码，每一种编码都有其特点和适用场景，更多信息请自行搜索，这里不多做赘述。期间，我们尝试过将上面获取到的**RAW H.264**数据保存为文件，想研究视频文件为什么会呈现为**绿屏**的画面。经过翻阅资料和试验我们发现，**H.264**编码有着特殊的分层结构。

> H.264 的功能分为两层：视频编码层(VCL, Video Coding Layer)和网络提取层(NAL, Network Abstraction Layer)。VCL 数据即编码处理的输出，它表示被压缩编码后的视频数据 序列。在 VCL 数据传输或存储之前,这些编码的 VCL 数据，先被映射或封装进 NAL 单元中。每个 NAL 单元包括一个原始字节序列负荷(RBSP, Raw Byte Sequence Payload)、一组对应于视频编码的 NAL 头信息。RBSP 的基本结构是：在原始编码数据的后面填加了结尾比特。一个bit“1”若干比特“0”，以便字节对齐。

![NAL](/img/nal.png)

因此，为了将帧序列变成合法的H.264编码，我们需要手动构建**NAL单元**。H.264的帧是以**NAL单元**为单位进行封装的，NAL单元的结构如上图所示。H.264分为`Annexb`和`RTP`两种格式，`RTP`格式更适合用于网络传输，因为其结构更加节省空间，但由于Android系统提供的数据本身就是`Annexb`格式的，因此我们采用`Annexb`格式进行传输。

按照`Annexb`格式的要求，我们需要将数据封装为如下格式：

```
0000 0001 + SPS + 0000 0001 + PPS + 0000 0001 + 视频帧（IDR帧）
```

> H.264的SPS和PPS串，包含了初始化H.264解码器所需要的信息参数，包括编码所用的profile，level，图像的宽和高，deblock滤波器等。

然后不断重复以上格式即可输出正确的**H.264**编码的视频流了。这里的SPS和PPS在每一个NAL单元中重复存在，主要是适用于流式传播的场景，设想一下如果流式传播过程中漏掉了开头的SPS和PPS，那么整个视频流将永远无法被正确解码。

因此我们在实践过程中，SPS和PPS只传递了一次，这样的方式比较适合我们的项目场景，也比较省流量。因此在我们的方案中，格式变为如下形式：

```
0000 0001 + SPS + 0000 0001 + PPS + 0000 0001 + 视频帧（IDR帧）+ 0000 0001 + 视频帧 + ...
```

H.264编码比较复杂，我也只是在做项目期间查阅一些资料才有一点大概的了解，然后在项目完成之后才去反思和理解背后的原理。如果要深入学习，可以查阅相关的资料（**H.264视频压缩标准<sup>5</sup>**）。

介绍完**H.264**的基本原理，下面看看Android上具体的实现。其实Android系统的`MediaCodec`类库已经帮助我们完成了较多的工作，我们只需要在开始录制时（或每一次传输视频帧前）在视频帧之前写入SPS和PPS信息即可。`MediaCodec`已经默认在数据流（视频帧和SPS、PPS）之前添加了`start code`(0x01)，我们不需要手动填写。

SPS和PPS分别对应了`bufferFormat`中的`csd-0`和`csd-1`字段。

```java
...

} else if (outputBufferId == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
	MediaFormat outputFormat = codec.getOutputFormat();
	outputFormat.getByteBuffer("csd-0");    // SPS
	outputFormat.getByteBuffer("csd-1");    // PPS
	/* 然后直接写入传输流 */
}
   
```

# 服务器端
实时的数据流通过Socket(tcp)传输到服务器端，服务器端采用`Node.js`实现视频流转码和`WebSocket`转播。为了使前端可以播放实时的视频，我们必须将格式转换为前端支持的视频格式，这里解码使用`FFmpeg`的Node.js封装（**stream-transcoder项目<sup>6</sup>**）。以下是Socket通讯和转码的关键代码：

```js
var Transcoder = require('stream-transcoder');
var net = require('net');

net.createServer(function(sock) {

    sock.on('close', function(data) {
        console.log('CLOSED: ' +
            sock.remoteAddress + ' ' + sock.remotePort);
    });

    sock.on('error', (err) => {
        console.log(err)
    });
    
    // 转码  H.264 => mpeg1video
    new Transcoder(sock)
      .size(width, height)
      .fps(30)
      .videoBitrate(500 * 1000)
      .format('mpeg1video')
      .channels(0)
      .stream()
      .on('data', function(data) {
        // WebSocket转播
        socketServer.broadcast(data, {binary:true});
      })

}).listen(9091);
```

# Web直播
紧接着，Web前端与服务器建立`WebSocket`连接，使用**jsmpeg项目<sup>7</sup>**对mpeg1video的视频流进行解码并呈现在Canvas上。

```js
var client = new WebSocket('ws://127.0.0.1:9092/');

var canvas = document.getElementById('videoCanvas');
var player = new jsmpeg(client, {canvas:canvas});
```

后续还可以做一些灵活的配置以及错误处理，可以让整个直播的流程更加稳定。至于视频方面的优化，也可以继续尝试各种参数的调节等等。

为了完成这个项目，我们前后花费了四五天的时间，进行各种摸索和尝试，所以我决定记录下这个方案，希望可以帮到有需要的人。

## 其他
1. 参考[JavaCV](https://github.com/bytedeco/javacv)项目
2. 参考[RtmpRecoder开源项目](https://github.com/beautifulSoup/RtmpRecoder)的实现
3. 参考[ScreenRecorder开源项目](https://github.com/yrom/ScreenRecorder)的实现
4. 参考[Android文档](https://developer.android.com/reference/android/media/MediaCodec.html)
5. 文献[H.264视频压缩标准](http://www.axis.com/files/whitepaper/wp_h264_34203_cn_0901_lo.pdf)
6. 使用[stream-transcoder项目](https://github.com/trenskow/stream-transcoder.js)
6. 使用[jsmpeg项目](https://github.com/phoboslab/jsmpeg)