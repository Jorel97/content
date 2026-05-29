---
title: 'Pre-recorded Speech-to-Text API'
description:
  'A pre-recorded speech-to-text API converts existing audio or video files into
  text through a request-response or batch transcription workflow.'
---

# Pre-recorded Speech-to-Text API

## Definition

A pre-recorded speech-to-text API accepts an existing audio or video recording
and returns a transcript. Unlike live transcription, where audio is streamed as
it is captured, a pre-recorded workflow sends a complete file or a file URL to
the provider after the recording already exists.

These APIs are useful for engineering teams that need repeatable processing for
meetings, product demos, interviews, lectures, support calls, podcasts, or test
fixtures. A typical request includes authentication, the audio payload, a model
selection, and optional parameters such as language, punctuation, formatting,
speaker diarization, or keyword hints. The response usually contains the
transcript plus provider-specific metadata, confidence scores, word timings, or
speaker labels.

Pre-recorded transcription is often easier to test than live transcription
because the input file is stable. Engineers can run the same fixture through
the same provider settings, compare the generated transcript, and keep the
setup in a reproducible development environment. This is also a good fit for
tools such as Sapat, which convert local video files into an audio format,
submit that audio to a selected provider, and write the resulting transcript
next to the source media.

The main tradeoff is latency. A pre-recorded API normally waits until the file
is available before processing begins, so it is not meant for real-time captions
or voice assistants. In exchange, it can support larger files, richer
post-processing, and a simpler retry model for automation jobs.
