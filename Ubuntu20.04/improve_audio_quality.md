# Audio Quality Problem in Ubuntu

Even with good audio devices the audio quality in ubuntu becomes worse compared to windows. This is because of the bad default audio configuration settings (pulseaudio setting - checkout [Guide to Linux Audio](https://linuxhint.com/guide_linux_audio/)). Here is an attempt to change the audio configuration settings to better suit modern laptops.

## Short basic fixes
With the following three basic fixes your audio quality will improve drastically and you may not need to go further in the article if you need short fixes. The pulseaudio daemon read its configuration from `/etc/pulse/daemon.conf` which can be overrided by specifying configurations in `~/.config/pulse/daemon.conf`. (more about this can be read using `man pulse-daemon.conf`).

1. Obtain the output of `pacmd list-sinks | grep sample` which in my comes out to be `sample spec: s32le 2ch 48000Hz`. It means my sound card supports sample format: s32le, sample channels: 2 and sample rate: 48 kHz.

2. The default conf present are `default-sample-format = s16le` and `default-sample-rate = 44100`. Override them by entering `default-sample-format = s32le` and `default-sample-rate = 48000` in `~/.config/pulse/daemon.conf`.

3. The default resampling method is `speex-float-1` which quite old and is required for old machines with weak processors. Modern computers can use better `soxr-vhq` which maybe more cpu intensive but will not affect modern machines. So oveeride default setting by entering `resample-method = soxr-vhq`.

For more info checkout [Enable High Quality Audio on Linux](https://medium.com/@gamunu/enable-high-quality-audio-on-linux-6f16f3fe7e1f) and [PulseAudio Reviews](https://linuxreviews.org/PulseAudio).

## Setting fragment details

#### 1. Calculate the bits processing rate
Since default sample rate is 48 kHz and bit width of sample is 32 bits (from s**32**le), bits processed per second = 48000*32 = 1536000 bps.

#### 2. Obtain the fragment and buffer size
Using `pactl list sinks | grep device.buffering` you will obtain fragment and buffer size like in my case I obtained the following output:
```
device.buffering.buffer_size = "768000"
device.buffering.fragment_size = "384000"
```

#### 3. Calculate fragment size in msec and number of fragments
fragment size in msec = 384000/1536000 = 0.25 s = 250 ms
buffer size in msec = 768000/1536000 = 0.5 s = 500 ms
number of fragments = buffer size / fragment size = 2

Hence override default value by writing `default-fragment-size-msec = 250` and `default-fragments = 2`.

**References:** 
1. https://bbs.archlinux.org/viewtopic.php?id=223446
2. https://forums.linuxmint.com/viewtopic.php?t=44862

## Increasing priority of pulseaudio

Simplest method of increasing performance of a process can be by increasing the time allocated to it by cpu, meaning increasing its priority level among processes.

1. You can decrease the [nice-level](https://en.wikipedia.org/wiki/Nice_(Unix)#:~:text=nice%20is%20a%20program%20found,operating%20systems%20such%20as%20Linux.&text=nice%20is%20used%20to%20invoke,19%20is%20the%20lowest%20priority.) by changing default value of `nice-level` from -11 (default) to -15 (decreasing is not recommended as it may cause expensive consumption of resources).

2. You can increase the realtime priority by changing value of `realtime-priority` from 5 (default) to 9.

## Achieving low latency on audio output

We have to configure alsa for best audio output. The default configuration is specified in the file `/usr/share/alsa/pulse-alsa.conf`. We have to change the code from:
```
pcm.!default {
    type pulse
    hint {
        show on
        description "Playback/recording through the PulseAudio sound server"
    }
}
```

to the following code for slave.pcm 'hw' plugin to directly communicate with alsa kernel driver.
```
pcm.!default {
    type plug
    slave.pcm hw
}
```