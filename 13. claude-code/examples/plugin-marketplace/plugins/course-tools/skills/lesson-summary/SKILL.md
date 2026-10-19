---
name: lesson-summary
description: Summarize one lesson folder of the AI LLM course. Use when the user asks what a lesson covers or which example to look at first.
argument-hint: <lesson folder, e.g. "10. LangGraph">
allowed-tools: Read, Glob
---

Summarize the course lesson in the folder "$ARGUMENTS".

1. List the files in the folder. Read only the first ~40 lines of each script, enough to see what it does. Do not run anything.
2. Write a short overview: what the lesson teaches, in two or three sentences.
3. Give a table with one row per example: file name, what it shows, and what it needs to run (API keys, Ollama, extra packages).
4. Suggest the order to go through the examples in.

Keep the whole answer under 300 words.
