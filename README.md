# Bangla tokenizer behaviour under Qwen3-4B

A short measurement prompted by Shi, Hauer, Basil, Zhang and Kondrak,
[*Word Surprisal Correlates with Sentential Contradiction in LLMs*](https://aclanthology.org/2026.eacl-long.211/)
(EACL 2026).

Section 4 of that paper introduces a token-to-word decoding algorithm that
reconstructs word probabilities from subword token probabilities. It partitions
the token space into beginning-of-word tokens (`S_bow`) and the rest
(`S_mid`), normalises over `S_bow` at the first decoding step and over `S_mid`
afterwards, and synthesises an end-of-word event by reassigning the probability
mass sitting on `S_bow`. The search is a constrained beam of width 10 and
depth 10. Footnote 2 notes that the partition depends on the tokenizer and the
language.

This repository measures what that partition looks like for Bangla. Tokenizer
only — no model weights, no GPU.

## Method

300 sentences each from Bangla and English Wikipedia (`wikimedia/wikipedia`,
configs `20231101.bn` and `20231101.en`), sentence-split on the danda and the
period respectively. Each whitespace-delimited word is encoded with a leading
space so it is treated as word-initial, and its subword token count recorded.
Coverage is measured over the tokenized Bangla corpus rather than by scanning
the vocabulary, for the reason given below.

The two sentence sets are comparable prose, not parallel text.

## Results

### Fertility

| | Bangla | English |
|---|---|---|
| words analysed | 4,953 | 6,857 |
| mean tokens/word | 7.35 | 1.30 |
| median tokens/word | 7 | 1 |
| max tokens/word | 29 | 11 |
| words needing > 10 tokens | 14.6% | 0.0% |

Fertility ratio: **5.67x**.

### Coverage

| | |
|---|---|
| vocabulary size | 151,669 |
| distinct entries used by Bangla text | 188 |
| — decoding to well-formed text | 130 |
| — holding a partial UTF-8 sequence | 58 |
| token occurrences | 36,198 |
| — that are partial sequences | 25,280 (**69.8%**) |

Nine of the 39 basic letters have no single-token representation:
ঊ ঋ ঌ ঐ ঘ ঝ ঞ ঠ ঢ.

Four vocabulary entries consist solely of a dependent vowel sign or hasanta
(া, ্, ে, ি), which cannot occur alone in Bangla orthography.

### Segmentation granularity

The merges are byte-level rather than character-level. A Bengali codepoint
occupies three bytes in UTF-8, and merges frequently stop between those bytes,
so individual tokens decode to U+FFFD rather than to text. `বিশ্ববিদ্যালয়`
segments into **16 tokens**, of which two (ব, য) decode to a well-formed
character on their own.

A note for anyone repeating this: scanning the vocabulary for entries that
*decode* to Bangla script finds only 47 of 151,669, but that figure is an
artefact. A token holding a partial UTF-8 sequence contains no Bangla codepoint
once decoded, so it is invisible to such a test despite being used constantly.
Hence the corpus-based count above.

## What this does and does not show

Appendix C of the paper states that when the target word is known, beam search
is unnecessary and the estimate reduces to a single forward pass. The NLI
experiments evaluate surprisal over known hypothesis words, so **depth 10 does
not constrain the paper's reported results.**

What these numbers bear on is narrower:

- The open-vocabulary retrieval the algorithm also supports would not reach the
  14.6% of Bangla words exceeding depth 10.
- `S_mid` membership carries a different meaning. In English it indicates a
  word-internal position, which is what the end-of-word assumption reasons
  about. In Bangla it indicates a mid-*character* position roughly 70% of the
  time. A well-trained model should place near-zero mass on `S_bow` at such a
  position, since no word can begin mid-codepoint, so the injection should
  correctly decline to terminate there. Whether the partition still does the
  work it was designed for across seven such steps per word is an open
  question, not a demonstrated failure.

Neither point is tested here. Establishing either would mean running the
algorithm itself over Bangla text against a reference, which this repository
does not do.

## Running it

```
pip install -r requirements.txt
jupyter notebook tokenizer_check.ipynb
```

Raw console output from the run reported above is in `results.txt`.
