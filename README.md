[![license](https://img.shields.io/badge/license-BSD--3--Clause-blue.svg)](https://img.shields.io/badge/license-BSD--3--Clause-blue.svg)

我在寻找如何开始使用[FFmpeg](https://www.ffmpeg.org/)库(即libav)相关的教程或书籍时看到了 ["How to write a video player in less than 1k lines"](http://dranger.com/ffmpeg/) 教程。
非常不幸的是，它已经过时了，于是我决定写这篇文章。

这里的大多数代码是用C写的，**莫慌**： 你可以很容易理解并方便地应用到你常用的语言中去。
FFmpeg libav 有许多其他语言版本，如： [python](https://mikeboers.github.io/PyAV/), [go](https://github.com/imkira/go-libav) 即使没有你使用的语言，你仍然可以通过`ffi` (here's an example with [Lua](https://github.com/daurnimator/ffmpeg-lua-ffi/blob/master/init.lua))来支持。

我们一开始将对视频、音频、编码器、容器等概念做一个快速的概览，然后快速地过一下如何使用 `FFmpeg` 命令行，最后开始写代码。也可以直接跳到[ ](http://newmediarockstars.com/wp-content/uploads/2015/11/nintendo-direct-iwata.jpg)[Learn FFmpeg libav the Hard Way.](#learn-ffmpeg-libav-the-hard-way)

人们常说网络视频流是传统TV的未来，因此FFmpeg是值得去学习的。

__Table of Contents__

* [Intro](#intro)
  * [video - what you see!](#video---what-you-see)
  * [audio - what you listen!](#audio---what-you-listen)
  * [codec - shrinking data](#codec---shrinking-data)
  * [container - a comfy place for audio and video](#container---a-comfy-place-for-audio-and-video)
* [FFmpeg - command line](#ffmpeg---command-line)
  * [FFmpeg command line tool 101](#ffmpeg-command-line-tool-101)
* [Common video operations](#common-video-operations)
  * [Transcoding](#transcoding)
  * [Transmuxing](#transmuxing)
  * [Transrating](#transrating)
  * [Transsizing](#transsizing)
  * [Bonus Round: Adaptive Streaming](#bonus-round-adaptive-streaming)
  * [Going beyond](#going-beyond)
* [Learn FFmpeg libav the Hard Way](#learn-ffmpeg-libav-the-hard-way)
  * [Chapter 0 - The infamous hello world](#chapter-0---the-infamous-hello-world)
    * [FFmpeg libav architecture](#ffmpeg-libav-architecture)
    * [Chapter 0 - code walkthrough](#chapter-0---code-walkthrough)

# Intro

## video - what you see!

如果你有一系列有序的图片，然后按给定的频率改变(假定 [24 张图片每秒](https://www.filmindependent.org/blog/hacking-film-24-frames-per-second/))，你就可以创建一个 [动画](https://en.wikipedia.org/wiki/Persistence_of_vision).
总而言之，这是视频背后最基本的原理： **一系列图片 / 以给定速率播放这些帧**.

<img src="https://upload.wikimedia.org/wikipedia/commons/1/1f/Linnet_kineograph_1886.jpg" title="flip book" height="280"></img>

Zeitgenössische Illustration (1886)

## audio - what you listen!

尽管一个复合的视频可以传递大量的感觉，但加入音频可以带来更多的愉悦体验。

声音是通过空气或其他媒介（如气体、液体或固体）传播的类似压力波的震动。

> 在数字音频系统中，麦克风将声音转换为模拟电子信号，然后利用模数转换（ADC）——常用 [pulse-code modulation—converts (PCM)](https://en.wikipedia.org/wiki/Pulse-code_modulation) 将模拟信号转换为数字信号。

![audio analog to digital](https://upload.wikimedia.org/wikipedia/commons/thumb/c/c7/CPT-Sound-ADC-DAC.svg/640px-CPT-Sound-ADC-DAC.svg.png "audio analog to digital")
>[Source](https://commons.wikimedia.org/wiki/File:CPT-Sound-ADC-DAC.svg)

## codec - shrinking data

> CODEC 是一种电子电路或者软件可以**压缩或解压数字音频及视频.** 它将原始（未压缩）的数字音频/视频转换为压缩的格式或其他副本。
> https://en.wikipedia.org/wiki/Video_codec

如果我们要打包成百上千万张图片到一个单一的文件（即所谓的电影）中我们可能最终会得到一个超级大的文件。我们来做个计算：

假设我们要创建一个分辨率为`1080 x 1920` (高 x 宽)视频，每个像素（屏幕上最小的点）用`3 字节`来编码颜色(或[24位真彩色](https://en.wikipedia.org/wiki/Color_depth#True_color_.2824-bit.29), 它可以编码16,777,216种不同颜色)，帧率为`24 帧每秒`，总长 `30分钟`。 

```c
toppf = 1080 * 1920 //每一帧的总像素
cpp = 3 //编码每个像素需要的字节数
tis = 30 * 60 //总秒数
fps = 24 //每秒的帧数
需要存储空间 = tis * fps * toppf * cpp
```

这个视频大约需要`250.28GB`存储空间或`1.11Gbps`的带宽！这就是为什么我们需要 [CODEC](https://github.com/leandromoreira/digital_video_introduction#how-does-a-video-codec-work).

## container - a comfy place for audio and video

> 容器或封装格式是一种定义了描述不同的元素或数据与数据标签如何共存于一个计算机文件的元文件数据格式。
> https://en.wikipedia.org/wiki/Digital_container_format

一个 **单一文件包含了所有数据流** （绝大多数是音频和视频流）同时还有**同步和一般数据元**，比如标题、分辨率等。

通常我们通过文件扩展名来猜测一个文件的格式，如：`video.webm`可能是使用[`webm`](https://www.webmproject.org/)容器的视频。

![container](/img/container.png)

# FFmpeg - command line

> 一个完整的、交叉平台音视频流记录、转换解决方案。

我们可以利用神一样的[FFmpeg](https://www.ffmpeg.org/)工具库来制作多媒体。或许你已经知道或使用并直接或间接的在使用了。(有用 [Chrome？](https://www.chromium.org/developers/design-documents/video)).

它有一个简单但非常强大的命令行程序叫做`ffmpeg`。来看个例子，将`mp4`转换为`avi`只要在命令行输入：

```bash
$ ffmpeg -i input.mp4 output.avi
```

我们在这里只要做**remuxing**，将一种容器转换为另一种。我们后面还会讲到另一种操作：转码。

## FFmpeg command line tool 101

FFmpeg的文档[documentation](https://www.ffmpeg.org/ffmpeg.html)做得非常好，解释了它如何工作。

我们简化一下FFmpeg的命令行程序的参数格式形如：`ffmpeg {1} {2} -i {3} {4} {5}`，这里：

1. 全局选项
2. 输入文件选项
3. 输入文件
4. 输出文件选项
5. 输出文件

第2, 3, 4 和 5 可以要多少有多少。
我们用个简单的例子来理解一下：

``` bash
# WARNING: this file is around 300MB
$ wget -O bunny_1080p_60fps.mp4 http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_60fps_normal.mp4

$ ffmpeg \
-y \ # 全局选项
-c:a libfdk_aac -c:v libx264 \ # 输入选项
-i bunny_1080p_60fps.mp4 \ # 输入文件
-c:v libvpx-vp9 -c:a libvorbis \ # 输出选项
bunny_1080p_60fps_vp9.webm # 输出文件
```
这个命令有一个输入文件`mp4`含有两个流(音频流采用`aac`编码、视频流采用`h264`编码) 并将其转换为`webm`，同时改变了音视频编码器。

我们下面将简化这个命令，要注意到FFmpeg会匹配或猜测一个默认值。例如输入`ffmpeg -i input.avi output.mp4`，用什么音视频编码来产生`output.mp4`呢？

Werner Robitza 写过一篇教程[tutorial about encoding and editing with FFmpeg](http://slhck.info/ffmpeg-encoding-course/#/)，非常值得一读。

# Common video operations

While working with audio/video we usually do a set of tasks with the media.
说到搞音视频通常是搞一堆媒体工作。（怎么翻？）

## Transcoding

![transcoding](/img/transcoding.png)

**是什么？** 将流（音频、视频）从一种编码方式转为另一种。

**为什么？** 有时候有些设备（电视、智能手机、控制台等）不支持这种但支持另一种，且新编码器有更好压缩率。

**怎么做？** 将`H264` (AVC) 视频转为 `H265` (HEVC)。
```bash
$ ffmpeg \
-i bunny_1080p_60fps.mp4 \
-c:v libx265 \
bunny_1080p_60fps_h265.mp4
```

## Transmuxing

![transmuxing](/img/transmuxing.png)

**是什么？** 从一种格式转换为另一种。（换一种容器，流的编码方式未改变。）

**为什么？** 有时候有些设备（电视、智能手机、控制台等）不支持这种但支持另一种，且有时新容器有更先进的特性。

**怎么做？** 将`mp4`转为`webm`.
```bash
$ ffmpeg \
-i bunny_1080p_60fps.mp4 \
-c copy \ # just saying to ffmpeg to skip encoding
bunny_1080p_60fps.webm
```

## Transrating

![transrating](/img/transrating.png)

**是什么？** 转换码率。

**为什么？** 别人希望在`2G` (edge)网络下观看你的视频，更差的智能手机或用`光纤` 网络连接他们的4K电视，因此你需要提供多种不码率的视频。

**怎么做？** 产生3856K及2000K.
```bash
$ ffmpeg \
-i bunny_1080p_60fps.mp4 \
-minrate 964K -maxrate 3856K -bufsize 2000K \
bunny_1080p_60fps_transrating_964_3856.mp4
```

一般我们常将变换码率和缩放尺寸一起使用。 Werner Robitza写了另一篇值得深读的文章[series of posts about FFmpeg rate control](http://slhck.info/posts/)。

## Transsizing

![transsizing](/img/transsizing.png)

**是什么** 从一另分辨率转换为另一种，如前述变换尺寸常常也会转换码率。

**为什么？** 原因同变换码率。

**怎么做？** 将`1080p`转为`480p`。
```bash
$ ffmpeg \
-i bunny_1080p_60fps.mp4 \
-vf scale=480:-1 \
bunny_1080p_60fps_transsizing_480.mp4
```

## Bonus Round: Adaptive Streaming

![adaptive streaming](/img/adaptive-streaming.png)

**是什么？** 用于生成多种分辨率（码率），将媒体分割成块通过http提供服务。

**为什么？** 提供一种便于既可在低端智能手机又可以在4K电视上观看的视频，还可以通过图标简单的进行缩放。

**怎么做？** 通过DASH创建一个可伸缩的WebM。
```bash
# video streams
$ ffmpeg -i bunny_1080p_60fps.mp4 -c:v libvpx-vp9 -s 160x90 -b:v 250k -keyint_min 150 -g 150 -an -f webm -dash 1 video_160x90_250k.webm

$ ffmpeg -i bunny_1080p_60fps.mp4 -c:v libvpx-vp9 -s 320x180 -b:v 500k -keyint_min 150 -g 150 -an -f webm -dash 1 video_320x180_500k.webm

$ ffmpeg -i bunny_1080p_60fps.mp4 -c:v libvpx-vp9 -s 640x360 -b:v 750k -keyint_min 150 -g 150 -an -f webm -dash 1 video_640x360_750k.webm

$ ffmpeg -i bunny_1080p_60fps.mp4 -c:v libvpx-vp9 -s 640x360 -b:v 1000k -keyint_min 150 -g 150 -an -f webm -dash 1 video_640x360_1000k.webm

$ ffmpeg -i bunny_1080p_60fps.mp4 -c:v libvpx-vp9 -s 1280x720 -b:v 1500k -keyint_min 150 -g 150 -an -f webm -dash 1 video_1280x720_1500k.webm

# audio streams
$ ffmpeg -i bunny_1080p_60fps.mp4 -c:a libvorbis -b:a 128k -vn -f webm -dash 1 audio_128k.webm

# the DASH manifest
$ ffmpeg \
 -f webm_dash_manifest -i video_160x90_250k.webm \
 -f webm_dash_manifest -i video_320x180_500k.webm \
 -f webm_dash_manifest -i video_640x360_750k.webm \
 -f webm_dash_manifest -i video_640x360_1000k.webm \
 -f webm_dash_manifest -i video_1280x720_500k.webm \
 -f webm_dash_manifest -i audio_128k.webm \
 -c copy -map 0 -map 1 -map 2 -map 3 -map 4 -map 5 \
 -f webm_dash_manifest \
 -adaptation_sets "id=0,streams=0,1,2,3,4 id=1,streams=5" \
 manifest.mpd
```

PS: 这个例子是我从[Instructions to playback Adaptive WebM using DASH](http://wiki.webmproject.org/adaptive-streaming/instructions-to-playback-adaptive-webm-using-dash) 偷来的。

## Going beyond

它们在 [many and many other usages for FFmpeg](https://github.com/leandromoreira/digital_video_introduction/blob/master/encoding_pratical_examples.md#split-and-merge-smoothly).
I use it in conjunction with *iMovie* to produce/edit some videos for YouTube and you can certainly use it professionally.

# Learn FFmpeg libav the Hard Way

> Don't you wonder sometimes 'bout sound and vision?
> **David Robert Jones**

虽然[FFmpeg](#ffmpeg---command-line)是非常有用的处理媒体文件命令行工具，但是我们怎么在我们的程序中使用它呢？

FFmpeg[由数个库组成](https://www.ffmpeg.org/doxygen/trunk/index.html) 可以被集成到我们程序中。一般在我们安装FFmpeg时也自动安装了它的所有库。也就是我前面也提到过的**FFmpeg libav**。

> 这个标题意于Zed Shaw's [Learn X the Hard Way](https://learncodethehardway.org/)系列，特别是他的《Learn C the Hard Way》。

## Chapter 0 - The infamous hello world
这个“hello world”实际上是在终端展示`"hello world"`。 :tongue:我们将以**打印输出视频信息**，一些像格式（容器）、时长、分辨率、音频通道数等信息，然后在最后，我们会**编码一些帧并存储为图像文件**。

### FFmpeg libav architecture

在我们写代码之前，先学习一下**FFmpeg libav架构**如何工作以及它的组件之间如何通信。

这是一个视频编码处理的框架图：

![ffmpeg libav architecture - decoding process](/img/decoding.png)

首先，你需要将你的媒体文件加载到一个叫做[`AVFormatContext`](https://ffmpeg.org/doxygen/trunk/structAVFormatContext.html) 组件中（视频容器也即格式）。它实际上并不是加载整体文件，一般仅读入文件头信息。

一但加载完了最少的**容器头部信息**，我们就可以访问到它所有的流（把它想像成原始的音频和视频数据）。每个流可以被用于叫做[`AVStream`](https://ffmpeg.org/doxygen/trunk/structAVStream.html)组件。

> Stream is a fancy name for a continuous flow of data.（不是很认同，这句不翻。）

假设我们的视频（原文是video，这里应该叫文件吧，否则会有点歧义）有两个流：一个用[AAC CODEC](https://en.wikipedia.org/wiki/Advanced_Audio_Coding) 编码的音频流和一个用[H264 (AVC) CODEC](https://en.wikipedia.org/wiki/H.264/MPEG-4_AVC)编码的视频流。从每个流中，我们可以提取出**pieces (slices) of data（一片？一截？一块？原文更好理解）**叫做包，并载入到叫做[`AVPacket`](https://ffmpeg.org/doxygen/trunk/structAVPacket.html)的组件中。

在上面**包中的数据仍然是被编码过的** (压缩) ，为了解压这些包，我们需要将其传入[`AVCodec`](https://ffmpeg.org/doxygen/trunk/structAVCodec.html)。

`AVCodec`会将它们解压到[`AVFrame`](https://ffmpeg.org/doxygen/trunk/structAVFrame.html)中，最终，这个组件会返回**未压缩的帧**。注意，音频和视频是同一套的处理过程。

### Chapter 0 - code walkthrough

> #### TLDR; show me the [code](/0_hello_world.c) and execution.
> ```bash
> # WARNING: 此文件大约 300MB
> $ make
> ```

我们会跳过一些信息，但别慌，完整代码在这里： [source code is available at github](/0_hello_world.c)。（Makefile有问题，我编译没通过过，只要把.c文件放到链接库前就行了，gcc-4.8，搞了很久，晕。）

我们第一件需要做的事情就是注册所有编码器，格式和协议。我们只要调用函数[`av_register_all`](http://ffmpeg.org/doxygen/trunk/group__lavf__core.html#ga917265caec45ef5a0646356ed1a507e3)就行了：

```c
av_register_all();
```

接下来为组件[`AVFormatContext`](http://ffmpeg.org/doxygen/trunk/structAVFormatContext.html)分配内存，它持有关于格式（容器）的信息。

```c
AVFormatContext *pFormatContext = avformat_alloc_context();
```

然后打开文件，读入头部信息（最少的格式相关信息）并填充到`AVFormatContext` (注意，一般这里还没有打开编码器)。
函数[`avformat_open_input`](http://ffmpeg.org/doxygen/trunk/group__lavf__decoding.html#ga31d601155e9035d5b0e7efedc894ee49)用来完成这个操作。它需要一个`AVFormatContext`、一个`filename`以及两个可选参数：[`AVInputFormat`](https://ffmpeg.org/doxygen/trunk/structAVInputFormat.html) (若使用`NULL`，FFmpeg会猜测一个匹配格式)还有[`AVDictionary`](https://ffmpeg.org/doxygen/trunk/structAVDictionary.html) (这个是解复用选项)。

```c
avformat_open_input(&pFormatContext, filename, NULL, NULL);
```

打印格式名及媒体时长：

```c
printf("Format %s, duration %lld us", pFormatContext->iformat->long_name, pFormatContext->duration);
```

我们需要从媒体中读入数据来使用`streams`，函数[`avformat_find_stream_info`](https://ffmpeg.org/doxygen/trunk/group__lavf__decoding.html#gad42172e27cddafb81096939783b157bb)实现这个操作。
现在，`pFormatContext->nb_streams`为流总数，而`pFormatContext->streams[i]`是第`i`个流(一个[`AVStream`](https://ffmpeg.org/doxygen/trunk/structAVStream.html))。

```c
avformat_find_stream_info(pFormatContext,  NULL);
```

遍历所有流。

```c
for (int i = 0; i < pFormatContext->nb_streams; i++)
{
  //
}
```

对于每个流，我们需要保持[`AVCodecParameters`](https://ffmpeg.org/doxygen/trunk/structAVCodecParameters.html)，它描述了i流的编码器特性。

```c
AVCodecParameters *pLocalCodecParameters = pFormatContext->streams[i]->codecpar;
```

利用函数[`avcodec_find_decoder`](https://ffmpeg.org/doxygen/trunk/group__lavc__decoding.html#ga19a0ca553277f019dd5b0fec6e1f9dca)可以查看编码器特性，以及通过codec id返回[`AVCodec`](http://ffmpeg.org/doxygen/trunk/structAVCodec.html)从而找出编码器。 这个组件知道如何 en**CO**de and **DEC**ode流。
```c
AVCodec *pLocalCodec = avcodec_find_decoder(pLocalCodecParameters->codec_id);
```

现在我们可以打印编码器相关信息。

```c
// 音视频规格
if (pLocalCodecParameters->codec_type == AVMEDIA_TYPE_VIDEO) {
  printf("Video Codec: resolution %d x %d", pLocalCodecParameters->width, pLocalCodecParameters->height);
} else if (pLocalCodecParameters->codec_type == AVMEDIA_TYPE_AUDIO) {
  printf("Audio Codec: %d channels, sample rate %d", pLocalCodecParameters->channels, pLocalCodecParameters->sample_rate);
}
// 一般信息
printf("\tCodec %s ID %d bit_rate %lld", pLocalCodec->long_name, pLocalCodec->id, pCodecParameters->bit_rate);
```

有了codec，我们可以分配内存给[`AVCodecContext`](https://ffmpeg.org/doxygen/trunk/structAVCodecContext.html)，它持有解码/编码过程上下文，但之后我们需要填入CODEC参数。我们在[`avcodec_parameters_to_context`](https://ffmpeg.org/doxygen/trunk/group__lavc__core.html#gac7b282f51540ca7a99416a3ba6ee0d16)中操作。

一旦我们填充了codec context， 我们需要打开codec，调用[`avcodec_open2`](https://ffmpeg.org/doxygen/trunk/group__lavc__core.html#ga11f785a188d7d9df71621001465b0f1d)。

```c
AVCodecContext *pCodecContext = avcodec_alloc_context3(pCodec);
avcodec_parameters_to_context(pCodecContext, pCodecParameters);
avcodec_open2(pCodecContext, pCodec, NULL);
```

然后我们从流中读入包并压缩到帧中，不过在此之前，需要分配内存给[`AVPacket`](https://ffmpeg.org/doxygen/trunk/structAVPacket.html) 和 [`AVFrame`](https://ffmpeg.org/doxygen/trunk/structAVFrame.html)。

```c
AVPacket *pPacket = av_packet_alloc();
AVFrame *pFrame = av_frame_alloc();
```

只要流中还有包就调用函数[`av_read_frame`](https://ffmpeg.org/doxygen/trunk/group__lavf__decoding.html#ga4fdb3084415a82e3810de6ee60e46a61) 从流中读入包。

```c
while (av_read_frame(pFormatContext, pPacket) >= 0) {
  //...
}
```

我们来**发送原始数据包** (压缩帧)给编码器，通过codec context, 使用函数 [`avcodec_send_packet`](https://ffmpeg.org/doxygen/trunk/group__lavc__decoding.html#ga58bc4bf1e0ac59e27362597e467efff3)。

```c
avcodec_send_packet(pCodecContext, pPacket);
```

然后我们再**接收原始数据帧** (未压缩的帧) 来自解码器，使用相同的codec context，函数[`avcodec_receive_frame`](https://ffmpeg.org/doxygen/trunk/group__lavc__decoding.html#ga11e6542c4e66d3028668788a1a74217c)。

```c
avcodec_receive_frame(pCodecContext, pFrame);
```

打印帧数，[PTS](https://en.wikipedia.org/wiki/Presentation_timestamp), DTS, [帧类型](https://en.wikipedia.org/wiki/Video_compression_picture_types)等等。

```c
printf(
    "Frame %c (%d) pts %d dts %d key_frame %d [coded_picture_number %d, display_picture_number %d]",
    av_get_picture_type_char(pFrame->pict_type),
    pCodecContext->frame_number,
    pFrame->pts,
    pFrame->pkt_dts,
    pFrame->key_frame,
    pFrame->coded_picture_number,
    pFrame->display_picture_number
);
```

最后，我们将解码了的帧保存为[单灰度图像](https://en.wikipedia.org/wiki/Netpbm_format#PGM_example)。这个操作非常简单，我们使用`pFrame->data` 这里的索引关系是：[planes Y, Cb and Cr](https://en.wikipedia.org/wiki/YCbCr)，我们只要使用`0` (Y)来保存灰度图像。

```c
save_gray_frame(pFrame->data[0], pFrame->linesize[0], pFrame->width, pFrame->height, frame_filename);

static void save_gray_frame(unsigned char *buf, int wrap, int xsize, int ysize, char *filename)
{
    FILE *f;
    int i;
    f = fopen(filename,"w");
    // writing the minimal required header for a pgm file format
    // portable graymap format -> https://en.wikipedia.org/wiki/Netpbm_format#PGM_example
    fprintf(f, "P5\n%d %d\n%d\n", xsize, ysize, 255);

    // writing line by line
    for (i = 0; i < ysize; i++)
        fwrite(buf + i * wrap, 1, xsize, f);
    fclose(f);
}
```

And voilà!哇哦！ 现在我们有一个2MB的灰度图像了：

![saved frame](/img/generated_frame.png)

## Chapter 1 - syncing audio and video

> **做一个播放器** - 一个年轻的JS开发人员写了一个新的MSE视频播放器。

在我们步入[code a transcoding example](#chapter-2---transcoding)之前，我们来讨论一下**时序**，播放器如何知道在一个正确的时间播放帧。

在最后的例子中，我们保存了一些帧，也就是：

![frame 0](/img/hello_world_frames/frame0.png)
![frame 1](/img/hello_world_frames/frame1.png)
![frame 2](/img/hello_world_frames/frame2.png)
![frame 3](/img/hello_world_frames/frame3.png)
![frame 4](/img/hello_world_frames/frame4.png)
![frame 5](/img/hello_world_frames/frame5.png)

当我们在设计一个视频播放器的时候我们需要**在每一个节点播放一帧图像**，否则太快或太慢将会很难受。

因此我们需要介绍一些逻辑使得播放每一帧都平滑。基于此，每个帧都有一个**展现时间戳** (PTS)， 它是一个递增的数字除以一个**时基** ，时基是一个比值 (它的分母**时间尺度**)除以**帧率(fps)**。这一段没看懂，请看原文。

我们来看一些例子就很好理解了，我们来模拟一些场景：

例如一个`fps=60/1`，`timebase=1/60000`每个PTS都会累加`timescale / fps = 1000`，因此**PTS real time**对于每一帧来说(假设从0开始)：

* `frame=0, PTS = 0, PTS_TIME = 0`
* `frame=1, PTS = 1000, PTS_TIME = PTS * timebase = 0.016`
* `frame=2, PTS = 2000, PTS_TIME = PTS * timebase = 0.033`

对于更多类似的场景，但是timebase等于`1/60`。

* `frame=0, PTS = 0, PTS_TIME = 0`
* `frame=1, PTS = 1, PTS_TIME = PTS * timebase = 0.016`
* `frame=2, PTS = 2, PTS_TIME = PTS * timebase = 0.033`
* `frame=3, PTS = 3, PTS_TIME = PTS * timebase = 0.050`

对于`fps=25/1`，`timebase=1/75`每个PTS将`timescale / fps = 3` and the PTS time could be:

* `frame=0, PTS = 0, PTS_TIME = 0`
* `frame=1, PTS = 3, PTS_TIME = PTS * timebase = 0.04`
* `frame=2, PTS = 6, PTS_TIME = PTS * timebase = 0.08`
* `frame=3, PTS = 9, PTS_TIME = PTS * timebase = 0.12`
* ...
* `frame=24, PTS = 72, PTS_TIME = PTS * timebase = 0.96`
* ...
* `frame=4064, PTS = 12192, PTS_TIME = PTS * timebase = 162.56`

现在有了`pts_time`我们就找到一个方法使用系统时钟来渲染和同步音频`pts_time`，FFmpeg libav通过API来提供这些信息：

- fps = [`AVStream->avg_frame_rate`](https://ffmpeg.org/doxygen/trunk/structAVStream.html#a946e1e9b89eeeae4cab8a833b482c1ad)
- tbr = [`AVStream->r_frame_rate`](https://ffmpeg.org/doxygen/trunk/structAVStream.html#ad63fb11cc1415e278e09ddc676e8a1ad)
- tbn = [`AVStream->time_base`](https://ffmpeg.org/doxygen/trunk/structAVStream.html#a9db755451f14e2bf590d4b85d82b32e6)

仅出于好奇，我们保存的帧是以DTS序列发送(帧序： 1,6,4,2,3,5)，但是播放是以PTS序列(帧序： 1,2,3,4,5)。另外，对比看看B-Frames和P及I-Frames有多小。

```
LOG: AVStream->r_frame_rate 60/1
LOG: AVStream->time_base 1/60000
...
LOG: Frame 1 (type=I, size=153797 bytes) pts 6000 key_frame 1 [DTS 0]
LOG: Frame 2 (type=B, size=8117 bytes) pts 7000 key_frame 0 [DTS 3]
LOG: Frame 3 (type=B, size=8226 bytes) pts 8000 key_frame 0 [DTS 4]
LOG: Frame 4 (type=B, size=17699 bytes) pts 9000 key_frame 0 [DTS 2]
LOG: Frame 5 (type=B, size=6253 bytes) pts 10000 key_frame 0 [DTS 5]
LOG: Frame 6 (type=P, size=34992 bytes) pts 11000 key_frame 0 [DTS 1]
```

## Chapter 2 - transcoding
