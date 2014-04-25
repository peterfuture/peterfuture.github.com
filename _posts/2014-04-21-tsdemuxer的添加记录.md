---
layout: post
title: ts demuxer的添加记录
---

{{ page.title }}
================
目录
1 初衷
2 ts demux的功能介绍

1 初衷
    之前打算给dtplayer添加一些亮点功能，最初的想法是：bt下载播放 + hls支持
    bt下载由于以来libtorrent库，虽然搞懂了如何添加，但需要修改libtorrent库来集成，
    若将libtorrent集成到代码中，会将代码变得庞大，框架清晰度会变差，随机暂时取消了bt功能的开发/
   
    后面开始添加hls支持，hls支持打算添加如下模块： stream - hls   demuxer - ts  decoder - h264 这样加上之前的faad
    对于特定网络流就可以不依赖ffmpeg来播放了。
    ts demuxer便是第一步功能。
    凭借之前对ts的理解，感觉应该比较快完成的，当时从github中找到了一个开源的ts解析库：https://github.com/nevali/tsdemux
    但不幸的是这个库只提供了 section的解析功能，并没有提供读取es包的功能。遂打算自行将这部分功能补全。

代码： https://github.com/peterfuture/dtplayer/blob/dev-ts/dtdemux/demuxer/demuxer_ts.c


2 ts demux的功能
    ts demux符合dtplayer demuxer的标准接口，
    先看下定义：
demuxer_wrapper_t demuxer_ts = {
    .name = "ts demuxer",
    .id = DEMUXER_TS, 
    .probe = ts_probe,
    .open = ts_open,
    .read_frame = ts_read_frame,
    .setup_info = ts_setup_info,
    .seek_frame = ts_seek_frame,
    .close = ts_close
};
下面依次介绍各个功能。

2.1 probe
static int ts_probe(demuxer_wrapper_t *wrapper, dt_buffer_t *probe_buf)
{
    const uint8_t *buf = probe_buf->data;
    const uint8_t *end = buf + probe_buf->level - 7;
    
    if(probe_buf->level < 10)
        return 0;
    int retry_times = 100;
    for(; buf < end; buf++) 
    {
        uint32_t header = DT_RB8(buf);
        if((header&0xFF) != 0x47)
        {
            if(retry_times-- == 0)
                return 0;
            continue;
        }
        //found 0x47
        if(buf+188 > end)
            return 0;
        header = DT_RB8(buf+188);
        if((header & 0xFF) == 0x47)
        {
            dt_info(TAG,"ts detect \n");
            return 1;
        }
        else
            return 0;
    }
    return 0;
}
这里实现了一个比较简单的probe功能，先找到同步字0x47,若后面188字节后面也是0x47，则认为是ts流，暂时probe有些流还不能解析
主要问题有： a 有些流的包大小并不是188， 这种流会出错 b 单纯的只判断一次会有偶然性的问题
但这些修正起来比较简单，由于功能还未实现，这里只介绍思想，还有面会完善

2 ts_open
这里是解析ts头信息，主要功能包括：
pat解析
pmt解析
stream 信息解析： duration - bitrate
es信息解析：audio: channel-samplerate-bps  video: width-height-fmt
这里代码比较多就只说下思想，感兴趣的自己去读代码就可以了

首先pat pmt的解析是通过之前说的开源库完成的，通过解析pat pmt可以得到文件中有几个流，分别是什么格式
得到后一般若有video stream,则以video为参考来计算流信息（正确的做法是找到流的pcr_pid，也就是参考流来计算，但一般情况下参考流都是video）
计算duration： 首先找到第一帧的pts ， 然后找到最后一帧的pts， 通过pts差距得到duration
计算bitrate： 有了stream size和 duration， 直接计算得到bitrate

其中计算pts的时候，这里介绍下，具体代码在ts/stream.c中，是后面我自己加的
if(packet->payloadlen > 0 && packet->unitstart)
    {
        uint8_t *pcrbuf = packet->payload;
        int len = pcrbuf[0];
        if(len <= 0 || len > 183)	//broken from the stream layer or invalid
            goto QUIT;
        pcrbuf++;
	int flags = pcrbuf[0];
	int has_pcr;
	has_pcr = flags & 0x10;
        pcrbuf++;
        if(!has_pcr)
            goto QUIT;
        int64_t pcr = -1;
        int64_t pcr_ext = -1;
        unsigned int v = 0;
	   //v = (uint32_t)pcrbuf[3]<<24 | pcrbuf[2]<<16 |	pcrbuf[1]<<8 |pcrbuf[0];
	v = (uint32_t)pcrbuf[0]<<24 | pcrbuf[1]<<16 |	pcrbuf[2]<<8 |pcrbuf[3];
        pcr = ((int64_t)v<<1) | (pcrbuf[4] >> 7);
	pcr_ext = (pcrbuf[4] & 0x01) << 8;
	pcr_ext |= pcrbuf[5];
        pcr = pcr * 300 + pcr_ext
        packet->pts = pcr / 300;
        //printf("get pts:%lld \n",pcr);
    }

这里是去掉了ts包开头的四个字节，首先看是否是一帧的开头，若是的话判断是否有pcr flag
若有则直接解析，解析方法比较简单，按照标准即可
这里介绍下ts的编码参考时钟为：27M HZ，而pts的单位与时间s的换算为：90000
因此获取参考时间（也就是上面的pcr后），换算为pts的计算方式为： pts = pcr * 9000 / 27000000 = pcr / 300
这里得到的就直接是pts了。

还有一个问题是：这里直接解析es流貌似不能获取 视频： width height  音频：channel samplerate等信息，
对于ffmpeg来讲没有问题，因此在av_find_stream_info中可以通过decode one frame来获取， 但dtplayer框架上是不方便直接启动解码器的
对于mplayer 的ts 也是没有decode的，知道的同学可以指导下最好，不胜感激，否则得自己扒mplayer的代码了。
【补充】通过阅读代码，发现并不能从ts流的标准处理中得到具体的流信息，如视频的宽度高度，这样只能先设置为默认值
后面通过解码模块后重新更新视频 音频信息。 这里之前接触一些项目，在得到实际的数据流之前，已经通过网络先传了一些参数过来，这样就直接使用就好
现有的代码，已经可以满足一些基本的ts支持了。

3 ts_setup_info
这里依据ts_open获取的信息，来setup dtplayer的media info
然后选择av视频流就可以了，比较简单

4 ts_read_frame
这里比较重要，也是花了比较多时间的地方，一开始的时候，以为拿到av的pid后，后面直接组装数据就可以了
流程为： 解析ts包获取pid -> 若是选择的pid，直接将payload保存下来，通过简单的判断是否是unit_start来判断是否读取到了完整帧 --> 返回完整帧
但实际情况并不是这样，保存在ts包中的是实际的pes包，而不是es流，在读包的时候需要将pes包头去掉

这里参考ffmpeg进行了模拟，具体代码在：handle_ts_pkt中，具体逻辑不说了，只是更正自己的一个认知误区
不过这里还有个问题，读取的数据包会掺杂错误信息： 经过跟代码，大体定位了问题，在解析pes包头的时候，是可以知道这个包的大小的
但读取ts包的时候都是按照188，即payload一般是184字节，这样很有可能就超过了pes包的大小
此时应该如何处理： 若只读取固定大小凑足包大小返回，则声音是错的
若将所有payload都打包进去，则播放过程会有杂音。
这里还需要研究下。有熟悉的同学也请不吝赐教。
[补充]这里原因找到了，每个ts包不一定都会包含payload或者adaption_field，需要在解析的时候控制下
具体可参考：https://github.com/peterfuture/dtplayer/blob/master/dtdemux/demuxer/ts/stream.c
ts_stream_fill_packet函数中的实现。 

5 ts_seek
这里seek就比较简单了，虽然还未完成，但思想可以说下，参考ffmpeg，一句bitrate seek到某个位置，读取ts包，计算pts
若不匹配，则按照二分查找算法，继续seek
来达到seek的目的。（后面会实现）

这里ts demux只是为了实现一个简单的功能，给后面基于dtplayer开发简单的应用的开发者提供便利： 若服务端也自己做的话，可以很方便的改造dtplayer中的ts demux达到字节的要求，
同时去掉了ffmpeg的负担，岂不非常方便，这也是dtplayer会一直遵循的目标。

github：https://github.com/avplayer/dtplayer   # C++
github：https://github.com/peterfuture/dtplayer # C
bug report: peter_future@outlook.com
blog: http://blog.csdn.net/dtplayer
bbs: http://avboost.com/
wiki: http://wiki.avplayer.org/Dtplayer

由于后面随着开发的进行文章会进行细节的更新，因此为了保证读者随时读到最新的内容，文章禁止转载，多谢大家支持！
