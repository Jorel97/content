---
title: 'Run Deepgram transcription with Sapat'
description:
  'Build a reproducible Daytona workspace for Sapat and transcribe existing
  video files with a Deepgram provider.'
date: 2026-05-29
author: 'Jorel97'
tags: ['daytona', 'sapat', 'deepgram', 'speech-to-text']
---

# Run Deepgram transcription with Sapat

## Introduction

Video transcription looks simple until it becomes part of a real engineering
workflow. A one-off transcript can be created with a web upload form, but teams
usually need something more repeatable: a clean environment, predictable
provider settings, local fixtures, and an easy way to rerun the same command
when a model or prompt changes.

This guide shows how to run Sapat in a Daytona workspace and route
pre-recorded media through Deepgram. Sapat is a Python command-line tool that
converts local video files to audio, calls a selected speech-to-text provider,
and writes the transcript beside the source file. Daytona gives the workflow a
fresh, reproducible development environment so every contributor can start from
the same system packages, Python dependencies, and source branch.

The Deepgram provider used here is implemented in the companion Sapat pull
request [nibzard/sapat#59](https://github.com/nibzard/sapat/pull/59). The
provider uses Deepgram's pre-recorded Listen API, which accepts an audio payload
at `https://api.deepgram.com/v1/listen` and supports options such as model,
language, smart formatting, punctuation, diarization, and keyword hints. Once
that provider lands upstream, the same workflow can install Sapat from the main
repository instead of the feature branch.

## TL;DR

- Create a Daytona workspace from the Sapat repository or the Deepgram branch.
- Install Sapat with editable Python dependencies and make sure `ffmpeg` exists.
- Set `DEEPGRAM_API_KEY` in `.env`; never commit this file.
- Run Sapat with `--provider deepgram` against one `.mp4` file or a folder.
- Validate the `.txt` transcript and tune model, language, and formatting flags.

![Deepgram Sapat workflow in Daytona](/assets/20260529_run_deepgram_transcription_with_sapat_in_daytona_workflow.svg)

## Prerequisites

You need a working Daytona installation, Git, Python 3.10 or newer, and a
Deepgram account with an API key. You should also have at least one short video
fixture, preferably a file that can be shared internally without privacy risk.

This workflow sends audio to a hosted
[pre-recorded speech-to-text API](../definitions/20260529_definition_pre_recorded_speech_to_text_api.md).
Use recordings that your team is allowed to process with a third-party provider.
Do not put secrets, private customer media, or generated transcripts into Git.

## Step 1: Create the Daytona workspace

Start with a clean workspace so the transcription stack does not depend on
packages from your laptop. If the Deepgram provider PR has not merged yet, use
the fork that contains the companion branch:

```bash
daytona create https://github.com/Jorel97/sapat --code
git fetch origin codex/add-deepgram-provider
git checkout codex/add-deepgram-provider
```

After the branch is merged, use upstream Sapat instead:

```bash
daytona create https://github.com/nibzard/sapat --code
```

Open the workspace terminal and confirm the checkout:

```bash
git status --short --branch
python --version
```

The branch command should show a clean working tree. Keeping the workspace
clean matters because transcription experiments often produce media, logs, and
transcripts. Those artifacts are useful locally, but they should not be mixed
with source changes.

## Step 2: Install Sapat and ffmpeg

Sapat converts video to MP3 before calling the provider, so `ffmpeg` must be
available in the workspace. If your image already has it, this command prints a
version. If it is missing, install it with your workspace package manager.

```bash
ffmpeg -version
```

On a Debian or Ubuntu based workspace:

```bash
sudo apt-get update
sudo apt-get install -y ffmpeg
```

Install Sapat in editable mode so local provider changes are picked up without
rebuilding a wheel:

```bash
python -m pip install --upgrade pip
python -m pip install -e .
```

Then confirm the command-line interface loads:

```bash
sapat --version
sapat --help
```

The Deepgram branch adds a dynamically discovered provider named `deepgram`.
Sapat's current CLI selects providers with `--provider` or `-p`, rather than
the older `--api` flag shown in some historical examples.

## Step 3: Configure Deepgram safely

Create a local `.env` file from the example values. Keep it untracked.

```bash
cp .env.example .env
```

Add your Deepgram key and optional provider settings:

```bash
DEEPGRAM_API_KEY=replace-with-your-key
DEEPGRAM_MODEL=nova-3
DEEPGRAM_SMART_FORMAT=true
DEEPGRAM_DIARIZE=false
DEEPGRAM_PUNCTUATE=false
```

The provider reads `DEEPGRAM_API_KEY` for authentication and defaults to the
Listen endpoint. `DEEPGRAM_MODEL` lets you pin a model, while the formatting
flags decide which query parameters are sent with each request. Deepgram's
documentation notes that `smart_format`, `diarize`, and `punctuate` are query
options for pre-recorded transcription. Enable them deliberately because richer
features can change output shape and provider cost.

For a team workflow, put only placeholder variable names in committed files.
Store real values in Daytona secrets, your shell profile, or a private `.env`
that never leaves the workspace.

## Step 4: Add a small media fixture

Create a local folder for recordings and place one short `.mp4` file inside.
Keep the first test small. A 15 to 60 second fixture is enough to prove the
provider path, reduce cost, and make errors easier to read.

```bash
mkdir -p samples
cp ~/Downloads/product-demo.mp4 samples/product-demo.mp4
```

If you only have an audio file, wrap it in a small video container or adjust
Sapat's media handling in your branch. The current Sapat workflow is designed
around `.mp4` inputs and directory scans for `.mp4` files.

## Step 5: Transcribe one file with Deepgram

Run Sapat with an explicit provider, model, language, and transcription prompt:

```bash
sapat samples/product-demo.mp4 \
  --provider deepgram \
  --model nova-3 \
  --language en \
  --transcription-prompt "Daytona, Sapat, Deepgram" \
  --quality H
```

Sapat prints the selected provider and model before processing. Internally, it
converts the video to MP3, sends the binary audio to Deepgram, extracts the
first transcript from `results.channels[0].alternatives[0].transcript`, and
writes a text file next to the input.

After the command completes, inspect the output:

```bash
ls samples
sed -n '1,80p' samples/product-demo.txt
```

If the transcript exists and the first lines match the recording, the full path
is working: Daytona workspace, local conversion, Deepgram request, response
parsing, and Sapat output writing.

## Step 6: Process a directory

Once a single file works, use the same command against a folder:

```bash
sapat samples \
  --provider deepgram \
  --model nova-3 \
  --language en \
  --transcription-prompt "Product names: Daytona, Sapat" \
  --quality M
```

Sapat scans the directory for `.mp4` files and processes them with a progress
bar. This makes it useful for batches such as meeting recordings, customer
interviews, or product demo libraries. Keep batches modest at first. A failed
environment variable or a provider quota issue is easier to debug on three
files than on a hundred.

For repeatable team runs, save the exact command in a script:

```bash
#!/usr/bin/env bash
set -euo pipefail

sapat samples \
  --provider deepgram \
  --model "${DEEPGRAM_MODEL:-nova-3}" \
  --language "${TRANSCRIPT_LANGUAGE:-en}" \
  --transcription-prompt "${TRANSCRIPT_KEYWORDS:-Daytona, Sapat}" \
  --quality "${TRANSCRIPT_QUALITY:-M}"
```

Commit the script only if it contains placeholders. Keep `.env`, recordings,
and generated transcripts out of source control unless they are approved test
fixtures.

## Step 7: Tune output quality

Start with `nova-3`, smart formatting, and a short keyword prompt. That
combination is a practical default for demos, product names, and technical
terms. If you need speaker labels, set `DEEPGRAM_DIARIZE=true` and run the
same short fixture again. If you need more readable prose, enable
`DEEPGRAM_PUNCTUATE=true` or keep `DEEPGRAM_SMART_FORMAT=true`.

The `--language` flag is useful when recordings are known to be in one
language. Passing a language reduces ambiguity and makes regression testing
more stable. The `--transcription-prompt` value is mapped to Deepgram keyword
hints in the companion provider, so use compact terms rather than a long
instruction paragraph.

When comparing runs, change one variable at a time. For example, test model
first, then diarization, then keyword hints. Store notes outside the transcript
file so downstream systems can ingest clean text without experiment metadata.

## Troubleshooting

**Provider is not available:** Check that `DEEPGRAM_API_KEY` is present in the
same shell that runs Sapat. The provider registry only registers providers whose
required environment variables are available.

**The command says ffmpeg is missing:** Install `ffmpeg` in the workspace and
restart the terminal. Sapat needs it before the Deepgram request is made.

**Deepgram returns 401 or 403:** Regenerate the key or confirm the workspace is
using the correct `.env` file. Do not paste keys into pull requests, logs, or
screenshots.

**The transcript is empty:** Try a shorter file, confirm the audio track plays,
and rerun with `--quality H`. Also check whether the provider response contains
channels and alternatives. Empty output usually means the request succeeded but
the provider did not return a transcript where Sapat expected it.

**The wording is close but misses product names:** Add a compact
`--transcription-prompt` with the terms that matter. For example:
`"Daytona, Sapat, dev container, workspace"`.

## Conclusion

Daytona and Sapat make a good pairing for transcription experiments because the
environment and command are both repeatable. Daytona gives every contributor a
fresh workspace. Sapat keeps the provider selection behind one CLI. The
Deepgram provider adds a hosted pre-recorded transcription path with useful
formatting options and straightforward API-key configuration.

Use the short-file workflow first, then move to directory batches once the
provider, model, language, and formatting settings produce transcripts your
team trusts. When the output is stable, the same setup can feed QA review,
documentation, summarization, search indexing, or any other downstream AI
workflow that benefits from clean text.

## References

- [Sapat repository](https://github.com/nkkko/sapat)
- [Companion Deepgram provider PR](https://github.com/nibzard/sapat/pull/59)
- [Deepgram pre-recorded Listen API](https://developers.deepgram.com/reference/speech-to-text-api/listen)
- [Deepgram diarization docs](https://developers.deepgram.com/docs/diarization)
- [Daytona documentation](https://www.daytona.io/docs)
