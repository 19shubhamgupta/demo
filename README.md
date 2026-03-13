# Task 1 — What happens when I tokenize?

Model used: **BPE**

Mantra:

> ॐ भूर्भुवः स्व: तत्सवितुर्वरेण्यं भर्गो देवस्य धीमहि धियो यो नः प्रचोदयात् ॥

---

## Setup + training

Commands I ran:

```powershell
# (venv already activated)
New-Item -ItemType Directory -Force artifacts, data

# small corpus
'hello world
नमस्ते दुनिया
hello नमस्ते world' | Set-Content -Encoding utf8 data\corpus.txt

# train BPE
abctokz train --corpus data\corpus.txt --model bpe --vocab-size 200 --output artifacts\task1_bpe
```

Artifacts created:
- `artifacts/task1_bpe/vocab.json`
- `artifacts/task1_bpe/merges.txt`
- `artifacts/task1_bpe/manifest.json`

---

## Encode output (tokens + IDs)

Command:

```powershell
abctokz encode --model artifacts\task1_bpe --input data\mantra.txt
```

```text
Encoding: ॐ भूर्भुवः स्व:तत्सवितुर्वरेण्यं भर्गो देवस्य धीमहि धियो यो...     
┌─────┬───────┬────┐
│ Pos │ Token │ ID │
├─────┼───────┼────┤
│   0 │ ॐ    │  0 │
│   1 │ ##    │  0 │
│   2 │ ##भ   │  0 │
│   3 │ ##ू   │  0 │
│   4 │ ##र   │  0 │
│   5 │ ##्   │ 16 │
│   6 │ ##भ   │  0 │
│   7 │ ##ु   │  0 │
│   8 │ ##व   │  0 │
│   9 │ ##ः   │  0 │
│  10 │ ##    │  0 │
│  11 │ ##स   │ 14 │
│  12 │ ##्   │ 16 │
│  13 │ ##व   │  0 │
│  14 │ ##:   │  0 │
│  15 │ ##    │  0 │
│  16 │ ##त   │  8 │
│  17 │ ##त   │  8 │
│  18 │ ##्   │ 16 │
│  19 │ ##स   │ 14 │
│  20 │ ##व   │  0 │
│  21 │ ##ि   │  0 │
│  22 │ ##त   │  8 │
│  23 │ ##ु   │  0 │
│  24 │ ##र   │  0 │
│  25 │ ##्   │ 16 │
│  26 │ ##व   │  0 │
│  27 │ ##र   │  0 │
│  28 │ ##े   │ 15 │
│  29 │ ##ण   │  0 │
│  30 │ ##्   │ 16 │
│  31 │ ##य   │  0 │
│  32 │ ##ं   │  0 │
│  33 │ ##    │  0 │
│  34 │ ##भ   │  0 │
│  35 │ ##र   │  0 │
│  36 │ ##्   │ 16 │
│  37 │ ##ग   │  0 │
│  38 │ ##ो   │  0 │
│  39 │ ##    │  0 │
│  40 │ ##द   │  0 │
│  41 │ ##े   │ 15 │
│  42 │ ##व   │  0 │
│  43 │ ##स   │ 14 │
│  44 │ ##्   │ 16 │
│  45 │ ##य   │  0 │
│  46 │ ##    │  0 │
│  47 │ ##ध   │  0 │
│  48 │ ##ी   │  0 │
│  49 │ ##म   │ 10 │
│  50 │ ##ह   │  0 │
│  51 │ ##ि   │  0 │
│  52 │ ##    │  0 │
│  53 │ ##ध   │  0 │
│  54 │ ##ि   │  0 │
│  55 │ ##य   │  0 │
│  56 │ ##ो   │  0 │
│  57 │ ##    │  0 │
│  58 │ ##य   │  0 │
│  59 │ ##ो   │  0 │
│  60 │ ##    │  0 │
│  61 │ ##न   │  0 │
│  62 │ ##ः   │  0 │
│  63 │ ##    │  0 │
│  64 │ ##प   │  0 │
│  65 │ ##्   │ 16 │
│  66 │ ##र   │  0 │
│  67 │ ##च   │  0 │
│  68 │ ##ो   │  0 │
│  69 │ ##द   │  0 │
│  70 │ ##य   │  0 │
│  71 │ ##ा   │  0 │
│  72 │ ##त   │  8 │
│  73 │ ##्   │ 16 │
│  74 │ ##    │  0 │
│  75 │ ##॥   │  0 │
└─────┴───────┴────┘

```



---

## Trace of what happened from `encode()` → IDs

The encode pipeline in this repo is basically:

1. normalize (string → string)
2. pre-tokenize (string → list of pre-tokens)
3. model tokenize (each pre-token → subword pieces + IDs)
4. optional post-processing (e.g., BOS/EOS)

Then `decode()` does the inverse: IDs → token strings → text.

### Which files/classes were involved at each stage?

- Orchestrator: `src/abctokz/tokenizer.py`
  - class: `AugenblickTokenizer`
  - methods: `encode()` and `decode()`
- Normalizer stage:
  - factory: `src/abctokz/normalizers/__init__.py` → `build_normalizer(...)`
  - components: `DevanagariNormalizer`, `WhitespaceNormalizer`, chained via `SequenceNormalizer`
- Pre-tokenizer stage:
  - factory: `src/abctokz/pretokenizers/__init__.py` → `build_pretokenizer(...)`
  - component: `DevanagariAwarePreTokenizer` (inside `SequencePreTokenizer`)
- Model stage:
  - `src/abctokz/models/bpe.py` → `BPEModel.tokenize(...)`
  - merges/vocab loaded from `artifacts/task1_bpe/merges.txt` and `artifacts/task1_bpe/vocab.json`
- Decode stage:
  - `src/abctokz/tokenizer.py` → `decode()` (IDs → token strings)
  - `src/abctokz/decoders/subword_decoder.py` → `SubwordDecoder.decode(...)` (join pieces)

### What did the normalizer do to the string?

See section (3) for RAW vs NORMALIZED output.

In this config (from `bpe_multilingual`), normalization is a sequence:
- Devanagari-safe NFC normalization + exotic-space normalization
- then whitespace strip/collapse

Code:
- preset wiring: `src/abctokz/config/defaults.py`
- construction: `src/abctokz/normalizers/__init__.py`
- behavior: `src/abctokz/normalizers/devanagari.py` and `src/abctokz/normalizers/whitespace.py`

### What did the pre-tokenizer do after normalization?

See section (4) for the PRETOKENS list.

The Devanagari-aware pre-tokenizer:
- splits on whitespace
- can split inside a token when script changes (Devanagari ↔ Latin)
- keeps grapheme clusters intact (matras/halant stay attached)

Code:
- construction: `src/abctokz/pretokenizers/__init__.py`
- behavior: `src/abctokz/pretokenizers/devanagari_aware.py`

### How did the model turn pre-tokens into subword pieces and IDs?

`AugenblickTokenizer.encode()` loops over `pre_tokens` and calls `self._model.tokenize(pre_tok)`.

For BPE (`src/abctokz/models/bpe.py`):
- initialize pieces at character level (`_init_pieces`)
  - first char stays plain
  - later chars get `##` continuation prefix
- repeatedly apply the best-ranked merge pair until nothing matches (`_apply_merges`)
- map final piece strings to IDs using the vocab

Evidence artifacts:
- learned merge rules: `artifacts/task1_bpe/merges.txt`
- token→ID mapping: `artifacts/task1_bpe/vocab.json`

### How were pieces turned back into a string during `decode()`?

In `src/abctokz/tokenizer.py`, `decode()`:
- builds an inverse vocab (ID→token)
- optionally drops special tokens
- calls the decoder

`SubwordDecoder` (`src/abctokz/decoders/subword_decoder.py`) joins BPE pieces like this:
- token starts with `##` → strip `##` and glue to previous token
- otherwise → start a new word (insert a space before it, except at the beginning)

---

## 3) What the normalizer did

Command used to inspect it:

```powershell
python -c "from abctokz.config.defaults import bpe_multilingual; from abctokz.normalizers import build_normalizer; from abctokz.pretokenizers import build_pretokenizer; text=open('data/mantra.txt','r',encoding='utf-8').read().strip(); cfg=bpe_multilingual(vocab_size=200); n=build_normalizer(cfg.normalizer); p=build_pretokenizer(cfg.pretokenizer); norm=n.normalize(text); print('RAW:', text); print('NORMALIZED:', norm); print('PRETOKENS:', p.pre_tokenize(norm))"
```

Raw:
```text
ॐ भूर्भुवः स्व: तत्सवितुर्वरेण्यं भर्गो देवस्य धीमहि धियो यो नः प्रचोदयात् ॥
```

Normalized:
```text
 ॐ भूर्भुवः स्व: तत्सवितुर्वरेण्यं भर्गो देवस्य धीमहि धियो यो नः प्रचोदयात् ॥        'यो', 'नः', 'प्
```

---

## 4) What the pre-tokenizer did

Pre-tokens:
```text
['ॐ', 'भूर्भुवः', 'स्व:', 'तत्सवितुर्वरेण्यं', 'भर्गो', 'देवस्य', 'धीमहि', 'धियो', 'यो', 'नः', 'प्रचोदयात्', '॥']
```

Notes (what splits happened and why):
- 

---

## 5) How BPE turned pre-tokens into subword pieces

High-level description (short):
- Start with character-level pieces.
- Non-initial pieces use the `##` continuation prefix.
- Apply merge rules from `artifacts/task1_bpe/merges.txt` (ranked order).
- Map final pieces to IDs using `artifacts/task1_bpe/vocab.json`.

It happens in code:
- `src/abctokz/models/bpe.py`

---

## 6) How decode() reconstructs text

What decode does:
- IDs are mapped back to token strings using the inverse vocab.
- The decoder joins pieces:
  - if a piece starts with `##`, it’s glued to the previous piece (prefix removed)
  - otherwise it starts a new word (space inserted)

Where it happens in code:
- `src/abctokz/tokenizer.py` (`decode` builds token strings)
- `src/abctokz/decoders/subword_decoder.py` (joins `##` pieces)


---

## 7) Pipeline map (one paragraph)

When I call `encode(text)`, the tokenizer runs:
1) normalizer → 2) pre-tokenizer → 3) BPE model tokenization → 4) optional post-processing.
Then the `Encoding` object contains the final `tokens` and `ids`.

