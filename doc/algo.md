## Alignment algorithm and its parameters

### Step 1 - Splitting audio

A voice activity detector (at the moment this is `webrtcvad`) is used
to split the provided audio data into voice fragments.
These fragments are essentially streams of continuous speech without any longer pauses 
(e.g. sentences).

`--audio-vad-aggressiveness <AGGRESSIVENESS>` can be used to influence the length of the
resulting fragments.

### Step 2 - Preparation of original text

STT transcripts are typically provided in a normalized textual form with
- no casing,
- no punctuation and
- normalized whitespace (single spaces only).

So for being able to align STT transcripts with the original text it is necessary
to internally convert the original text into the same form.

This happens in two steps:
1. Normalization of whitespace, lower-casing all text and 
replacing some characters with spaces (e.g. dashes)
2. Removal of all characters that are not in the languages's alphabet
(see DeepSpeech model data)

Be aware: *This conversion happens on text basis and will not remove unspoken content
like markup/markdown tags or artifacts. This should be done beforehand.
Reducing the difference of spoken and original text will improve alignment quality and speed.*

In the very unlikely situation that you have to change the default behavior (of step 1),
there are some switches:

`--text-keep-dashes` will prevent substitution of dashes with spaces.

`--text-keep-ws` will keep whitespace untouched.

`--text-keep-casing` will keep character casing as provided.

### Step 3 (optional) - Generating document specific language model

If the [dependencies](lm.md) for 
individual language model generation got installed, this document-individual model will
now be generated by default.

Assuming your text document is named `original.txt`, these files will be generated:
- `original.txt.clean` - cleaned version of the original text
- `original.txt.arpa` - text file with probabilities in ARPA format
- `original.txt.lm` - binary representation of the former one
- `original.txt.trie` - prefix-tree optimized for probability lookup

`--stt-no-own-lm` deactivates creation of individual language models per document and
uses the one from model dir instead.

### Step 4 - Transcription of voice fragments through STT

After VAD splitting the resulting fragments are transcribed into textual phrases.
This transcription is done through [DeepSpeech](https://github.com/mozilla/DeepSpeech/) STT.

As this can take a longer time, all resulting phrases are - together with their 
timestamps - saved as JSON into a transcription log file 
(the `audio` parameter path with suffix `.tlog` instead of `.wav`).
Consecutive calls will look for that file and - if found - 
load it and skip the transcription phase.

`--stt-model-dir <DIR>` points DeepSpeech to the language specific model data directory.
It defaults to `models/en`. Use `bin/getmodel.sh` for preparing it.  

### Step 5 - Rough alignment

The actual text alignment is based on a recursive divide and conquer approach:

1. Construct an ordered list of of all phrases in the current interval
(at the beginning this is the list of all phrases that are to be aligned),
where long phrases close to the middle of the interval come first.
2. Iterate through the list and compute the best Smith-Waterman alignment
(see the following sub-sections) with the document's original text...
3. ...till there is a phrase whose Smith-Waterman alignment score surpasses a (low) recursion-depth 
dependent threshold (in most cases this should already be the first phrase).
4. Recursively continue with step 1 for the sub-intervals and original text ranges
to the left and right of the phrase and its aligned text range within the original text.
5. Return all phrases in order of appearance (depth-first) that were aligned with the minimum 
Smith-Waterman score on their recursion level.

This approach assumes that all phrases were spoken in the same order as they appear in the
original transcript. It has the following advantages compared to individual
global phrase matching:

- Long non-matching chunks of spoken text or the original transcript will automatically and 
cleanly get ignored.
- Short phrases (with the risk of matching more than one time per document) will automatically
get aligned to their intended locations by longer ones who "squeeze" them in.
- Smith-Waterman score thresholds can be kept lower 
(and thus better match lower quality STT transcripts), as there is a lower chance for 
  - long sequences to match at a wrong location and for 
  - shorter sequences to match at a wrong location within their shortened intervals
  (as they are getting matched later and deeper in the recursion tree).

#### Smith-Waterman candidate selection

Finding the best match of a given phrase within the original (potentially long) transcript
using vanilla Smith-Waterman is not feasible.

So this tool follows a two-phase approach where the first goal is to get a list of alignment 
candidates. As the first step the original text is virtually partitioned into windows of the 
same length as the search pattern. These windows are ordered descending by the number of 3-grams
they share with the pattern.
Best alignment candidates are now taken from the beginning of this ordered list.

`--align-max-candidates <CANDIDATES>` sets the maximum number of candidate windows
taken from the beginning of the list for further alignment.

`--align-candidate-threshold <THRESHOLD>` multiplied with the number of 3-grams of the predecessor
window it gives the minimum number of 3-grams the next candidate window has to have to also be
considered a candidate.

#### Smith-Waterman alignment

For each candidate, the best possible alignment is computed using the 
[Smith-Waterman](https://en.wikipedia.org/wiki/Smith%E2%80%93Waterman_algorithm) algorithm
within an extended interval of one window-size around the candidate window.

`--align-match-score <SCORE>` is the score per correctly matched character. Default: 100

`--align-mismatch-score <SCORE>` is the score per non-matching (exchanged) character. Default: -100

`--align-gap-score <SCORE>` is the score per character gap (removing 1 character from pattern or original). Default: -100

The overall best score for the best match is normalized to a value of about 100 maximum by dividing
it through the maximum character count of either the match or the pattern.

### Step 6 - Gap alignment

After recursive matching of fragments there are potential text leftovers between aligned original
texts.

Some examples:
- Often: Missing (and therefore unaligned) STT transcripts of word-endings (e.g. English past tense endings _-d_ and _-ed_)
on phrase endings to the left of the gap
- Seldom: Phrase beginnings or endings that were wrongly matched on unspoken (but written) text whose actual
alignments are now left unaligned in the gap
- Big unmatched chunks of text, like
  - Preface, text summaries or any other kind of meta information
  - Copyright headers/footers
  - Table of contents
- Chapter headers (if not spoken as they appear)
- Captions of figures
- Contents of tables
- Line-headers like character names in drama scripts
- Dependent of the (pre-processing) quality: OCR leftovers like
  - page headers
  - page numbers
  - reader's notes
  
The basic challenge here is to figure out, if all or some of the gap text should be used to extend 
the phrase to the left and/or to the right of the gap.

As Smith-Waterman alignment led to the current (potentially incomplete or even wrong) result,
its score cannot be used for further fine-tuning. Therefore there is a collection of
so called [test-distance metrics](metrics.md) to pick from using the `--align-similarity-algo <METRIC-ID>`
parameter.

Using the selected distance metric, the gap alignment is done by looking for the best scoring 
extension of the left and right phrases up to their maximum extension.

`--align-stretch-factor <FRACTION>` is the fraction of the text length that it could get
stretched at max.  

For many languages it is worth putting some emphasis on matching to words boundaries 
(that is white-space separated sub-sequences).

`--align-snap-factor <FACTOR>` allows to control the snappiness to word boundaries.

If the best scoring extensions should overlap, the best scoring sum of non-overlapping
(but touching) extensions will win.

### Step 7 - Selection, filtering and output

Finally the best alignment of all candidate windows is selected as the winner.
It has to survive a series of filters for getting into the result file.

For each text distance metric there are two filter parameters:

`--output-min-<METRIC-ID> <VALUE>` only keeps utterances having the provided minimum value for the
metric with id `METRIC-ID`
                              
`--output-max-<METRIC-ID> <VALUE>` only keeps utterances having the provided maximum value for the
metric with id `METRIC-ID`

For each text distance metric there's also the option to have it added to each utterance's entry:

`--output-<METRIC-ID>` adds the computed value for `<METRIC-ID>` to the utterances array-entry

Error rates and scores are provided as fractional values (typically between 0.0 = 0% and 1.0 = 100%
where numbers >1.0 are theoretically possible).
