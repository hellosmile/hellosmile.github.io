---
layout: post
title:  "ExoPlayer RTMP/FLV 支持 H265 扩展"
date:   2022-03-19 23:22:36 +0530
---

### 背景
HLS 和 TS 都已经支持了 H265，但是国内大部分 CDN 更多的是采用 RTMP/HTTP-FLV；因为 HLS 的延迟比较高，所以国内大部分直播业务采用的都是 RTMP/HTTP-FLV。

此外 Adobe 在制定 RTMP/HTTP-FLV，还没有出现 H265，因此 RTMP/HTTP-FLV 不支持 H265 ，而且也没有计划对 RTMP/HTTP-FLV 支持 H265 作出标准升级，因此 FFmpeg、ExoPlayer 没有让 FLV 支持 H265。

基于上述背景，需要定制化 ExoPlayer，达到 ExoPlayer RTMP/FLV 支持 H265 扩展。


### ExoPlayer RTMP/FLV 支持 H265 扩展
修改 com.google.android.exoplayer2.extractor.flv.VideoTagPayloadReader，见“Ext：”开头的注释：

```java

/*
 * Copyright (C) 2016 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package com.google.android.exoplayer2.extractor.flv;

import com.google.android.exoplayer2.C;
import com.google.android.exoplayer2.Format;
import com.google.android.exoplayer2.ParserException;
import com.google.android.exoplayer2.extractor.TrackOutput;
import com.google.android.exoplayer2.util.MimeTypes;
import com.google.android.exoplayer2.util.NalUnitUtil;
import com.google.android.exoplayer2.util.ParsableByteArray;
import com.google.android.exoplayer2.video.AvcConfig;
import com.google.android.exoplayer2.video.HevcConfig;

/** Parses video tags from an FLV stream and extracts H.264 nal units. */
/* package */ final class VideoTagPayloadReader extends TagPayloadReader {

  // Video codec.
  private static final int VIDEO_CODEC_AVC = 7;
  // Ext：扩展视频编码标识 HEVC
  private static final int VIDEO_CODEC_HEVC = 12;

  // Frame types.
  private static final int VIDEO_FRAME_KEYFRAME = 1;
  private static final int VIDEO_FRAME_VIDEO_INFO = 5;

  // Packet types.
  private static final int AVC_PACKET_TYPE_SEQUENCE_HEADER = 0;
  private static final int AVC_PACKET_TYPE_AVC_NALU = 1;

  // Temporary arrays.
  private final ParsableByteArray nalStartCode;
  private final ParsableByteArray nalLength;
  private int nalUnitLengthFieldLength;

  // State variables.
  private boolean hasOutputFormat;
  private boolean hasOutputKeyframe;
  private int frameType;
  // Ext：记录视频编码标识
  private int lastVideoCodecId;

  /** @param output A {@link TrackOutput} to which samples should be written. */
  public VideoTagPayloadReader(TrackOutput output) {
    super(output);
    nalStartCode = new ParsableByteArray(NalUnitUtil.NAL_START_CODE);
    nalLength = new ParsableByteArray(4);
  }

  @Override
  public void seek() {
    hasOutputKeyframe = false;
  }

  @Override
  protected boolean parseHeader(ParsableByteArray data) throws UnsupportedFormatException {
    int header = data.readUnsignedByte();
    int frameType = (header >> 4) & 0x0F;
    int videoCodec = (header & 0x0F);

    // Ext：记录视频编码标识
    if (videoCodec == VIDEO_CODEC_AVC || videoCodec == VIDEO_CODEC_HEVC) {
      lastVideoCodecId = videoCodec;
    }
    // Support just H.264 encoded content.
    // Ext：支持 H.265 编码内容
    if (videoCodec != VIDEO_CODEC_AVC && videoCodec != VIDEO_CODEC_HEVC) {
      throw new UnsupportedFormatException("Video format not supported: " + videoCodec);
    }
    this.frameType = frameType;
    return (frameType != VIDEO_FRAME_VIDEO_INFO);
  }

  @Override
  protected boolean parsePayload(ParsableByteArray data, long timeUs) throws ParserException {
    int packetType = data.readUnsignedByte();
    int compositionTimeMs = data.readInt24();

    timeUs += compositionTimeMs * 1000L;
    // Parse avc sequence header in case this was not done before.
    // Ext：支持 H.265 编码内容
    if (packetType == AVC_PACKET_TYPE_SEQUENCE_HEADER && !hasOutputFormat) {
      ParsableByteArray videoSequence = new ParsableByteArray(new byte[data.bytesLeft()]);
      data.readBytes(videoSequence.getData(), 0, data.bytesLeft());
      if (lastVideoCodecId == VIDEO_CODEC_AVC) {
        AvcConfig avcConfig = AvcConfig.parse(videoSequence);
        nalUnitLengthFieldLength = avcConfig.nalUnitLengthFieldLength;
        // Construct and output the format.
        Format format =
            new Format.Builder()
                .setSampleMimeType(MimeTypes.VIDEO_H264)
                .setCodecs(avcConfig.codecs)
                .setWidth(avcConfig.width)
                .setHeight(avcConfig.height)
                .setPixelWidthHeightRatio(avcConfig.pixelWidthHeightRatio)
                .setInitializationData(avcConfig.initializationData)
                .build();
        output.format(format);
      } else if (lastVideoCodecId == VIDEO_CODEC_HEVC) {
        HevcConfig hevcConfig = HevcConfig.parse(videoSequence);
        nalUnitLengthFieldLength = hevcConfig.nalUnitLengthFieldLength;
        // Construct and output the format.
        Format format =
            new Format.Builder()
                .setSampleMimeType(MimeTypes.VIDEO_H265)
                .setCodecs(hevcConfig.codecs)
                .setWidth(hevcConfig.width)
                .setHeight(hevcConfig.height)
                .setPixelWidthHeightRatio(hevcConfig.pixelWidthHeightRatio)
                .setInitializationData(hevcConfig.initializationData)
                .build();
        output.format(format);
      }
      hasOutputFormat = true;
      return false;
    } else if (packetType == AVC_PACKET_TYPE_AVC_NALU && hasOutputFormat) {
      boolean isKeyframe = frameType == VIDEO_FRAME_KEYFRAME;
      if (!hasOutputKeyframe && !isKeyframe) {
        return false;
      }
      // TODO: Deduplicate with Mp4Extractor.
      // Zero the top three bytes of the array that we'll use to decode nal unit lengths, in case
      // they're only 1 or 2 bytes long.
      byte[] nalLengthData = nalLength.getData();
      nalLengthData[0] = 0;
      nalLengthData[1] = 0;
      nalLengthData[2] = 0;
      int nalUnitLengthFieldLengthDiff = 4 - nalUnitLengthFieldLength;
      // NAL units are length delimited, but the decoder requires start code delimited units.
      // Loop until we've written the sample to the track output, replacing length delimiters with
      // start codes as we encounter them.
      int bytesWritten = 0;
      int bytesToWrite;
      while (data.bytesLeft() > 0) {
        // Read the NAL length so that we know where we find the next one.
        data.readBytes(nalLength.getData(), nalUnitLengthFieldLengthDiff, nalUnitLengthFieldLength);
        nalLength.setPosition(0);
        bytesToWrite = nalLength.readUnsignedIntToInt();

        // Write a start code for the current NAL unit.
        nalStartCode.setPosition(0);
        output.sampleData(nalStartCode, 4);
        bytesWritten += 4;

        // Write the payload of the NAL unit.
        output.sampleData(data, bytesToWrite);
        bytesWritten += bytesToWrite;
      }
      output.sampleMetadata(
          timeUs, isKeyframe ? C.BUFFER_FLAG_KEY_FRAME : 0, bytesWritten, 0, null);
      hasOutputKeyframe = true;
      return true;
    } else {
      return false;
    }
  }
}

```


### 针对FFmpeg Trac：Support H.265 over Adobe HTTP-FLV or RTMP 的思考
支持 FFmpeg 和 ExoPlayer 对待 RTMP/HTTP-FLV 拒绝支持 H.265 的做法，基于共同遵守的协议/标准，实现协议/标准定义的功能，正是对协议/标准的敬畏和维护。


### 参考资料
* [FFmpeg Trac：Support H.265 over Adobe HTTP-FLV or RTMP][FFmpeg Trac：Support H.265 over Adobe HTTP-FLV or RTMP]

[FFmpeg Trac：Support H.265 over Adobe HTTP-FLV or RTMP]: https://trac.ffmpeg.org/ticket/6389