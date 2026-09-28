---
layout: post
title: Keeping MQTT history on disk without a live broker
---

Most MQTT tooling assumes you are looking at *now*. `mosquitto_sub`, MQTT Explorer, a Telegraf consumer: they subscribe, they print, they forget. That is the right default for a dashboard. It is the wrong default when you are staring at a breadboard at 21:00 and trying to answer “did `sensors/door/state` actually change while I was in the other room.”

On the bench Pi I run mosquitto on 1883 with no TLS (it is a LAN I can see). For a long time the “history” was a tmux pane:

```
mosquitto_sub -h 127.0.0.1 -t '#' -v >> ~/captures/lab.log
```

`-v` is topic, a space, then the payload. That looks like a log file until you treat it like one.

**Payloads have spaces.** A door payload of `open with bounce` makes `awk '{print $3}'` a lie. **I leave notes in the file.** `# swapped DHT, yellow is data` is not a topic. **Replaying through a broker to filter is theatre.** If the bytes are already on disk, starting mosquitto so I can subscribe to `sensors/#` is a round trip I do not need.

I tried the usual Unix-shaped mistakes. `grep sensors lab.log` also matches `sensors` inside a payload. `awk '$2 ~ /^sensors\//'` drops my comments and still cannot see a payload as one field. I once passed `sensors/#` to awk as if `#` were a regex. It is not; in MQTT it is a wildcard that only means something at a topic-filter parser.

Existing tools did not fit this job. MQTT Explorer wants a live session. `mosquitto_sub` is live. sqlite-plus-broker is a project, not a Thursday night. I wanted a line format I could `tail`, `grep` (carefully), rotate, and CSV without standing anything up.

The line format I settled on is three fields, payload allowed to contain spaces:

```
<iso-timestamp> <topic> <payload...>
```

Topic is one token. Payload is the rest of the line, spaces included. Lines starting with `#` are mine, not the bus. Rotation is by byte size because a 1 Hz stream fills a 4G card in a week if you never cut it (I did that once).

[mqttlog](https://github.com/nolan-alexander/mqttlog) is the CLI I use for that file. MIT, unittest, GitHub Actions. It is not a broker, not a time-series database, and not a replacement for MQTT Explorer. It is the missing “I already have the dump” path.

```
$ PYTHONPATH=. python3 -m mqttlog replay lab.log --topic 'sensors/#' --out logs/
wrote 128 records to logs/mqttlog-000.log

$ PYTHONPATH=. python3 -m mqttlog csv lab.log --topic 'sensors/+' --out -
ts,topic,payload
2026-10-13T20:11:00-07:00,sensors/lab/temp,20.0
```

A real capture (comments included) is in [this gist](https://gist.github.com/nolan-alexander/f4c0c24e2268719196141bcfadd2ea31). If you do the same thing with `>> lab.log` and then wish you had `sensors/#` after the fact, that is the hole this fills. I am in Portland; if you hire people who write small Python around sensors and serial, that repo is the thing I would rather you clone than a LeetCode dump.
