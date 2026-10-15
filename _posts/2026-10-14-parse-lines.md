---
layout: post
title: MQTT wildcards on a file you already captured
---

MQTT topic filters are not glob and not regex. I keep relearning that when I try to filter a capture with Unix tools.

On a live broker, `sensors/+` is one level (`sensors/lab` yes, `sensors/lab/temp` no). `sensors/#` is the remainder of the path, including zero extra levels depending on how you read the spec in the wild—people argue about `sensors/#` vs `sensors`. `#` as the whole filter is “everything,” except that a real broker will not deliver `$SYS/...` on a `#` subscription. A file on disk has no `$SYS` policy. If the line is in the dump, a filter is just a function of two strings.

I first filtered with `str.startswith`, which is the wrong model for topic levels. `topic.startswith("sensors")` matches `sensors_backup/foo`. `topic.startswith("sensors/")` misses a topic that is exactly `sensors`. Treating `#` as any substring matches payloads, comments, and the word `hash`.

What goes wrong with regex: `#` is not “rest of path.” `+` is not “any characters.” A regex you write in thirty seconds will accept `sensors/lab/temp` for `sensors/+` unless you anchor levels on `/`.

mqttlog implements the level walk in `topic_match` (`mqttlog/record.py`). Split both strings on `/`. `+` consumes one level. `#` is only legal as the last token and then succeeds. If the pattern is exhausted, the topic must be exhausted too—otherwise `sensors/+` would accept `sensors/lab/temp`. Tests in `tests/test_record.py` are the cases that bit me, not a tutorial:

```
sensors/lab/temp  vs  sensors/#     → True
actuators/fan     vs  sensors/#     → False
sensors/lab       vs  sensors/+     → True
sensors/lab/temp  vs  sensors/+     → False
```

`mqttlog csv` and `replay` call that same `topic_match`:

```
$ PYTHONPATH=. python3 -m mqttlog csv lab.log --topic 'sensors/#' --out -
$ PYTHONPATH=. python3 -m mqttlog replay lab.log --topic 'sensors/+' --out out/
```

I also write comments in dumps (`# swapped DHT at 20:04`). A strict “three fields or die” parser raises on those. Skipping `#` and blank lines when slurping a whole file is not the interesting part of this tool. The interesting part is that the filter you would have used on mosquitto still means the same thing on a file from last weekend.

A 30-line [filter_sensors.py](https://gist.github.com/nolan-alexander/362209c525a3f4e943b50d78419c41df) is what I used before the package existed. It implements “starts with `sensors/`” only. It does not implement `+`. If you care about the difference, use the library.

Code, MIT, CI: [nolan-alexander/mqttlog](https://github.com/nolan-alexander/mqttlog). I want that difference (file vs broker, `+` vs `#`) to be the thing a hiring reader remembers, not that I wrapped `str.split` in a module.
