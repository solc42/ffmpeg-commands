**testsrc2 — test pattern**
Built-in video source: running timer, color bars, flashing blocks. No input files needed.
```bash
ffmpeg -hide_banner -f lavfi -i testsrc2=duration=20:size=1280x720:rate=30 -c:v libx264 -pix_fmt yuv420p output.mp4
```

---

**single keyframe — video with one I-frame only**
Only 1 keyframe (IDR) at the start, all others are P-frames. `keyint=infinite` prevents additional keyframes.
```bash
ffmpeg -hide_banner -f lavfi -i testsrc2=duration=5:size=1280x720:rate=30 -c:v libx264 -x264-params "keyint=infinite" -pix_fmt yuv420p single_keyframe.mp4
```

---

**show frame types**
Prints pict_type (I/P/B), key_frame flag, pts_time per frame.
```bash
ffprobe -hide_banner -show_frames -select_streams v -show_entries frame=pict_type,key_frame,pts_time -print_format csv single_keyframe.mp4
```

---

**show NAL unit types (trace_headers)**
Sequential list of NAL units with index and human-readable type name.
```bash
ffmpeg -hide_banner -i single_keyframe.mp4 -c copy -bsf:v trace_headers -f null - 2>&1 \
  | grep "nal_unit_type" \
  | awk -F'= ' '{ \
      t=$2+0; n="?"; \
      if(t==1) n="non-IDR"; \
      if(t==5) n="IDR"; \
      if(t==6) n="SEI"; \
      if(t==7) n="SPS"; \
      if(t==8) n="PPS"; \
      printf "%4d: type=%d (%s)\n", NR, t, n \
    }'
```

---

**video with B-frames (I/P/B mix)**
Generates video with B-frames enabled. Use `show frame types` command to see I/P/B distribution.
```bash
ffmpeg -hide_banner -f lavfi -i testsrc2=duration=5:size=1280x720:rate=30 -c:v libx264 -bf 3 -pix_fmt yuv420p bframes.mp4
```

---

**color switch — scenecut keyframe detection**
3s red then 3s blue. x264 scenecut detection automatically inserts a new keyframe at the color change point (3.000s).
```bash
ffmpeg -hide_banner \
  -f lavfi -i "color=c=red:duration=3:size=1280x720:rate=30,format=yuv420p[v0]; \
               color=c=blue:duration=3:size=1280x720:rate=30,format=yuv420p[v1]; \
               [v0][v1]concat=n=2:v=1:a=0" \
  -c:v libx264 -pix_fmt yuv420p color_switch.mp4
```

---

**transcode mp4 to HLS**
Re-encodes video and splits into HLS segments (.m3u8 playlist + .ts chunks).
```bash
ffmpeg -hide_banner -i input.mp4 -c:v libx264 -c:a aac -hls_time 4 -hls_playlist_type vod output.m3u8
```

---

**transmux mp4 to HLS (no re-encoding)**
Copies streams as-is, only changes container. Much faster, no quality loss.
```bash
ffmpeg -hide_banner -i input.mp4 -c copy -hls_time 4 -hls_playlist_type vod output.m3u8
```

---

**show packets (keyframe flags, pts, size)**
One line per packet: pts_time, K=keyframe / _=non-keyframe, size in bytes.
Note: packet flags only support K (keyframe) or _ (non-keyframe) — no P/B distinction. Use `show frame types` command for I/P/B pict_type.
```bash
ffprobe -hide_banner -show_packets -select_streams v -show_entries packet=pts_time,flags,size -of csv input.mp4
```
