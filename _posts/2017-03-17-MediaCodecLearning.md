---
title: "MediaCodecLearning"
date: 2017-03-17
categories:
  - blog
tags: []
---

# <a href="#MediaCodec编码与解码学习笔记" class="headerlink" title="MediaCodec编码与解码学习笔记"></a>MediaCodec编码与解码学习笔记

上了秦爷的车，进了视频渲染的坑。

这两天跟着几个Demo敲了一下MediaCodec的编码和解码，记录一下。

<a href="https://developer.android.com/reference/android/media/MediaCodec.html" target="_blank" rel="noopener">AndroidDeveloper MediaCodec</a>

官方的文档写的很详细，应该要仔细去翻阅一下

## <a href="#概述" class="headerlink" title="概述"></a>概述

​MediacCodec是Android平台的一个音视频的编解码类，是比较底层的多媒体工具类，经常配合使用的有MediaFormat（媒体格式类）、MediaExtractor（抓取多媒体轨道）、MediaMuxer（混合轨道）、Surface（容器）这几个类。

​MediaCodec主要的工作过程就是它持有一个InputBuffer和OutputBuffer，使用的时候，根据需要提供输入流，交给codec处理，然后拿到输出流作为处理后的结果。这是一个典型的“生产者-消费者”模型。

![MediaCodecStates](/img/codecState.png)

上面是Codec的状态机

使用的主要流程就是通过MediaFormat来配置MediaCodec 进入Configured状态 然后start 不断地取出输入流 直到EOS标识，使用完后释放占用的资源。

<span id="more"></span>

## <a href="#编码" class="headerlink" title="编码"></a>编码

这里以编码本地的一个mp4视频输入，用一个Surface作为输出的容器。

看一下主要的代码:

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90</code></pre></td>
<td class="code"><pre><code>private class PlayerThread extends Thread {
     //主要用到的类
        private MediaExtractor extractor; //轨道抓取
        private MediaCodec decoder; //解码器
        private Surface surface; // 内容容器
  
        public PlayerThread(Surface surface) {
            this.surface = surface;
        }
        @Override
        public void run() {
                extractor = new MediaExtractor();
                extractor.setDataSource(SAMPLE); //设置元数据 这里的SAMPLE是视频文件的路径

            for (int i = 0; i &lt; extractor.getTrackCount(); i++) {
                     MediaFormat format = extractor.getTrackFormat(i);
                    String mime = format.getString(MediaFormat.KEY_MIME);

                    if(mime.startsWith(&quot;video/&quot;)) {//如果是我们想要的视频轨的格式 
                      // 音频轨的开头一般是 &quot;audio/&quot;
                        extractor.selectTrack(i);
                      //生成对应的解码器
                        decoder = MediaCodec.createDecoderByType(mime);

                        //MediaFormat 有几个必须设置的属性 不设置decoder会报初始化异常
                        format.setInteger(MediaFormat.KEY_BIT_RATE, 870000);
                        format.setInteger(MediaFormat.KEY_SAMPLE_RATE,44100 );
                        format.setInteger(MediaFormat.KEY_CHANNEL_COUNT, 1);
                        decoder.configure(format,surface,null,0);
                        break;
                    }
                }

                if(decoder == null){
                    Log.d(TAG, &quot;can&#39;t find video info&quot;);
                    return;
                }
                decoder.start();

            ByteBuffer[] inputBuffers = decoder.getInputBuffers();
            ByteBuffer[] outputBuffers = decoder.getOutputBuffers();

            MediaCodec.BufferInfo info = new MediaCodec.BufferInfo();
            boolean isEOS = false ;
            long startMs = System.currentTimeMillis();

            while(!Thread.interrupted()) {
                if(!isEOS) {
                    int inIndex = decoder.dequeueInputBuffer(10000);
                    if(inIndex &gt;= 0 ) {
                        ByteBuffer buffer = inputBuffers[inIndex];
                        int sampleSize = extractor.readSampleData(buffer,0);
                        if(sampleSize &lt; 0) {
                         //不终止 把eos标志传给 encoder 我们会再次 从 outputBuffer 中得到这个标志
                             Log.d(TAG, &quot;InputBuffer BUFFER_FALG_END_OF_STREAM&quot;);                                        decoder.                         queueInputBuffer(inIndex,0,0,0,MediaCodec.BUFFER_FLAG_END_OF_STREAM);
                              isEOS = true;
                            } else {
                                decoder.queueInputBuffer(inIndex,0,sampleSize,extractor.getSampleTime(),0);
                                //读取下一条数据
                                extractor.advance();
                            }
                        }
                    }
                  //后一个参数是设置等待时间 -1为永久等待
                    int outIndex = decoder.dequeueOutputBuffer(info,10000);
                    switch (outIndex) {
                        //.. 对一些异常情况进行处理
                        default:
                            ByteBuffer buffer = outputBuffers[outIndex];
                            // keep the fps using a simple clock
                            while(info.presentationTimeUs / 1000 &gt; System.currentTimeMillis() - startMs) {
                                    sleep(10);
                                    break;
                                
                            }
                            decoder.releaseOutputBuffer(outIndex,true);
                            break;
                    }

                    if( (info.flags &amp; MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
                        Log.d(TAG, &quot;BUFFER_FLAG_END_OFSTREAM&quot;);
                    }
                }
                decoder.stop();
          
                decoder.release();
                extractor.release();
        }
   
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

对于视频的解码主要是以下几个步骤:

1、用MediaExtractor类抓取我们想要的音频轨的格式，生成对应的MediaFormat类format和MediaCodec类decoder

2、**对生成的format类进行一些必要参数的设置，不然会报错**，然后将decoder进行配置，启动decoder

3、利用decoder的inputBuffers，通过extractor将数据以ByteBuffer的形式送到inputBuffer里，注意一下eos的处理

4、然后再处理输出流，直到同样出现EOS标记

5、停止decoder，释放相关资源

需要注意的是，因为这里只对视频轨进行了解码，所以效果是只有画面没有声音的，如果希望能够有声音，需要再开一个解码器同时对音频轨进行解码，并且涉及到复杂的同步。

> 根据时间戳和一个计时器来控制同步，必要的时候手动掉帧，EXOPlayer源码里有体现。
>
> ——秦爷

暂时就不研究了…

## <a href="#解码" class="headerlink" title="解码"></a>解码

解码的学习主要参考了grafic的soft\_input\_movie demo

代码主要的目的是通过Canvas绘图 然后作为输入，编码生成在对应路径生成mp4格式的视频

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14</code></pre></td>
<td class="code"><pre><code>private void generateMovie(File outputFile) {
        try {
            prepareEncoder(outputFile);
            for (int i = 0; i &lt; NUM_FRAMES; i++) {
                drainEncoder(false);
                generateFrame(i);
            }
            drainEncoder(true);
        }catch (IOException e ) {
            e.printStackTrace();
        }finally {
            releaseEncoder();
        }
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这里把解码主要的几个步骤封装成了独立的函数

主要就是 准备编码器( prepareEncoder() ) -&gt; 绘图产生输入 并榨干编码器(generateFrame() 和drainEncoder()) -&gt; 释放资源( releaseEncoder() )

先来看看编码器的准备工作，主要就是format的配置，这一块和解码类似，设置几个必须的参数，以及设置MediaMuxer，muxer释义是多路器/合成器，它的作用和MediaExtractor恰好相反，是把音视频轨合成成文件，这里我们给Muxer指定了输出的文件路径和格式，但是并没有启动它，因为Encoder的数据还没有准备好。

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30</code></pre></td>
<td class="code"><pre><code>private void prepareEncoder(File outputFile) throws IOException {
      mBufferInfo = new MediaCodec.BufferInfo();
      MediaFormat format = MediaFormat.createVideoFormat(MIME_TYPE,WIDTH,HEIGHT);
  
      // set necessary properties for the format
      format.setInteger(MediaFormat.KEY_COLOR_FORMAT,MediaCodecInfo.CodecCapabilities
          .COLOR_FormatSurface);
      format.setInteger(MediaFormat.KEY_BIT_RATE,BIT_RATE);
      format.setInteger(MediaFormat.KEY_FRAME_RATE,FRAMES_PER_SECOND);
      format.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL,IFRAME_INTERVAL);
      if(VERBOSE) Log.d(TAG, &quot;format:&quot; + format);

      // create a media encoder and configure it with our format. get a surface
      mEncoder = MediaCodec.createEncoderByType(MIME_TYPE);
      mEncoder.configure(format,null,null,MediaCodec.CONFIGURE_FLAG_ENCODE);
      mInputSurface = mEncoder.createInputSurface();
      mEncoder.start();

      // Create a MediaMuxer.  We can&#39;t add the video track and start() the muxer here,
      // because our MediaFormat doesn&#39;t have the Magic Goodies.  These can only be
      // obtained from the encoder after it has started processing data.
      //
      // We&#39;re not actually interested in multiplexing audio.  We just want to convert
      // the raw H.264 elementary stream we get from MediaCodec into a .mp4 file.
      if(VERBOSE) Log.d(TAG, &quot;output will go to &quot; + outputFile);
      mMuxer = new MediaMuxer(outputFile.getPath(),
              MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4);
      mTrackIndex = -1;
      mMuxerStarted = false;
  }</code></pre></td>
</tr>
</tbody>
</table>
</figure>

在上大块头的drainEncoder代码之前我们先看最后一个函数 realeaseEncoder() 释放相关的资源。  
**这里最后给mEncoder 和mMuxer赋值为null 是比较优雅的写法，相当于通知GC来回收，更彻底的节省了资源**

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16</code></pre></td>
<td class="code"><pre><code>private void releaseEncoder() {
       if(VERBOSE) Log.d(TAG, &quot;release encode objects&quot;);
       if(mEncoder != null) {
           mEncoder.stop();
           mEncoder.release();
           mEncoder = null ;
       }
       if(mMuxer != null) {
           mMuxer.stop();
           mMuxer.release();
           mMuxer = null ;
       }
       if(mInputSurface != null) {
           mInputSurface.release();
       }
   }</code></pre></td>
</tr>
</tbody>
</table>
</figure>

接下来就是重头戏drainEncoder了，这里给他传入的参数是一个EOS标志，我们在Canvas上的绘图结束之后再给他传EOS，标志着输入流的结束。

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80</code></pre></td>
<td class="code"><pre><code>
private void drainEncoder(boolean endOfStream) {
      //设置超时时间
    final int TIMEOUT_USEC = 10000;

    if(endOfStream) {
        if(VERBOSE) Log.d(TAG, &quot;sending eos to encoder&quot;);
        mEncoder.signalEndOfInputStream();
    }

    ByteBuffer[] encoderOutputBuffers = mEncoder.getOutputBuffers();
    while(true) {
         //获取编码的状态，让输出流缓冲队列出列，dequeueOutputBuffer会返回一个int类型的值作为编码状态
        int encodeStatus = mEncoder.dequeueOutputBuffer(mBufferInfo,TIMEOUT_USEC);
      //处理各种可能出现的编码状态
        if(encodeStatus == MediaCodec.INFO_TRY_AGAIN_LATER) {
            // no output available yet
            if(!endOfStream) {
                break; // out of while
            } else {
                if(VERBOSE) Log.d(TAG, &quot;no output available ,spinning to await EOS&quot;);
            }
        } else if (encodeStatus == MediaCodec.INFO_OUTPUT_BUFFERS_CHANGED) {
            //not expected for an encoder
            encoderOutputBuffers = mEncoder.getOutputBuffers();
        } else if (encodeStatus == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED) {
            // should happen before receiving buffers ,and should only happen once
            if(mMuxerStarted) {
                throw new RuntimeException(&quot;format changed twice&quot;);
            }
            // the first time format change 
          //format改变 意味着要开始将得到的数据编码成mp4了，启动mMuxer
            MediaFormat newFormat = mEncoder.getOutputFormat();
            Log.d(TAG, &quot;encoder output format changed : &quot; + newFormat);
            // now we have the magic goodies ,start the muxer
            mTrackIndex = mMuxer.addTrack(newFormat);
            mMuxer.start();
            mMuxerStarted = true;
        } else if (encodeStatus &lt; 0) {
            Log.w(TAG, &quot;unexpected result from encoder dequeue output buffers&quot; +
                encodeStatus);
        } else {
          // 启动了mMuxer之后的逻辑 以Byte为单位来读取数据
            ByteBuffer encodeData = encoderOutputBuffers[encodeStatus];
            if(encodeData == null) {
                throw new RuntimeException(&quot;encoder data &quot; + encodeStatus + &quot;is null&quot;);
            }
            if((mBufferInfo.flags &amp; MediaCodec.BUFFER_FLAG_CODEC_CONFIG) != 0) {
                // The codec config data was pulled out and fed to the muxer when we got the INFO_OUTPUT_FORMAT_CHANGED status.  Ignore it.
                if (VERBOSE) Log.d(TAG, &quot;ignoring BUFFER_FLAG_CODEC_CONFIG&quot;);
                mBufferInfo.size = 0;
            }
            if(mBufferInfo.size != 0) {
                if(!mMuxerStarted) {
                    throw new RuntimeException(&quot;muxer hasn&#39;t started&quot;);
                }
              //这段代码是定位我们要写入的数据，通过在dequeueOutputBuffer中传入的mBufferInfo来设置
                encodeData.position(mBufferInfo.offset);
                encodeData.limit(mBufferInfo.offset+mBufferInfo.size);
              
                mBufferInfo.presentationTimeUs = mFakePts;
                mFakePts += 1000000L /FRAMES_PER_SECOND;
              //通过mMuer写入编码后的数据
                mMuxer.writeSampleData(mTrackIndex,encodeData,mBufferInfo);
                if(VERBOSE) Log.d(TAG, &quot;send &quot; + mBufferInfo.size + &quot;bytes to muxer &quot;);
            }
          //释放持有的OutputBuffer
            mEncoder.releaseOutputBuffer(encodeStatus,false);

            if((mBufferInfo.flags &amp; MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
                if(!endOfStream) {
                    Log.d(TAG, &quot;reach end of stream unexpretly&quot;);
                } else {
                    if(VERBOSE) Log.d(TAG, &quot;reach end of stream&quot;);
                }
                break; // break of a while
            }
        }
    }
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

## <a href="#总结" class="headerlink" title="总结"></a>总结

​其实编码的样例代码的函数封装基本上就代表了使用MediaCodec的几个主要的步骤了，编码和解码，其实都大同小异。无非一个是利用MediaExtractor从已有文件中抓取轨道读取数据，而另一个则是将较为底层的数据利用MediaMuxer来合成。使用的时候还是要多加小心，里面有很多不小心就会踩的坑。

​PS：图书馆的插座居然都没有电，我已经要报警了!!!

## <a href="#参考资料" class="headerlink" title="参考资料"></a>参考资料

[grafic](%5Bhttps://github.com/google/grafika%5D)

[MediaCodecDemo](%5Bhttps://github.com/vecio/MediaCodecDemo%5D)

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color3">Android</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color1">MediaCodec</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2017/03/17/MediaCodecLearning/)