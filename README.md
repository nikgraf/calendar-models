# Solunivo Assistant model

A small language model that reads what a person says to a calendar app and
writes the calendar entry as JSON: "Lunch with Tom tomorrow at 1", then
"make it 2", "invite Anna", "every Friday except in August". It is
[Qwen3.5 0.8B](https://huggingface.co/Qwen/Qwen3.5-0.8B) fine-tuned for this
one task, and it runs in [Solunivo](https://solunivo.com)'s Assistant
through llama.cpp: on the iPhone with
[llama.rn](https://github.com/mybigday/llama.rn), on the Mac with
llama.cpp's own `llama-server`, the same commit. The app downloads it
once, after asking.

## Files

Each version is a [release](../../releases) of this repository.

| Version | File | Bytes | SHA-256 |
| --- | --- | --- | --- |
| v1 | `solunivo-assistant-q8_0.gguf` | 833,592,288 | `954ca95027feb405e2f98ef26fdeb0814d2fdbf4bd2ebddbeb9e9c321ba838d3` |

GGUF, quantized to Q8_0 with llama.cpp v0.8.0 (`8642541`), the version
llama.rn vendors.

## Where it comes from

- **Base model:** `Qwen/Qwen3.5-0.8B` at revision `2fc0636`, Apache License
  2.0, by the Qwen team (Alibaba Cloud).
- **What was changed:** a LoRA (rank 16, every layer; 4,496 batches of 4
  examples, gradients summed over two; about three passes) was trained on the examples below with
  [mlx-lm](https://github.com/ml-explore/mlx-lm), merged into the base
  weights, converted to GGUF and quantized. The tokenizer and chat template
  are the base model's.
- **Training data:** 5,994 practice turns (330 more held back to check the
  result), English and German, first phrases and follow-ups to a draft.
  All of it is synthetic. The phrases, and the made-up people and places in
  them, were written by DeepSeek V4-Pro from written briefs; the answers by
  DeepSeek V4-Flash with brief thinking, through DeepSeek's own API, in
  November 2026. DeepSeek V4's weights are MIT-licensed, and DeepSeek's Open
  Platform Terms of Service (effective 2026-04-29), section 4.2(3), allow
  using its outputs for "training other models (such as model
  distillation)". No user data, no personal data, and no output of Claude
  or any other Anthropic model went into the data.
- **The code** that made the data, trained the model and scores it is in
  [nikgraf/calendar](https://github.com/nikgraf/calendar), under
  `apps/desktop/e2e/bench/` (`train/README.md`), and the code that asks it
  in the app is `packages/ai/src/tuned.ts`.

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
`"+2w"` for "in two weeks". A repeat lists the weekdays the words ruled out
in `except` ("every day except Sunday": `freq: "weekly"`, `except: ["SU"]`).
The app's code for both is `packages/ai/src/said.ts`.

## How well it does

Scored on the app's model bench (70 cases, none of them in the training
data or close to one; first phrases and follow-up turns,
recordings run as their transcripts), the file served by llama.cpp on a Mac:

| | Cases | Turns |
| --- | --- | --- |
| This model, Q8_0 | 57 / 70 | 99 / 115 |
| This model before quantizing (MLX, bf16) | 58 / 70 | 100 / 115 |
| Its teacher, DeepSeek V4-Flash (the bench's plain prompt, the schema in it) | 65 / 70 | |

On the 330 practice turns held back from training, it gives its teacher's
answer for 242 once said dates are worked out: 161 of 222 first phrases and
81 of 108 follow-ups (measured before quantizing). Most of what it misses on
the bench are follow-ups: arithmetic on a time ("30 minutes later" as
`17:60`), "stop repeating", "remove Tom", "all day instead".

A score here says the approach works on phrases of this kind, not how the
model reads what real people say: the practice phrases were written by a
model, and follow-ups are short and alike.

## Limits

- English and German only.
- It reads one phrase about one calendar entry at a time. It does not
  chat, answer questions or find free time.
- It knows the two weeks of days it is shown; a date further out comes back
  as said and needs the app's arithmetic.
- It was never taught to turn notifications off ("no reminder"): its data
  has no such turn.
- Loaded with a 4,096-token context it takes about a gigabyte and a half of
  memory, so the app offers it on iPhones with 8 GB.

## License

Apache License 2.0, as the base model (`LICENSE`). This model is a modified
version of Qwen3.5-0.8B: fine-tuned with a LoRA, merged and quantized as
described above.
