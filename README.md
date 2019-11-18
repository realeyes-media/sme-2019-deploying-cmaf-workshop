# SME 2019 Deploying CMAF: Why, When & How

## Slides

SME
https://docs.google.com/presentation/d/1YGAn-DEDU81NP28LlYr1y06ER9siYUXoMKiHf1MtriY/edit#slide=id.p

SMW
https://docs.google.com/presentation/d/1862tti3iKDXk9TlDTB9S25S7kGFFk3WwfUB9uZa2rM4/edit#slide=id.p

## Library Setup

1. Install [FFMPEG](https://ffmpeg.org/download.html)
NOTE: For macOS: `brew install ffmpeg`

2. Install [Bento4](https://www.bento4.com/downloads/)

3. Install Shaka [Packager](https://github.com/google/shaka-packager/releases)

## Initial MP4

For this workshop we will be using [Big Buck Bunny](https://peach.blender.org/download/)

Download the file

```bash
curl --url http://mirror.cessen.com/blender.org/peach/trailer/trailer_1080p.mov --output ./files/bbb_trailer_1080p.mov
```

Convert .mov to .mp4

```bash
ffmpeg -i ./files/bbb_trailer_1080p.mov -vcodec h264 -acodec aac ./files/bbb_trailer_1080p.mp4
```

Convert to CMAF w/ 2 second durations

Video

```bash
ffmpeg -i ./files/bbb_trailer_1080p.mp4 \
    -movflags frag_keyframe+empty_moov+default_base_moof+faststart  -bf 2 -g 50 -sc_threshold 0 -an -strict experimental -profile:v baseline -b:v 2048k -f mp4 ./files/bbb_trailer_1080p_video.fmp4
```

Audio

```bash
ffmpeg -i ./files/bbb_trailer_1080p.mp4 \
    -movflags frag_keyframe+empty_moov+default_base_moof+faststart -vn -strict experimental -profile:v baseline -c:a copy -frag_duration 2000000 -f mp4 ./files/bbb_trailer_1080p_audio.fmp4
```

## Packaging

### Prep

Create necessary directories

```bash
mkdir output
```

### Bento4

Package HLS and DASH manifests.

```bash
./tools/Bento4/bin/mp4dash -o ./output/bento --hls --use-segment-timeline ./files/bbb_trailer_1080p_*.fmp4
```

### Shaka

Package HLS and DASH manifests

```bash
./tools/Shaka/packager-osx \
  'in=./files/bbb_trailer_1080p_audio.fmp4,stream=audio,init_segment=output/shaka/audio/init.mp4,segment_template=output/shaka/audio/$Number$.m4s,playlist_name=audio/main.m3u8,hls_group_id=audio,hls_name=English' \
  'in=./files/bbb_trailer_1080p_video.fmp4,stream=video,init_segment=output/shaka/video_1080p/init.mp4,segment_template=output/shaka/video_1080p/$Number$.m4s,playlist_name=video_1080p/main.m3u8,iframe_playlist_name=video_1080p/iframe.m3u8' \
  --segment_duration 2 \
  --hls_master_playlist_output ./output/shaka/master.m3u8 \
  --mpd_output ./output/shaka/master.mpd
```

## Verify

Create SSL certs

```bash
openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout key.pem -out cert.pem
```

NOTE: use 127.0.0.1 as Common Name

Start Server

```bash
http-server -p 4200 -c-1 --cors -S -C cert.pem
```

### Verify Bento4

[HLS](https://hls-js.netlify.com/demo/?src=https%3A%2F%2Flocalhost%3A4200%2Foutput%2Fbento%2Fmaster.m3u8&demoConfig=eyJlbmFibGVTdHJlYW1pbmciOnRydWUsImF1dG9SZWNvdmVyRXJyb3IiOnRydWUsImVuYWJsZVdvcmtlciI6dHJ1ZSwiZHVtcGZNUDQiOmZhbHNlLCJsZXZlbENhcHBpbmciOi0xLCJsaW1pdE1ldHJpY3MiOi0xLCJ3aWRldmluZUxpY2Vuc2VVcmwiOiIifQ==)

[DASH](http://reference.dashif.org/dash.js/nightly/samples/dash-if-reference-player/index.html)

Use `https://localhost:4200/output/bento/stream.mpd` as the source

### Verify Shaka

[HLS](https://hls-js.netlify.com/demo/?src=https%3A%2F%2Flocalhost%3A4200%2Foutput%2Fshaka%2Fmaster.m3u8&demoConfig=eyJlbmFibGVTdHJlYW1pbmciOnRydWUsImF1dG9SZWNvdmVyRXJyb3IiOnRydWUsImVuYWJsZVdvcmtlciI6dHJ1ZSwiZHVtcGZNUDQiOmZhbHNlLCJsZXZlbENhcHBpbmciOi0xLCJsaW1pdE1ldHJpY3MiOi0xLCJ3aWRldmluZUxpY2Vuc2VVcmwiOiIifQ==)

[DASH](http://reference.dashif.org/dash.js/nightly/samples/dash-if-reference-player/index.html)

Use `https://localhost:4200/output/shaka/master.mpd` as the source



//////////////////////////////////////////////

#WINDOW

//MOV > MP4
ffmpeg -i bbb_trailer_1080p.mov -vcodec h264 -acodec aac bbb_trailer_1080p.mp4

//VIDEO > FMP4
ffmpeg -i bbb_trailer_1080p.mp4 -movflags frag_keyframe+empty_moov+default_base_moof+faststart  -bf 2 -g 50 -sc_threshold 0 -an -strict experimental -profile:v baseline -b:v 2048k -f mp4 bbb_trailer_1080p_video.fmp4

//AUDIO
ffmpeg -i bbb_trailer_1080p.mp4 -movflags frag_keyframe+empty_moov+default_base_moof+faststart -vn -strict experimental -profile:v baseline -c:a copy -frag_duration 2000000 -f mp4 bbb_trailer_1080p_audio.fmp4

//BENTO - Package hls/dash
mp4dash -o /output/bento --hls --use-segment-timeline bbb_trailer_1080p_*.fmp4

//SHAKA - Package hls/dash
packager-win in=bbb_trailer_1080p_audio.fmp4,stream=audio,init_segment=output/shaka/audio/init.mp4,segment_template=output/shaka/audio/$Number$.m4s,playlist_name=audio/main.m3u8,hls_group_id=audio,hls_name=English in=bbb_trailer_1080p_video.fmp4,stream=video,init_segment=output/shaka/video_1080p/init.mp4,segment_template=output/shaka/video_1080p/$Number$.m4s,playlist_name=video_1080p/main.m3u8,iframe_playlist_name=video_1080p/iframe.m3u8 --segment_duration 2 --hls_master_playlist_output output/shaka/master.m3u8 --mpd_output output/shaka/master.mpd

