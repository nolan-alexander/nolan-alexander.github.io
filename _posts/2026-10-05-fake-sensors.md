---
layout: post
title: CI for a tool that talks to hardware it does not have
---

mqttlog’s job is files. The files come from a Pi, a DHT22, a reed switch, and a radio that browns out a USB hub when it transmits. None of that belongs in GitHub Actions.

If the tests require the breadboard, they will not run on a PR. If they do not run on a PR, the CI badge is decoration. I wanted the opposite: a skeptical skim of [mqttlog](https://github.com/nolan-alexander/mqttlog) should see `unittest` actually exercise rotation, topic filters, and the CLI.

The cheap way out is to check in one golden `sample.log` and only test parse. That misses the rotator. Rotation depends on byte counts, not on “a file exists.” The tests write records in a temp dir with `--max-bytes 40` so a handful of lines must spill into `mqttlog-001.log`. If that test only ran when I had the DHT plugged in, it would not exist.

`mqttlog sim` is a fixture generator, not a digital twin. It emits lab temp, RH, and a door contact on a 15s cadence in the same line format as a live dump. Door is `open` every fourth tick so the CSV is not a constant. Temp walks in 0.3° steps because a flat 20.0 looks like I forgot to write the loop. It is fake on purpose. CI should not pretend it smelled the room.

```
$ PYTHONPATH=. python3 -m mqttlog sim --count 8 --out /tmp/sim --max-bytes 120
wrote 8 records to /tmp/sim/mqttlog-001.log
```

The hardware still taught me something tests cannot. The CP2102 and the radio shared a cheap hub. Meter on TX dropped to 4.4V when the radio keyed. Python was innocent. Baud 115200 dropped characters; 57600 held. I keep that in a private note so the next time a dump looks “corrupted” I check 5V before I check `select()`. That story does not go in CI. CI gets the simulator.

`python -m unittest discover -s tests -v` covers parse, `+`/`#`, rotation, and CLI replay/csv. `.github/workflows/ci.yml` runs it on push. If you work on firmware-adjacent Python and you have been burned by “the test needs the board,” that is the design I am advertising: keep the I/O in files, fake the stream, never make Actions SSH to a Pi in my apartment.
