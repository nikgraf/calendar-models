# Solunivo Assistant model

A small language model that reads what a person says to a calendar app and
writes it as JSON: the calendar entry for "Lunch with Tom tomorrow at 1",
then "make it 2", "invite Anna", "every Friday except in August"; and,
since v2, what to search for in "find 90 minutes for focus next week,
mornings". It is [Qwen3.5 0.8B](https://huggingface.co/Qwen/Qwen3.5-0.8B)
fine-tuned for these two jobs, told which one by the prompt's first line,
and it runs in [Solunivo](https://solunivo.com)'s Assistant and its find a
time through llama.cpp: on the iPhone with
[llama.rn](https://github.com/mybigday/llama.rn), on the Mac with
llama.cpp's own `llama-server`, the same commit. The app downloads it
once, when the person asks for it.

## Files

Each version is a [release](../../releases) of this repository: the model
file the app downloads, and the adapter it was made from.

| Version | File | Bytes | SHA-256 |
| --- | --- | --- | --- |
| v2 | `solunivo-assistant-q8_0.gguf` | 833,592,288 | `cfae339ce992836c607e123fc2108d21ab02f79ede77f409ec08309888f38230` |
| v2 | `solunivo-assistant-lora.zip` | 40,043,542 | `daba09152c56ecbd503658467e244d7e4686a87d71cd8502671e7bae8b364a1e` |
| v1 | `solunivo-assistant-q8_0.gguf` | 833,592,288 | `954ca95027feb405e2f98ef26fdeb0814d2fdbf4bd2ebddbeb9e9c321ba838d3` |
| v1 | `solunivo-assistant-lora.zip` | 40,055,673 | `2817c38384c3bf2bc57532b8cd4a179c07357a4a9b9e5841bdcf34dbe5cac65a` |

GGUF, quantized to Q8_0 with llama.cpp v0.8.0 (`8642541`), the version
llama.rn vendors.

The zip holds the LoRA adapter as [mlx-lm](https://github.com/ml-explore/mlx-lm)
wrote it (`adapters.safetensors`, `adapter_config.json`) and its training
settings (`lora.yaml`). Written into the base model at revision `2fc0636`
with `fuse_hf.py` (in nikgraf/calendar, below), it gives the very weights
the GGUF was converted from, byte for byte, so another quantization needs
no new training run.

In this repository, per version:

- `data/<version>/`: the training data (see below).
- `reports/<version>/`: the bench reports the scores below come from,
  which the bench can compare a later model with (`MODEL_BENCH_COMPARE`),
  and the model's answers to the held-back turns
  (`held-back-answers.jsonl`: the prompt, the teacher's answer as `want`,
  the model's as `got`).

## Where it comes from

- **Base model:** `Qwen/Qwen3.5-0.8B` at revision `2fc0636`, Apache License
  2.0, by the Qwen team (Alibaba Cloud).
- **What was changed:** a LoRA (rank 16, every layer; batches of 4
  examples, gradients summed over two; three passes: 8,328 batches for v2,
  4,496 for v1) was trained on the examples below with
  [mlx-lm](https://github.com/ml-explore/mlx-lm), merged into the base
  weights, converted to GGUF and quantized. The tokenizer and chat template
  are the base model's. v2's adapter is the run's last; the checkpoint
  with the lowest validation loss agreed less often with the held-back
  answers.
- **Training data:** v2: 11,103 examples (598 more held back to check the
  result): 9,155 first phrases and follow-ups to a draft, 1,948
  find-a-time sentences. v1: 5,994 turns (330 held back). English and
  German. All of it is synthetic. The phrases, and the made-up people and places in
  them, were written by DeepSeek V4-Pro from written briefs; the answers by
  DeepSeek V4-Flash with brief thinking, through DeepSeek's own API, in
  November 2026. DeepSeek V4's weights are MIT-licensed, and DeepSeek's Open
  Platform Terms of Service (effective 2026-04-29), section 4.2(3), allow
  using its outputs for "training other models (such as model
  distillation)". No user data, no personal data, and no output of Claude
  or any other Anthropic model went into the data. Every email address in
  it is at `example.com`.
- **The code** that made the data, trained the model and scores it is in
  [nikgraf/calendar](https://github.com/nikgraf/calendar), under
  `apps/desktop/e2e/bench/` (`train/README.md`), and the code that asks it
  in the app is `packages/ai/src/tuned.ts`.

## The training data (`data/v2/`, `data/v1/`)

Each step's output, in the order the steps ran (`train/README.md` in
nikgraf/calendar says how each was made). v2 keeps v1's threads and adds
briefs for what v1 missed (time shifts and lengths, guests removed,
alerts added or dropped, repeats stopped, all day instead, renames) and
find a time's sentences. Counts are v2's:

| File | What it holds |
| --- | --- |
| `names.json` | The made-up people and places the phrases may name. |
| `threads.jsonl` | The practice threads as written, each with the brief it was written for (language, way of writing, what it exercises): 6,608. |
| `labelled.jsonl` | The 6,224 threads kept (repeats, phrases close to a bench case and copies of a brief's example left out), each with the day it is said on and the teacher's answer to every turn. |
| `said.jsonl` | For every turn: how the words gave the day (by its number, as an amount of time, otherwise, or not at all), the start and the end (a clock time, an amount later or earlier, a length), the weekdays a repeat rules out, and whether they ask for no notification. |
| `finder.jsonl` | Find a time's sentences as written, each with its brief: 2,166 (v2 only). |
| `finderLabelled.jsonl` | The 2,061 kept, with the day each is said on and the teacher's answer through the app's finder prompt (v2 only). |
| `train.jsonl`, `valid.jsonl` | The chats the model was trained on (v2: 11,103) and checked on (598): the prompt the app sends, and the answer to learn. |
| `stats.json` | Counts: by language, first phrases and follow-ups, how days and times were said, the finder's sentences. |

## How to ask it

One user turn, thinking off (`enable_thinking: false` in the chat
template), greedy decoding (temperature 0). The answer is one JSON object.

A first phrase:

```text
Calendar entry as JSON.
Days: Thu 2026-11-12 (today), Fri 2026-11-13, Sat 2026-11-14, Sun 2026-11-15, Mon 2026-11-16, Tue 2026-11-17, Wed 2026-11-18, Thu 2026-11-19, Fri 2026-11-20, Sat 2026-11-21, Sun 2026-11-22, Mon 2026-11-23, Tue 2026-11-24, Wed 2026-11-25
Phrase: Lunch tomorrow at 1
```

A later turn, with the draft as it stands and what was said before:

```text
Calendar entry as JSON, after the change.
Days: Thu 2026-11-12 (today), Fri 2026-11-13, …, Wed 2026-11-25
Draft: {"kind":"event","title":"Lunch with Tom","date":"2026-11-13","startTime":"13:00","endTime":"14:00","location":"Café Roth","guests":["tom.baker@example.com"],"reminders":[30]}
Said before: ["Lunch with Tom tomorrow at 1"]
Now: make it 2
```

The answer is the whole entry after the turn, with only the parts that have
a value, in this order: `kind` (`event` or `task`), `title`, `date`,
`startTime`, `endTime` (24-hour `HH:MM`), `timeAssumed`, `location`,
`repeat` (`freq`, `interval`, `byDay`, `except`, `count`, `untilDate`),
`guests`, `invite`, `uninvite`, `notes`, `reminders` (minutes before the
start). `{"nothingToSchedule":true}` when there is nothing to put in a
calendar.

A day inside the listed two weeks is written out (`2026-11-13`). A day the
model cannot look up there is written the way it was said, and the app
works out the date: `"14"` for "on the 14th", `"03-14"` for "March 14",
`"+2w"` for "in two weeks". Since v2 there is no `date` where the words
name no day ("Lunch at 1"): the app puts the entry on the day on show, or
keeps the draft's. A repeat lists the weekdays the words ruled out in
`except` ("every day except Sunday": `freq: "weekly"`, `except: ["SU"]`).

Since v2 a time said as an amount is written as said too: `startTime`
`"+30m"` or `"-1h"` moves the draft's start ("half an hour later" moves
the end as well: `endTime: "+30m"`), `endTime` `"45m"` or `"2h"` is a
length after the start ("make it 45 minutes"). Whole hours are written in
`h`, anything else in minutes. `reminders: []` on a first phrase means
"no notification". The app's code for all of it is
`packages/ai/src/said.ts`.

Find a time (v2):

```text
Free time as JSON.
Days: Thu 2026-11-12 (today), Fri 2026-11-13, …, Wed 2026-11-25
Phrase: find 90 minutes for focus next week, mornings
```

The answer has only what the words state, in this order:
`durationMinutes`, `windowStartDate` and `windowEndDate` (looked up in
the listed days), `daysOfWeek` (ISO, 1 = Monday), `earliestTime` and
`latestTime` (24-hour `HH:MM`; parts of the day and meals become
bounds), `title`. What it leaves out is the app's to fill: the coming
week, the finder's own length.

## How well it does

Scored on the app's model bench (72 cases, none of them in the training
data or close to one; first phrases and follow-up turns, recordings run as
their transcripts), each file served by llama.cpp on a Mac, the app's code
at `fb85077f` (its v2 pull request):

| | Cases | Turns |
| --- | --- | --- |
| v2, Q8_0 | 60 / 72 | 106 / 118 |
| v1, Q8_0 | 59 / 72 | 102 / 118 |

v2 gets right what v1 missed most: arithmetic on a time ("30 minutes
later", "an hour earlier", "make it 45 minutes"), "stop repeating", "all
day instead", a rename, "an hour before as well", "remove Tom". It misses
cases v1 got: a place narrowed to its city ("it's in Vienna" after a
venue writes the city alone; its data has no such turn), a long phrase's
note, "Call tomorrow at 4, invite Max" read as a to-do. Reports:
`reports/v2/bench-20261114-fb85077f.json` and, for v1 on the same code,
`reports/v2/bench-v1-20261114-fb85077f.json`. v1's own release scored
57 of the bench's 70 cases then (`reports/v1/`).

Find a time, on the app's finder bench (40 sentences, each part a person
means: length, window, weekdays, bounds, what for), read as the app reads
it, without word rules:

| | Sentences |
| --- | --- |
| v2, Q8_0 | 36 / 40 |
| Apple's on-device model, as the app asked it before v2 (with word rules) | 12 / 40 |
| The teacher, DeepSeek V4-Flash | 39 / 40 |

Report: `reports/v2/finder-bench-20261114-fb85077f.json`.

On v2's 598 held-back examples it gives its teacher's answer for 376 of
494 turns once said days and times are worked out (235 of 312 first
phrases, 141 of 182 follow-ups) and for 82 of 104 find-a-time sentences.

A score here says the approach works on phrases of this kind, not how the
model reads what real people say: the practice phrases were written by a
model, and follow-ups are short and alike.

## Limits

- English and German only.
- It reads one phrase about one calendar entry, or one sentence asking
  for free time, at a time. It does not chat or answer questions, and it
  finds no slot itself: it says what to search for.
- It knows the two weeks of days it is shown; a date further out comes back
  as said and needs the app's arithmetic.
- v1 was never taught to turn notifications off ("no reminder"); v2 was.
- Loaded with a 4,096-token context it takes about a gigabyte and a half of
  memory, so the app offers it on iPhones with 8 GB.

## License

Apache License 2.0, as the base model (`LICENSE`). This model is a modified
version of Qwen3.5-0.8B: fine-tuned with a LoRA, merged and quantized as
described above.
