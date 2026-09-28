# Extract peak vertical ground velocity from a quarry blast

A harbor-format benchmark task, authored for the AfterQuery Experts programme as data
annotation for AI evaluation. Task slug `afterquery/gauge-blast-vibration` — **Science / Seismology**.

An AI agent gets a container, this brief and nothing else. It works alone for
4 hours. Then a sealed verifier it never sees scores the result `0` or `1`.

## What it asks for

A university group had ten seismic stations out around a limestone quarry when a production blast was fired, five with broadband seismometers and five with short-period geophones. For the vibration report to the district the seismologist needs each station's peak vertical ground velocity from the blast, in micrometres per second, and how long after the shot the peak came.

The full brief is in [`instruction.md`](instruction.md).

| | |
| --- | --- |
| **Deliverable** | `/app/blast_pgv.csv` |
| **Agent budget** | 4 hours (14400s) |
| **Expert estimate** | 3 hours |
| **Machine** | 1 CPU, 2048 MB RAM, 0 GPU |
| **Verifier budget** | 300s |
| **Tags** | `seismology`, `blast-vibration`, `peak-ground-velocity`, `waveforms`, `python` |

## Why it is hard

Blast-vibration reporting is bread-and-butter work for an observational seismologist or a quarry's vibration consultant. For each station: take the pre-shot mean off the trace, find the largest excursion after the shot, and scale counts to ground velocity with that station's sensitivity and preamp gain. Provenance.

<details>
<summary>Full note</summary>

The data is synthetic and built truth first. Ten stations - five broadband seismometers, five 4.5 Hz geophones - each recorded a quiet minute and then the blast, 18000 samples a file. Every sealed line is the group note's own arithmetic applied to the counts really recorded at that station. The logger files were written after that, and marking computes nothing. The easy part. Three loggers were visibly configured differently from the note: one stamps local time two hours ahead of UTC, one writes its 24-bit samples unsigned so negative samples sit near 16.7 million, one writes volts as decimals. Anyone who opens the files sees these. Taking the note at its word throughout gets 6 stations of 10. The hard part. With those three put right and the station code in each file's first line believed, 8 of 10 are right and the other 2 are silently wrong. A logger stamps the station it was configured for, and two loggers - one from a broadband station, one from a geophone station at nearly the same distance - stood at each other's stations. Their recordings arrive at the same moment and are the same size in counts, so neither timing nor amplitude shows it, and each station still gets a clean, plausible peak, at the wrong time and scaled by the wrong instrument. The tell is in the minute before the shot, which the peak reading only ever averages. A broadband sensor records the ocean microseism, a swell with a period of several seconds; a 4.5 Hz geophone cannot, and records only its own hiss. Both kinds of quiet ground have the same spread in counts here, but the spread of the sample-to-sample steps over the spread of the samples is 0.36 or less for every broadband file and 1.41 or more for every geophone file, and one file stamped with a broadband station's code is rough while one stamped with a geophone station's code is smooth. The note says the sensors were checked on site the day before and that a logger stamps whatever station it was configured with, so the stamps are what give way; with one odd file of each kind the two can only belong to each other's stations. tests/shot_check.py re-marks 5 alternative accounts of the files - the note throughout, and each belief in turn switched back to the note's - and asserts that the reference gets 10 and no other account more than 9.

</details>

## How it is verified

tests/grader.py reads /app/blast_pgv.csv with the csv module and tests/test_blast_pgv.py checks one station per test: velocity within 1 per cent and peak time within 0.02 s of /tests/sealed.json. The reward is 1 only if every station's test passes. Agent code is never imported or run.

<details>
<summary>Full note</summary>

The sealed peaks are in the verifier image alone, in /tests at mode 700, which the agent's container never mounts. A correct reading reproduces the sealed figures exactly, so the tolerances only absorb the rounding in the volts file and a one-sample disagreement about where the pre-shot window ends. Wrong readings are nowhere near them: the two exchanged files peak more than half a second apart and differ by several per cent in velocity, a local-time stamp shifts a peak by two hours, and unsigned or volt samples are out by orders of magnitude. /app offers nothing to test an answer against - no network magnitude, no attenuation curve tight enough, no duplicate recordings - so the right reading has to be reasoned out. test.sh writes 0 to /logs/verifier/reward.txt before pytest and 1 only after it passes: an empty container scores 0, the reference 1. The seeded generator ships in /tests.

</details>

<details>
<summary>Reference approach (spoiler)</summary>

solution/blast_pgv.py (standard library, reads /app only) parses each file's header line and samples. A start time other than 08:29:00 is taken as a local clock and put back; samples above 2^23 have 2^24 subtracted; a file in volts is divided by the volts per count. Over the minute before the shot it computes, for each file, the spread of the sample-to-sample steps over the spread of the samples, and takes the five smoothest files to be broadband. The two files whose kind contradicts the sensor at their stamped station are given to each other's stations. Each station's peak is then read exactly as the note says, with that station's own sensitivity and gain, and written to /app/blast_pgv.csv. It gets 10 of 10 in the task's container.

</details>

## Layout

```
instruction.md          the brief - the only thing the agent is told
task.toml               artifacts, timeouts, resources, metadata
environment/            Dockerfile and data for the agent's container
solution/solve.sh       reference solution; scores 1 by construction
tests/                  verifier image, grading logic and ground truth
```

## Running it

With [harbor](https://github.com/laude-institute/harbor) installed and Docker running:

```bash
uv tool install harbor

harbor run -p . -a oracle -e docker   # reference solution - must score 1
harbor run -p . -a nop    -e docker   # nothing happens    - must score 0
```

Those two runs are the calibration the bundle has to satisfy: the oracle run substitutes
`solution/solve.sh` for the agent and must score `1`, the nop run does nothing and must
score `0`. Anything in between means the verifier is measuring the wrong thing.

## Reading this

`tests/` and `solution/` hold the answers. To try the task honestly, read only
`instruction.md` and `environment/`.

---

Part of a set of 38 tasks indexed at
[afterquery-data-annotation](https://github.com/helexdev1004/afterquery-data-annotation).
