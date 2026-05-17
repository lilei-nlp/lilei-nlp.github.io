---
title: "FilterPlayer"
date: 2017-04-23
categories:
  - blog
tags: []
---

​上次讲了MediaCodec的用法，其实也是为了后面的视频渲染做准备的。一开始定的学习目标就是：能够导入视频，并且加上滤镜。这篇文章就主要讲一下导入视频之后的滤镜渲染。

## <a href="#思路" class="headerlink" title="思路"></a>思路

​其实一开始实现的思路大致是这样:

> MediaCodec解码视频流 ——&gt; 输出到GLSurfaceView ——&gt; OpenGL渲染

​然而因为GLSurfaceView的Surface比较特殊，无法直接作为MediaCodec的Decoder的输出容器，所以失败了。

​后来开始研究 <a href="https://github.com/google/grafika" target="_blank" rel="noopener">grafika</a> 里的一个 Show+capture camera 的功能

> <a href="https://github.com/google/grafika/blob/master/src/com/android/grafika/CameraCaptureActivity.java" target="_blank" rel="noopener">Show + capture camera</a>. Attempts to record at 720p from the front-facing camera, displaying the preview and recording it simultaneously.
>
> - Use the record button to toggle recording on and off.
> - Recording continues until stopped. If you back out and return, recording will start again, with a real-time gap. If you try to play the movie while it’s recording, you will see an incomplete file (and probably cause the play movie activity to crash).
> - The recorded video is scaled to 640x480, so it will probably look squished. A real app would either set the recording size equal to the camera input size, or correct the aspect ratio by letter- or pillar-boxing the frames as they are rendered to the encoder.
> - You can select a filter to apply to the preview. It does not get applied to the recording. The shader used for the filters is not optimized, but seems to perform well on most devices (the original Nexus 7 (2012) being a notable exception). Demo here: <a href="http://www.youtube.com/watch?v=kH9kCP2T5Gg" target="_blank" rel="noopener">http://www.youtube.com/watch?v=kH9kCP2T5Gg</a>
> - The output is a video-only MP4 file (“camera-test.mp4”).

​它的Show模块和我想要做滤镜预览的是非常类似的，只是我的输入源是来自于本地文件，只要用MediaCodec来解码成视频流作为输入就可以了。当然，也和它的说明一样，他的录制功能是不带滤镜的，而我要做的进行添加滤镜之后的保存就要另辟蹊径了（还在努力中）。

​grafika实现的思路是这样的:

> Decoder –&gt; Surface -&gt; SurfaceTexture –&gt; OpenGL Filter –&gt; GLSurfaceView

​关键在于，它并没有直接把GLSurfaceView的Surface交给Deocder作为输出容器，而是中间通过一个FullFrameRect的类来持有了一个OpenGL的Texture,然后我们可以用这个纹理来创建一个Surface交给Decoder。就避免了直接用GLSurfaceView的Surface作为输出的问题。

<span id="more"></span>

## <a href="#关键的类-Texture2dProgram" class="headerlink" title="关键的类  Texture2dProgram"></a>关键的类 Texture2dProgram

​上面说到，FullFrameRect 起到了一个比较重要的作用。来看看代码。

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
3</code></pre></td>
<td class="code"><pre><code>public class FullFrameRect {
    private final Drawable2d mRectDrawable = new Drawable2d(Drawable2d.Prefab.FULL_RECTANGLE);
    private Texture2dProgram mProgram;</code></pre></td>
</tr>
</tbody>
</table>
</figure>

FullFrameRect其实就是一个精灵(sprite)，作为渲染的中间体，真正做工作的，其实是他持有的Texture2dProgram这个类，而这个类也是我们所说的Texture的真正的持有者。来看代码

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
29</code></pre></td>
<td class="code"><pre><code>public class Texture2dProgram {
 
    public enum ProgramType { // 几种program的类型
        TEXTURE_2D, TEXTURE_EXT, TEXTURE_EXT_BW, TEXTURE_EXT_FILT
    }
    private static final String VERTEX_SHADER = &quot;&quot;; // 定点和片段着色器的代码省略了
    private static final String FRAGMENT_SHADER_2D = &quot;&quot; ;
   
      // 以下几个成员变量才是关键
    private int mProgramHandle; // openGL program 实际上就是这个mProgramHandle
    private int muMVPMatrixLoc; // 投影矩阵
    private int muTexMatrixLoc; // 纹理矩阵
    private int muKernelLoc;  // 内核 通过他配合片段着色器来实现滤镜功能
    private int muTexOffsetLoc;
    private int muColorAdjustLoc; // 颜色调整
    private int maPositionLoc;
    private int maTextureCoordLoc;

    private int mTextureTarget;

    private float[] mKernel = new float[KERNEL_SIZE];
    private float[] mTexOffset;
    private float mColorAdjust;
  
  public Texture2dProgram(ProgramType programType) {}
  public int createTextureObject(){}
  public void setKernel(){}
  public void draw(){}
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

看看几个关键的方法:

`Texture2dProgram()` 构造方法，根据programType生成用对应的着色器生成对应的OpenGL program，如果设置有滤镜 会加载kernel数组。

`createTextureObject()` 利用openGL生成TextureObject 交给外界处理。

`setKernel()` 动态的来设置Kernel数组，来实现实时滤镜的切换。

`draw()` 用OpenGL在Texture上绘制

所以其实渲染都是由这个Texture2dProgram完成的，我们通过改变片段着色器的参数来实现视频滤镜的添加。

## <a href="#实现" class="headerlink" title="实现"></a>实现

​不过上面的思路还是有一个小问题，就是这样**渲染出来的视频是没有声音**的，因为在上一篇MediaCodec的文章里讲到，我们在解码的时候只选取的视频轨，而没有选取音频轨，所以如果要有声音，就要考虑音画同步，而这一点是比较困难的。既然如此，那有没有办法可以不用手动控制视频和音频呢？当然有，就是我们的MediaPlayer了。我们只要利用MediaPlayer类的`setSurface()`方法，其实就和decoder的使用一模一样了。**一来免去了控制音画同步的麻烦，二还节省了MediaCodec的解码代码，真是一石二鸟!**

这里我们创建一个VideoRender类，并且实现GLSurfaceView的Render接口，来作为我们渲染器。

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
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
109
110</code></pre></td>
<td class="code"><pre><code>public class VideoRenderer extends SurfaceView implements GLSurfaceView.Renderer{
    
    private static final String SAMPLE = Environment.getExternalStorageDirectory() + &quot;/SAMPLE.mp4&quot;;

    private final String TAG = &quot;VideoRenderer&quot;;
    private PlayerThread mPlayer;
    private FullFrameRect mFullScreen;
    private SurfaceTexture mSurfaceTexture;
    private final float[] mSTMatrix = new float[16];
    private int mTextureId;
    GLSurfaceView mGLSurfaceView ;
    private Surface mSurface;
    private MediaPlayerWithSurface playerWithSurface;
    private SeekBar seekBar;
    private boolean isStart = false;

    protected int surfaceWidth, surfaceHeight;
    private int mCurrentFilter;
    private int mNewFilter;
    private boolean mIncomingSizeUpdated;

    public VideoRenderer(Context context,GLSurfaceView glSurfaceView, SeekBar seekBar) {
        super(context);
        this.mGLSurfaceView = glSurfaceView;
        Log.d(TAG, &quot;VideoRenderer: created&quot;);
        mCurrentFilter = -1;
        mNewFilter = MainActivity.FILTER_NONE;
        this.seekBar = seekBar;
    }


    public void updateFilter() {
        Texture2dProgram.ProgramType programType;
        float[] kernel = null;
        float colorAdj = 0.0f;

        Log.d(TAG, &quot;Updating filter to &quot; + mNewFilter);
        switch (mNewFilter) {
           //...
        }

        if (programType != mFullScreen.getProgram().getProgramType()) {
            mFullScreen.changeProgram(new Texture2dProgram(programType));
            // If we created a new program, we need to initialize the texture width/height.
            mIncomingSizeUpdated = true;
        }
        // Update the filter kernel (if any).
        if (kernel != null) {
            mFullScreen.getProgram().setKernel(kernel, colorAdj);
        }
        mCurrentFilter = mNewFilter;
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        Log.d(TAG, &quot;onSurfaceCreated: &quot;);
        mFullScreen = new FullFrameRect(
                new Texture2dProgram(Texture2dProgram.ProgramType.TEXTURE_EXT));

        mTextureId = mFullScreen.createTextureObject();

        mSurfaceTexture = new SurfaceTexture(mTextureId);

        mSurface = new Surface(mSurfaceTexture);

        mSurfaceTexture.setOnFrameAvailableListener(new SurfaceTexture.OnFrameAvailableListener() {
            @Override
            public void onFrameAvailable(SurfaceTexture surfaceTexture) {
                mGLSurfaceView.requestRender();
            }
        });
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        Log.d(TAG, &quot;onSurfaceChanged:  changed&quot;);
        // 用MediaCodec来解码视频 无声音
//        if(mPlayer == null) {
//            mPlayer = new PlayerThread(mSurface);
//            mPlayer.start();
//        }

        if( playerWithSurface == null ) {
            playerWithSurface = new MediaPlayerWithSurface(SAMPLE,mSurface,seekBar);
            playerWithSurface.playVideoToSurface();
        }

        GLES20.glViewport(0,0,width, height);
        surfaceWidth = width;
        surfaceHeight = height;

    }

    @Override
    public void onDrawFrame(GL10 gl) {
        mSurfaceTexture.updateTexImage();
        if (mCurrentFilter != mNewFilter) {
            updateFilter();
        }
        mSurfaceTexture.getTransformMatrix(mSTMatrix);
        mFullScreen.drawFrame(mTextureId, mSTMatrix);
    }

    public void changeFilterMode(int filter) {
        mNewFilter = filter;
    }

    //MediaCodec解码线程 输出到surface 具体可以参照MediaCodec文章 
    private class PlayerThread extends Thread {}
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

实现GLSurfaceView.Render接口需要实现三个方法，也是我们渲染的主要的三个方法:

`onSurfaceCreated()` 当Surface被创建的时候，我们生成了一个FullFrameRect的实例`mFullScreen`，并且利用它来拿到TextureId，用来生成了一个`mSurface`，就和我们说的思路是一样的。然后很关键的这里要给`** mSurfaceTexture`设置onFrameAvailable的监听器，并且回调`GLSurfaceView.requestRender()`方法\*\*，一开始我就是没写这段代码，死都搞不出来…

`onSurfaceChanged()` 在这里我们开启了我们的`MediaPlayer`，并且把生成`mSurface`和文件路径作为参数交给它，`mSurface`会作为播放的输出容器。

`onDrawFrame()` 我们会拿到相应的矩阵，交给`mSurfaceTexture`，并且让`mFullScreen`其实也就是它所持有的`Texture2dProgram`去做绘制的工作。如果当前的滤镜和所选择的滤镜不同就会去调用`updateFilter()`方法来实现实时的滤镜切换。

`updateFilter()` 更新滤镜。会根据新的filter来生成对应的`mTexture2dProgram` 并且利用上面我们提到的`setKernel()`来设置着色器参数，达到更改滤镜的效果。

## <a href="#项目源码" class="headerlink" title="项目源码"></a>项目源码

​由于篇幅原因，多数的类的代码都被我省略了。

​不过这个初步的小成果已经被我放到Github上了，大致就是实现了简单的视频实时滤镜的预览，并且再利用MediaPlayer和SeekBar做一些简单的播放控制。

​项目地址 <a href="https://github.com/lileizhenshuai/FilterPlayer" target="_blank" rel="noopener">FIlterPlayer</a>

​接下来会努力把本地化保存做好，然后更新这个代码哒。

- <a href="javascript:void(0)" class="js-tag article-tag-list-link color3">Android</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color1">MediaCodec</a>
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color2">OpenGL</a>

<span class="tooltip-item"> <a href="javascript:;" class="share-sns share-outer"><em></em></a> </span> <span class="tooltip-content"> </span>

<a href="javascript:;" class="weibo share-sns" data-type="weibo"><em></em></a> <a href="javascript:;" class="weixin share-sns wxFab" data-type="weixin"><em></em></a> <a href="javascript:;" class="qq share-sns" data-type="qq"><em></em></a> <a href="javascript:;" class="douban share-sns" data-type="douban"><em></em></a> <a href="javascript:;" class="qzone share-sns" data-type="qzone"><em></em></a> <a href="javascript:;" class="facebook share-sns" data-type="facebook"><em></em></a> <a href="javascript:;" class="twitter share-sns" data-type="twitter"><em></em></a> <a href="javascript:;" class="google share-sns" data-type="google"><em></em></a>

<a href="javascript:;" class="close js-modal-close"><em></em></a>

扫一扫，分享到微信

![微信分享二维码](http://s.jiathis.com/qrcode.php?url=https://tobiaslee.top/2017/04/23/FilterPlayer/)