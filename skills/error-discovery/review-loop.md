# Review Loop

This is the ongoing interactive phase. The app is running and the human is reviewing. You are an active participant.

## The two modes: breadth and depth

Error discovery alternates between two modes.

**Breadth**: cover as much of the dataset as possible. Pick diverse samples, fill cluster gaps, add random records. The goal is to find different failure modes.

**Depth**: once a mode is found, examine it thoroughly. Re-review earlier items and refine the definition. After the human has annotated about 5 distinct records, scan all records for instances.

Alternate between these. Review broadly until you find something, then examine that mode in depth, then go broad again.

## Progress updates

Tell the user what you are doing at each step during the review loop. When you categorize annotations, say what modes you found. When you spawn a subagent, say which failure mode it is scanning for. When you propose new samples, say why you chose them. Do not go silent while background work runs.

## Launch and monitor

1. Start the Python server in the background.
2. Open the app in the browser.
3. Tell the user the app is ready. Explain the interaction briefly.
4. **Monitor the annotations file.** Watch `error_discovery_data/annotations.json` for new or deleted annotations by comparing content every 2 seconds.
   - Claude Code: use the Monitor tool with a read-compare loop that also reports if the server stops responding. Each detected change becomes a notification, so you react in real time. Monitors expire, so re-arm yours each time it does, and check that no annotations changed in the gap.
   - Everything else: write a polling script, run it in the background with output appended to a log file, and read the new lines whenever you take a turn.

   Tell the user how you are watching for annotations. Do not use `inotifywait` or filesystem events — poll with a simple read-compare loop for cross-platform reliability.

## Process annotations as they arrive

When new annotations appear:
1. Read all annotations.
2. Categorize each into a failure mode. Match to an existing mode or create a new one.
3. Maintain a running taxonomy: `{mode_name: {description, count, example_ids, example_quotes[]}}`.
4. Push the updated taxonomy to `POST /api/patterns`.
5. Track which records have been reviewed and which clusters or dimensions are covered.

Read the passage around each note before categorizing it. Tell the human when you merge or rename a mode, and ask when a note looks cut off or could mean two things. Build each mode only from the human's notes, and offer to split any merge they disagree with.

The human should be able to tell the state of the session from the app alone. When a change needs a page reload, say so. When work is running in the background, say what it is and where its results will appear.

## Depth mode: scan for failure mode instances

Wait until the human has annotated about 5 distinct records before the first corpus-wide scan; early annotations are too weak a signal of what the human considers a failure. Never scan before the human has reviewed anything. After that, when you discover a new failure mode (or an existing one becomes clearer), scan **all** records for instances. Include both reviewed and unreviewed records.

**Spawn one background subagent per failure mode.** Give the subagent the mode name, description, example quotes, the human's notes, and what falls outside the mode. It reads through all records and returns a list of suggested annotations: `{record_id, block, text, start, end, why, confidence}`. One mode = one subagent, not one per record. Multiple modes can run in parallel. This happens in the background while the human keeps reviewing.

For a corpus too large to read once per mode, first extract candidate passages per mode with a broad filter, and give each subagent those plus the preceding user message. Tell the human this scan covers filtered candidates, because real instances outside the filter can be missed. Tighten a filter that matches most of the corpus by keying on what the human's notes actually flagged.

Publish each subagent's results as soon as it returns, and tell the human, so the app never sits empty while results exist. Check that every quote occurs verbatim in its record before pushing to `POST /api/suggestions`, and skip passages the human has already annotated. Merge with the current queue, so accepted and dismissed suggestions stay resolved. The UI polls for new suggestions automatically and shows them in the Progress view queue and as purple highlights in the article view.

Keep the queue reviewable: list every match in records already in the sample, plus the strongest few per mode from outside it. Save the full set of matches inside `error_discovery_data/`, so it outlives the session, and report the listed and total counts. A mode created after a scan started goes in the next round.

There are two kinds of suggestions:
1. **Already-reviewed records**: the reviewer may have missed an instance because the mode was not yet in their head when they read the record (criteria drift).
2. **Unreviewed records**: new coverage. Add the strongest few per mode to the sample, so the human can open them, and say that the sample grew and why.

Agent suggestions are not ground truth. The human accepts or dismisses each one. Favor recall over precision. Dismissing a false positive is quick. Missing a real instance is costly. Flag modes where policy may require the flagged behavior, and report unreviewed matches as unconfirmed counts, because they do not measure how common a failure is.

## Breadth mode: propose new samples

After the human reviews a batch:
1. **Cover gaps**: sample from unreviewed clusters, topics, or feature regions.
2. **Random exploration**: always include a few random picks.
3. Push new samples to `POST /api/samples`. The app shows a banner.
4. Tell the human what was added and why.

## Encourage re-review

Do not treat review as one pass. The reviewer's criteria shift as they see more data (criteria drift).

The agent handles part of this automatically. Background subagents scan already-reviewed records and push suggestions. But the agent cannot catch everything. After the reviewer has gone through a batch and found new modes, explicitly encourage them to re-read earlier records. Say something like: "You've found 3 new failure modes since you reviewed records 0 through 5. Worth a second pass. You will likely spot things you missed the first time."

## Report and converge

Periodically:
- Report the failure mode taxonomy with confirmed and suggested counts.
- Report coverage: records reviewed, clusters or dimensions covered, what remains.
- Track convergence: are new records mostly repeats of known modes, or are they revealing new ones?
- When the rate of new discoveries drops, suggest stopping or narrowing focus.

## Key principles

1. **The human notices, the agent organizes.** Free-text notes become a structured taxonomy. Do not make the human categorize.
2. **The agent proposes coverage.** Propose new samples based on what has been found and what is missing.
3. **Live feedback loop.** Monitor and react as annotations come in.
4. **Include random samples.** Always include random picks alongside cluster-based ones so you do not miss what the clustering did not capture.
5. **Multiple passes.** The reviewer's criteria shift over time. Encourage re-review of earlier records.
6. **Agent suggests, human confirms.** Suggestions are visually distinct and require accept or dismiss. Favor recall over precision.
7. **Breadth then depth, repeat.** Review broadly to find modes, examine each mode in depth, then go broad again.
