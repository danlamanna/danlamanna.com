---
title: "GNU Parallel for Serial Execution"
date: 2023-06-10T15:22:38-04:00
draft: false
---
It's sometimes useful to use [GNU Parallel](https://www.gnu.org/software/parallel/) for running things serially. Why? Because it remembers
what it's done, it can pick up where it left off, store output per job, and [a ton of other stuff](https://www.gnu.org/software/parallel/parallel_examples.html).

### Example: re-encoding videos
Input file:
{{< highlight text >}}
some-video.mp4
some-other-video.mp4
...
{{< / highlight >}}

Command:
{{< highlight bash >}}
parallel --arg-file input.txt ffmpeg {} -o {.}.reencoded.mp4
{{< / highlight >}}

This will run `ffmpeg` on every line of input with `-o` set to the input line (without the extension) plus `.reencoded.mp4`.

If `--joblog` is given, Parallel keeps a log of every completed job and some metadata e.g.
{{< highlight bash >}}
Seq    Host    Starttime    JobRuntime    Send    Receive    Exitval    Signal    Command
1      :    1686425772.721         57.002    0    0    0    0    ffmpeg some-video.mp4 -o some-video.reencoded.mp4
2      :    1686425829.720         85.124    0    0    0    0    ffmpeg some-other-video.mp4 -o some-other-video.reencoded.mp4
...
{{< / highlight >}}

Parallel can use the joblog to pick up where it left off, just add `--resume`.

{{< highlight bash >}}
parallel --arg-file input.txt \
         --joblog reencode_job.joblog \
         --resume \
         ffmpeg {} -o {.}.reencoded.mp4
{{< / highlight >}}

This runs Parallel with one invocation of `ffmpeg` at a time, keeps track of every command it runs, and can pick up where it left off.

