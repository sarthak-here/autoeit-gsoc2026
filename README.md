# AutoEIT — Audio Transcription Pipeline
### GSoC 2026 | HumanAI Foundation

Evaluation test submission for the project **"Audio-to-text transcription for second/additional language learner data"** under the AutoEIT project.

**Applicant:** Sarthak Sharma  
**GitHub:** [sarthak-here](https://github.com/sarthak-here)  
**Email:** sarthak909999@gmail.com

---

## Project Overview

The Elicited Imitation Task (EIT) is a language proficiency assessment where learners listen to spoken sentences and repeat them verbatim. This pipeline automates the transcription of learner responses from audio recordings into text, enabling large-scale analysis without manual transcription.

The core challenge is that standard ASR tools struggle with non-native speech — learners produce phonological errors, disfluencies, incomplete repetitions, and L1 transfer effects that native-speech models are not trained to handle.

---

## Approach

### 1. Audio Preprocessing
- Resampled all audio files to **16kHz mono** (required by Whisper)
- Applied **non-stationary noise reduction** via `noisereduce` — adapts to variable background noise across the recording
- **Volume normalization** to handle quiet speakers without distorting louder ones

### 2. Transcription — Whisper large-v3
Selected `large-v3` as the transcription model for maximum accuracy on non-native speech. Key settings tuned for learner data:

| Parameter | Value | Reason |
|---|---|---|
| `language` | `es` | Force Spanish to prevent confusion from English instructions |
| `condition_on_previous_text` | `False` | Prevents hallucination on fragmented learner responses |
| `beam_size` | `5` | Better accuracy at minor speed cost |
| `best_of` | `5` | Samples 5 candidates, picks the best |
| `temperature` | `0.0` | Deterministic output |
| `word_timestamps` | `True` | Enables segment-level alignment |

> **Note on `initial_prompt`:** Initially tested with a prompt instructing Whisper to preserve disfluencies. This caused the model to reproduce the prompt text verbatim in the transcription. Removed — the model handles learner disfluencies adequately without it.

### 3. Segment Alignment
Each audio file contains warmup sentences and English instructions before the 30-item EIT task. The pipeline:
1. Identifies the EIT start point per participant (locates the "Now let's begin" instruction)
2. Applies a **12-minute offset** for participant 38012 (noted in the Excel template)
3. Uses **rapidfuzz partial string matching** to align each of the 30 transcribed segments to the corresponding stimulus sentence

Fuzzy matching was chosen over a fixed alternating-index approach because Whisper sometimes merges consecutive learner responses into a single segment, making index-based alignment unreliable.

### 4. Transcription Philosophy
- Learner grammar and vocabulary errors are **preserved exactly as spoken**
- Only clear **ASR errors** were corrected (e.g. misread proper nouns)
- Empty cells indicate genuinely undetectable responses — not system failures

---

## Dataset

4 MP3 audio files of Spanish EIT completions:

| File | Participant | Notes |
|---|---|---|
| `038010_EIT-2A.mp3` | 38010 | ~10 min |
| `038011_EIT-1A.mp3` | 38011 | ~10 min |
| `038012_EIT-2A.mp3` | 38012 | ~20 min — EIT begins at 12:00 |
| `038015_EIT-1A.mp3` | 38015 | ~10 min |

---

## Results

| Participant | Filled | Empty | Notes |
|---|---|---|---|
| 38010-2A | 30/30 | 0 | High accuracy throughout |
| 38011-1A | 20/30 | 10 | Segments 24–30 not detectable |
| 38012-2A | 16/30 | 14 | Very low proficiency learner |
| 38015-1A | 20/30 | 10 | Some segments merged or missing |

Participant 38012 represents the hardest case — a very low proficiency learner whose responses bear little resemblance to the target sentences. This is linguistically expected and reflects the limits of automated alignment on extremely non-native speech.

---

## Challenges & Limitations

**Segment merging:** Whisper sometimes merges consecutive responses into one segment when the learner pauses briefly. This affects alignment for a handful of sentences and required manual post-processing.

**Low proficiency speech:** For participant 38012, responses are phonologically distant enough from the stimulus that fuzzy matching produces low-confidence alignments.

**Future improvements:**
- Fine-tune Whisper on a small L2 Spanish EIT dataset with human-verified transcriptions
- Use forced alignment (e.g. `whisperx`) for more precise segment boundaries
- Build a confidence threshold filter to automatically flag uncertain alignments for human review

---

## Setup & Usage

```bash
# Clone the repo
git clone https://github.com/sarthak-here/autoeit-gsoc2026
cd autoeit-gsoc2026

# Install dependencies
pip install -r requirements.txt

# Recommended: run in Google Colab with T4 GPU
# Open AutoEIT_Sarthak_Sharma.ipynb
# Runtime → Change runtime type → T4 GPU
# Run all cells in order
```

---

## File Structure

```
autoeit-gsoc2026/
├── AutoEIT_Sarthak_Sharma.ipynb                     # main notebook
├── AutoEIT_Transcriptions_Sarthak_Sharma_v2.xlsx    # output transcriptions
├── requirements.txt
└── README.md
```
