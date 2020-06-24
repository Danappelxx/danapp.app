---
title: "Headless Setup of Spotify and Airplay on a Raspberry Pi"
description: "Step-by-step instructions to set up home audio with Spotify and Airplay on a Raspberry Pi"
slug: "home-audio"
date: "Wed, 24 Jun 2020 16:11:07 -0700"
---

I was gifted a Raspberry Pi 3B+ and decided to use it to replace the creepy Alexa Echo Dot I had connected to my speakers. This turned out to be pretty trivial (no coding required), but documentation along the way was fairly scattered so I figured I'd compile my notes here anyway.

<!--more-->

## Materials

This should work with any model of Raspberry Pi. If you have a monitor and keyboard on hand your life will probably be easier, but in the spirit of sitting in my chair all day this whole process is headless. I used a 32gb SD card but you can *probably* work with 8gb (especially if you cross-compile librespot).

The key components are [Alpine Linux](https://github.com/knoopx/alpine-raspberry-pi), [librespot](https://github.com/librespot-org/librespot) for Spotify, and [shairport-sync](https://github.com/mikebrady/shairport-sync) for Airplay.

## Build Alpine Linux for Raspberry Pi

You can probably get away with running Rasbian OS, but since I don't need any extra features I decided to go with Alpine Linux. Alpine is really lightweight, runs on Raspberry Pi just fine, and has almost all of the packages we need.

At first I attempted following the [official Alpine documentation](https://wiki.alpinelinux.org/wiki/Raspberry_Pi_-_Headless_Installation). That turned out to be a hassle so I ended up using [this fine project](https://github.com/knoopx/alpine-raspberry-pi), which builds an image with ssh and persistent storage that you can immediately flash onto an SD card. You can download a prebuilt image from the releases tab on the repository, or build it in a Docker container.

```bash
~> git clone https://github.com/knoopx/alpine-raspberry-pi
~> cd alpine-raspberry-pi
~> ./docker-runner
```

Pull up something like [balenaEtcher](https://www.balena.io/etcher/) and flash the image from `./alpine-raspberry-pi/dist/alpine-rpi-edge-armhf.img.gz` onto the SD card. Re-insert the SD card and modify the `wpa_suplicant.conf` file in the root directory with your Wifi credentials.

That's it! We're ready for first boot.

## First boot

Insert the SD card into the Raspberry Pi and boot it up. Log into your Wifi router (usually `192.168.1.1`) and look for a new lease in order to get it's IP. The default password for both `root` and `pi` is `raspberry`. Once you ssh in I recommend immediately changing this to something less obvious via `passwd`.

## Setting up audio

Follow the instructions [here](https://wiki.alpinelinux.org/wiki/Sound_Setup). To summarize,

```bash
~> apk add alsa-utils alsa-utils-doc alsa-lib alsaconf
~> addgroup pi audio
~> addgroup root audio
~> rc-service alsa start
~> rc-update add alsa
```

With that out of the way, you should have audio fully set up and running. This actually was not enough for me, and is the main reason I'm writing this post. To save you the effort - if running `lsmod | grep snd` doesn't output anything, then add `snd_bcm2835` to `/etc/modules` if it's not there already. Also, make sure `/boot/config.txt` and/or `/boot/usercfg.txt` have the line `dtparam=audio=on`. Reboot.

You can verify that your sound card is detected by running `alsamixer`.

```bash
~> sudo alsamixer # for me, this doesn't work without sudo ¯\_(ツ)_/¯
```

If you see `Card: bcm2835 Headphones` in the upper left corner, you have audio!

## Spotify

[librespot](https://github.com/librespot-org/librespot) is a fantastic open source Spotify Connect receiver written in Rust. I've used it for previous projects and it's been running pretty well, especially for a reverse engineered project. The only issue I've encountered is mediocre shuffling for large playlists. I'll submit a pull request when I figure out the cause.

Unfortunately, you're probably going to need to build from source. This took me a whole goddamn hour. I attempted to cross compile while I was waiting but ended up just hanging out with my good friends "Hacker News" and "Reddit".

```bash
~> apk add git alsa-lib-dev rust cargo
~> git clone https://github.com/librespot-org/librespot
~> cd librespot
~> cargo build --release
~> cp target/release/librespot /usr/local/bin/
```

Give it a run!

```bash
~> sudo librespot -n "alpine" # like alsamixer, this didn't work without sudo
```

Plug the Pi into your speakers and try connecting from your Spotify on your phone or laptop. If the audio quality is terrible, make sure you built librespot with the release flag. If you're hearing audio clipping, run `alsamixer` and turn it down to ~80.

### Running Spotify on boot

I ended up writing an OpenRC file with hardcoded flags. You can modify it for your own purposes if you'd like. librespot options are listed [here](https://github.com/librespot-org/librespot/wiki/Options).

**/etc/init.d/spotify**

```bash
#!/sbin/openrc-run

command="/usr/local/bin/librespot"
command_args="--name \"Dan's Room\" --disable-audio-cache --device-type speaker"
command_background=true
pidfile="/run/${RC_SVCNAME}.pid"

depend() {
	need net alsa
}
```

```bash
~> rc-service spotify start
~> rc-update add spotify default
```

Reboot to make sure that the Spotify starts up on its own. I followed the documentation [here](https://wiki.alpinelinux.org/wiki/Writing_Init_Scripts) and [here](https://github.com/OpenRC/openrc/blob/master/service-script-guide.md) to write this init script.

## Airplay

### Landscape

There have been a lot of attempts to reverse engineer Airplay. A few have been successful, though I've yet to see one which supports multi-room audio over Airplay 2. [This project](https://github.com/serezhka/java-airplay-server) in particular is really close, and supports all the video features of Airplay 2, but does not do any audio. If I find time I intend to port the core to Rust and implement the missing audio features following [this analysis](https://www.programmersought.com/article/2084789418/#).

Here's a brief report of the research I ended up doing.

- [Gist listing a few interesting projects](https://gist.github.com/watson/50e46a6085ffc805d326)
- [Old overview of the Airplay universe](https://github.com/openairplay/open-airplay)
- [Old unofficial Airplay protocol specification](https://nto.github.io/AirPlay.html)
- [RPiPlay](https://github.com/FD-/RPiPlay)
    - In retrospect, I probably should have used this, but totally forgot about it until now
- [shairplay](https://github.com/juhovh/shairplay)
- [Analysis of Airplay 2](https://www.programmersought.com/article/2084789418/#)
- [Airplay mirroring server](https://github.com/serezhka/java-airplay-server)
- [shairport-sync](https://github.com/mikebrady/shairport-sync)
    - This is what I ended up going with
    - Doesn't support Airplay 2
    - [Open issue for multi-room support](https://github.com/mikebrady/shairport-sync/issues/535)

### Setup

Anyway, that's all to say that I landed on [shairport-sync](https://github.com/mikebrady/shairport-sync) for now. Luckily, there's even a package for it in the Alpine `testing` repository, so this is super easy.

```bash
~> echo "http://dl-cdn.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories
~> apk update
~> apk add shairport-sync
~> rc-service shairport-sync start
~> rc-update add shairport-sync default
```

Make sure you edit the configuration at `/etc/shairport-sync.conf`. I ended up just changing the `name` parameter.

And you're good to go! Again, give your Pi a reboot to make sure the service works.

## Conclusion

That wasn't too bad! This was a fun weekend project, and now I have the flex of a bare Raspberry Pi sitting atop my speakers. If this helped you out or you run into any issues don't hesitate to [write to me](mailto:appeld@uci.edu?subject=Hello!)!
