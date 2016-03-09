---
layout: post
title: Android MediaCodec 硬编码H264格式
category: 学习
tags: KIDLOSERME
keywords: MediaCodec Android硬编码
description: 
---

最近在研究EasyDarwin的Push库EasyPuhser，EasyPuhser可以推送H264视频到Easydarwin服务器，终端可以通过rtsp协议访问该实时流，达到手机直播的功能，延迟基本在2秒以内。
EasyDarwinQQ群：496258327
本文主要记录一下最近研究的关于Android手机如何获取实时画面，并将数据编码为H264的格式的视频流，编码使用的是Android自带的MediaCodec，也就是硬解。
本demo的下载地址：[MediaCodecDemo](https://github.com/kidloserme/MediaCodecDemo)

MediaCodec是Android在4.1中加入的新的API，目前也有很多文章介绍MediaCodec的用法，但是很多时候很多手机都失败，主要问题出现在调用dequeueOutputBuffer的时候总是返回-1，让你以为No buffer available !这里介绍一个开源项目[libstreaming](https://github.com/fyhertz/libstreaming),我们借助此项目中封装的一个工具类[EncoderDebugger](https://github.com/fyhertz/libstreaming/tree/master/src/net/majorkernelpanic/streaming/hw)，来初始化MediaCodec会很好的解决此问题，目前为止测试了几个手机都可以成功，包括小米华为Moto。
看一下怎么使用的

``` java
EncoderDebugger debugger = EncoderDebugger.debug(getApplicationContext(), width, height);
MediaCodec mMediaCodec = MediaCodec.createByCodecName(debugger.getEncoderName());
```

嗯，就这样。当然了，后面还是要根据需要对mMediaCodec设置其他参数的，看一下本demo中设置参数的过程吧

```java
private void initMediaCodec() {
    int dgree = getDgree();
    framerate = 15;
    bitrate = 2 * width * height * framerate / 20;
    EncoderDebugger debugger = EncoderDebugger.debug(getApplicationContext(), width, height);
    mConvertor = debugger.getNV21Convertor();
    try {
        mMediaCodec = MediaCodec.createByCodecName(debugger.getEncoderName());
        MediaFormat mediaFormat;
        if (dgree == 0) {
            //dree==0的时候，需要将画面旋转90度，所以这里编码的时候需要将宽和高颠倒，
            //否则编码后的会面会出现四重画面并且花屏
            mediaFormat = MediaFormat.createVideoFormat("video/avc", height, width);
        } else {
            mediaFormat = MediaFormat.createVideoFormat("video/avc", width, height);
        }
        mediaFormat.setInteger(MediaFormat.KEY_BIT_RATE, bitrate);
        mediaFormat.setInteger(MediaFormat.KEY_FRAME_RATE, framerate);
        mediaFormat.setInteger(MediaFormat.KEY_COLOR_FORMAT,
                debugger.getEncoderColorFormat());
        mediaFormat.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, 1);
        mMediaCodec.configure(mediaFormat, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);
        mMediaCodec.start();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

编码之前先看一下要编码的数据怎么获取吧，这个当然是来自Camera。
首先是创建SurfaceView用于预览视频画面，并设置回调，来监控生命周期。

```java
surfaceView = (SurfaceView) findViewById(R.id.sv_surfaceview);
surfaceView.getHolder().addCallback(this);
surfaceView.getHolder().setFixedSize(getResources().getDisplayMetrics().widthPixels,
getResources().getDisplayMetrics().heightPixels);
```

然后是创建Camera的方法：

``` java
private boolean ctreateCamera(SurfaceHolder surfaceHolder) {
    try {
        //mCameraId=Camera.CameraInfo.CAMERA_FACING_BACK
        mCamera = Camera.open(mCameraId);
        Camera.Parameters parameters = mCamera.getParameters();
        Camera.CameraInfo camInfo = new Camera.CameraInfo();
        Camera.getCameraInfo(mCameraId, camInfo);
        int cameraRotationOffset = camInfo.orientation;
        //设置预览格式NV21，他属于YUV420SP
        parameters.setPreviewFormat(ImageFormat.NV21);
        parameters.setPreviewSize(width, height);
        mCamera.setParameters(parameters);
        mCamera.autoFocus(null);
        //计算preview画面需要旋转的角度。目前木有做横竖屏切换的时候无缝旋转画面，后面再搞。
        int  displayRotation = (cameraRotationOffset - getDgree() + 360) % 360;
        mCamera.setDisplayOrientation(displayRotation);
        mCamera.setPreviewDisplay(surfaceHolder);
        return true;
    } catch (Exception e) {
        destroyCamera();
        e.printStackTrace();
        return false;
    }
}

private int getDgree() {
    int rotation = getWindowManager().getDefaultDisplay().getRotation();
    int degrees = 0;
    switch (rotation) {
        case Surface.ROTATION_0:
            degrees = 0;
            break; // Natural orientation
        case Surface.ROTATION_90:
            degrees = 90;
            break; // Landscape left
        case Surface.ROTATION_180:
            degrees = 180;
            break;// Upside down
        case Surface.ROTATION_270:
            degrees = 270;
            break;// Landscape right
    }
    return degrees;
}
```


摄像头创建完毕，就是开启预览

``` java
/**
 * 开启预览
 */
public synchronized void startPreview() {
    if (mCamera != null && !started) {
        mCamera.startPreview();
        int previewFormat = mCamera.getParameters().getPreviewFormat();
        Camera.Size previewSize = mCamera.getParameters().getPreviewSize();
        int size = previewSize.width * previewSize.height
                * ImageFormat.getBitsPerPixel(previewFormat)
                / 8;
        mCamera.addCallbackBuffer(new byte[size]);
        mCamera.setPreviewCallbackWithBuffer(previewCallback);
        started = true;
        btnSwitch.setText("停止");
    }
}
```

上面就是设置了预览回调的方式，回调中将预览画面一帧一帧的返回给我们，给我们的数据就是NV21格式的,根据需要决定是否需要对数据进行旋转，旋转之后，就是转换，将NV21数据转为YUV420P格式的数据，然后就可以编码为H264数据了。

```java
Camera.PreviewCallback previewCallback = new Camera.PreviewCallback() {
    //mSpsPps用来存储sps pps数据，后面遇到关键帧（I帧），必须将spspps数据加到I帧前面
    byte[] mSpsPps = new byte[0];
    @Override
    public void onPreviewFrame(byte[] data, Camera camera) {
        if (data == null) {
            return;
        }
        ByteBuffer[] inputBuffers = mMediaCodec.getInputBuffers();
        ByteBuffer[] outputBuffers = mMediaCodec.getOutputBuffers();
        byte[] dst = new byte[data.length];
        Camera.Size previewSize = mCamera.getParameters().getPreviewSize();
        if (getDgree() == 0) {
            //手机竖屏的时候要将获取的数据顺时针旋转90度，否则画面不是正着的，而是逆时针90度
            dst = Util.rotateNV21Degree90(data, previewSize.width, previewSize.height);
        } else {
            dst = data;
        }
        try {
            int bufferIndex = mMediaCodec.dequeueInputBuffer(5000000);
            if (bufferIndex >= 0) {
                inputBuffers[bufferIndex].clear();
                //将YUV420SP数据转换成YUV420P的格式，并将结果存入inputBuffers[bufferIndex]
                mConvertor.convert(dst, inputBuffers[bufferIndex]);
                mMediaCodec.queueInputBuffer(bufferIndex, 0,
                        inputBuffers[bufferIndex].position(),
                        System.nanoTime() / 1000, 0);
                MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();
                int outputBufferIndex = mMediaCodec.dequeueOutputBuffer(bufferInfo, 0);
                while (outputBufferIndex >= 0) {
                    ByteBuffer outputBuffer = outputBuffers[outputBufferIndex];
                    byte[] outData = new byte[bufferInfo.size];
                    //从buff中读取数据到outData中
                    outputBuffer.get(outData);
                    //记录pps和sps，pps和sps数据开头是0x00 0x00 0x00 0x01 0x67，
                    // 0x67对应十进制103
                    if (outData[0] == 0 && outData[1] == 0 && outData[2] == 0 
                            && outData[3] == 1 && outData[4] == 103) {
                        mSpsPps = outData;
                    } else if (outData[0] == 0 && outData[1] == 0 && outData[2] == 0 
                            && outData[3] == 1 && outData[4] == 101) {
                        //关键帧开始规则是0x00 0x00 0x00 0x01 0x65，0x65对应十进制101
                        //在关键帧前面加上pps和sps数据
                        byte[] iframeData = new byte[mSpsPps.length + outData.length];
                        System.arraycopy(mSpsPps, 0, iframeData, 0, mSpsPps.length);
                        System.arraycopy(outData, 0, iframeData, mSpsPps.length, outData.length);
                        outData = iframeData;
                    }
                    //至此，这一帧的数据已经经过MediaCodec编码完毕，这个outData就是我们需要的数据了，
                    //因为EasyDarwin可以自动将H264打包为RTP，
                    //所以EasyPusher只需要负责将outData推给EasyDarwin就OK了
                    //保存H264数据到本地文件easy.h264
                    Util.save(outData, 0, outData.length, path, true);
                    mMediaCodec.releaseOutputBuffer(outputBufferIndex, false);
                    outputBufferIndex = mMediaCodec.dequeueOutputBuffer(bufferInfo, 0);
                }
            } else {
                Log.e("easypusher", "No buffer available !");
            }
        } catch (Exception e) {
            StringWriter sw = new StringWriter();
            PrintWriter pw = new PrintWriter(sw);
            e.printStackTrace(pw);
            String stack = sw.toString();
            Log.e("save_log", stack);
            e.printStackTrace();
        } finally {
            mCamera.addCallbackBuffer(dst);
        }
    }
};
```

保存之后的文件easy.h264我用VLC播放器打开，截屏如下：

![easy264截屏](http://img.blog.csdn.net/20160309132221153)

OK，基本上完毕了，该注意的地方都写在代码中了

需要Demo的请到这里[https://github.com/kidloserme/MediaCodecDemo](https://github.com/kidloserme/MediaCodecDemo)




