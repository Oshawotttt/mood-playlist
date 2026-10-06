# DAP Proposal Framework: Emotion-Based Song Matching

> The "trunk" draft of the proposal. Content is agreed here first, then copied into `DAP_Project_Proposal_Template.docx`.
> Sections follow the template (1–9). The EDA plan also maps to `DAP_IEDA_Template.ipynb`.
> Sources: the idea comes from [project-idea.md](project-idea.md) (where we iterate). Dataset details and the full EDA plan are in [dataset_review.md](dataset_review.md).
> Items marked `[TODO]` or `[DECIDE]` still need a team decision.

---

## 1. Team

| No. | Name | Year of study | Course / degree | Previous experience | Commitments (Winter + Term 2) | Individual learning objective |
|---|---|---|---|---|---|---|
| 1 | `[TODO]` | | | | | |
| 2 | `[TODO]` | | | | | |
| 3 | `[TODO]` | | | | | |

**[TODO]** Also note hours per week each person can give over winter. We'll front-load the data work there.

## 2. Group Learning Objectives

`[DECIDE]` Draft, pick 2–4:
1. Complete an end-to-end ML workflow: data collection, EDA, labelling, training, evaluation, demo.
2. Learn transfer learning on two kinds of data: text (transformer) and audio (CNN on mel-spectrograms).
3. Build our own evaluation set and measure retrieval quality with standard metrics (precision@k).
4. Deploy a working prototype that takes a prompt and returns matching songs with an explanation.

## 3. Mentor Preference

| Preference | Mentor names | Reason / relevant expertise |
|---|---|---|
| 1 | `[TODO]` | Ideally NLP or recommender systems |
| 2 | `[TODO]` | Ideally audio / music information retrieval |
| 3 | `[TODO]` | |

---

## 4. Project Idea

### Project title
`[DECIDE]` **MoodMix: songs that match how you feel.** (Placeholder name.)

### 4.1 Problem statement and motivation
- Music apps let you pick playlists by genre, activity or a few fixed moods ("chill", "happy"). People describe how they feel in their own words, e.g. *"I'm going through a breakup"* or *"nostalgic but hopeful"*, and there's no good way to turn that into songs.
- Existing recommenders (e.g. Spotify DJ) are black boxes: they don't say *why* a song was picked.
- Spotify stopped giving new apps its audio features and recommendations in Nov 2024, so students can't simply build on Spotify's numbers. The mood of a song has to be worked out from open data: what people say about the song (tags, descriptions, lyrics) and the audio itself.
- The problem can be investigated with data: there are public datasets of songs with tags, audio, and human emotion ratings, and of everyday text labelled with emotions.

### 4.2 Proposed approach
A **dual-encoder** system. Both the prompt and each song are turned into a vector of scores over the **same named emotion dimensions** (e.g. sadness, nostalgia, tension, joy), then compared with cosine similarity.

```
User prompt ──► Prompt Encoder ──► prompt vector ──┐
                                                    ▼
                                        Shared Emotion Space ──► Cosine Similarity Search ──► Ranked songs
                                                    ▲
Song (description / audio) ──► Song Encoder ──► song vector ──┘
```

- **Prompt Encoder:** pretrained transformer (frozen) + small trained head → N emotion scores.
- **Song Encoder:**
  - *text branch*: transformer + head, reading tags, descriptions and lyrics
  - *audio branch*: pretrained CNN on mel-spectrograms + head. Covers songs with no descriptions
  - *fusion*: weighted average of the two (audio only when there's no text)
- **Why named dimensions:** the two sides line up by design, so no prompt–song paired data is needed, and every match can be explained ("matched on sadness + nostalgia").
- **Labels:** Jev (System 1 model) labels songs and prompts using our written dimension definitions. We check Jev against human-labelled data before training on its labels.

**Workflow (stages):**
| Stage | What | Output |
|---|---|---|
| 0 | Pick the emotion dimensions from existing taxonomies (GEMS, valence/arousal, GoEmotions) and tag EDA | 10–20 named dimensions with written definitions |
| 1 | Song text branch | `text_emotion_vector` per song |
| 2 | Prompt encoder | prompt vector |
| 3 | Align and validate: retrieval test, calibration, text–audio consistency | precision@k on a test set |
| 4 | Song audio branch + fusion | `audio_emotion_vector`; audio vs text vs combined comparison |

**Final output:** a demo where a user types a prompt and gets a ranked list of songs, each with the emotions it matched on. `[DECIDE]` Optionally play the list through Spotify (playback control only).

**Stretch / optional:** a learned joint embedding space (contrastive learning, e.g. CLAP) as a baseline or to discover dimensions; training the audio CNN from scratch.

### 4.3 Key research questions
`[DECIDE]` Pick 2–3:
1. Can a free-text prompt be mapped reliably onto a small set of named emotion dimensions, measured against human-labelled data?
2. Does combining what people say about a song (text) with the audio itself match prompts better than either alone?
3. Can an interpretable named-dimension space match songs about as well as an off-the-shelf learned embedding (CLAP)?

### 4.4 Intended users and expected value
| User | How they benefit |
|---|---|
| Casual listeners | Songs for how they feel right now, described in their own words, without curating a playlist |
| People working through an emotion (breakup, stress, nostalgia) | Music that fits the feeling, with a reason given for each pick |
| Hosts, cafés, small events | Background music with a consistent mood from a one-line description |
| Content creators | Finding songs with a given mood for videos (CC music is also legal to reuse) |

---

## 5. Machine Learning Techniques Required

- **NLP:** pretrained sentence-embedding / BERT-style transformers for prompts and song descriptions.
- **Audio analysis:** mel-spectrograms, pretrained CNN backbones (e.g. Essentia models trained on MTG-Jamendo).
- **Transfer learning:** frozen backbones with small trained MLP heads.
- **Multi-label classification / regression:** predicting N emotion scores (0–1) per input.
- **Weak supervision:** labels generated by an LLM (Jev), checked against human labels.
- **Retrieval:** cosine similarity search over vectors, ranked results.
- **Late fusion:** combining text and audio predictions, with weights fixed or tuned.
- **Evaluation:** precision@k / recall@k, calibration checks, per-dimension accuracy, small listening study.
- *Optional:* contrastive learning (joint text–audio embeddings), clustering for dimension discovery.

## 6. Research Done

| No. | Source | What it's about | How it informs the project |
|---|---|---|---|
| 1 | GEMS (Geneva Emotional Music Scale), Zentner et al. 2008 `[TODO link]` | 9 emotions specific to music | Starting point for our dimension list (Stage 0) |
| 2 | [GoEmotions paper (ACL 2020)](https://arxiv.org/abs/2005.00547) | 27 emotions in ~58k Reddit comments | Emotion vocabulary of everyday text; training data for the Prompt Encoder |
| 3 | [MuSe (Akiki & Burghardt)](http://ceur-ws.org/Vol-2723/short26.pdf) | Songs found on Last.fm by mood tag, scored with a word lexicon | Song list and mood vocabulary; also shows the limits of lexicon-based scores |
| 4 | Music4All, Santana et al. 2020 `[TODO link]` | ~109k songs with audio clips, lyrics, tags | Catalogue with text and audio for the same songs |
| 5 | [MTG-Jamendo](https://github.com/MTG/mtg-jamendo-dataset) | CC music with mood/theme tags | Training data for the audio branch |
| 6 | [DEAM](https://cvml.unige.ch/databases/DEAM/) | Human valence/arousal ratings for ~1.8k songs | Human ground truth for checking our emotion scores |
| 7 | [Song Describer Dataset](https://github.com/mulab-mir/song-describer-dataset) | Human-written captions for 706 tracks | Ready-made retrieval test set |
| 8 | [CLAP (LAION)](https://github.com/LAION-AI/CLAP) | Pretrained joint text–audio embeddings | Baseline to compare our interpretable model against |
| 9 | [Essentia models](https://essentia.upf.edu/models.html) | Pretrained audio models incl. mood/theme | Audio-branch backbone or baseline |
| 10 | [Spotify Web API changes (Nov 2024)](https://community.spotify.com/t5/Spotify-for-Developers/Web-API-Get-Track-s-Audio-Features-403-error/m-p/6778841/highlight/true) · [Feb 2026 update](https://developer.spotify.com/blog/2026-02-06-update-on-developer-access-and-platform-security) | New apps lose audio features and recommendations; Dev Mode limits | Why we use open datasets and only use Spotify for playback |
| 11 | [Spotify DJ](https://support.spotify.com/bs/article/dj/) | The existing product | Motivation: no free-text mood input, no explanations |

`[TODO]` Fill in the missing links. The template table has 4 rows; pick the most relevant and add rows.

## 7. Datasets / Data Sources

Full review, including rejected sources, and the datasets grouped by model and ranked: [dataset_review.md](dataset_review.md#which-datasets-we-use-by-model).
We build the two text models first (song text branch, Prompt Encoder). The audio datasets (MTG-Jamendo, DEAM, Song Describer audio, PMEmo clips) are planned for after that.

| No. | Dataset / source | Link / access | Key information | Planned usage |
|---|---|---|---|---|
| 1 | **MuSe** | [Kaggle](https://www.kaggle.com/datasets/cakiki/muse-the-musical-sentiment-dataset), CC BY 4.0 | 90k songs, one or more seed mood tags each (276 distinct), lexicon-based valence/arousal/dominance, Spotify IDs for 68%. No full tag list | Stage 0 vocabulary; song list and weak labels for the text branch (Stage 1) |
| 2 | **Music4All** ⚠️ *tentative* | [Request form](https://sites.google.com/view/contact4music4all), research use | ~109k songs, 30s audio clips, lyrics, tags, features | Both branches on the same songs (Stages 1, 4); fusion test |
| 3 | **GoEmotions** | [GitHub](https://github.com/google-research/google-research/tree/master/goemotions) / HuggingFace | ~58k Reddit comments, 27 emotions + neutral | Stage 0 vocabulary; Prompt Encoder training and testing (Stage 2) |
| 4 | **EmpatheticDialogues** | [GitHub](https://github.com/facebookresearch/EmpatheticDialogues) / HuggingFace, CC BY-NC 4.0 | ~25k first-person emotional situations (median 16 words), one of 32 emotions each | Prompt Encoder training and testing (Stage 2), alongside GoEmotions; pool for Stage 3 test prompts |
| 5 | **MTG-Jamendo (mood/theme)** *planned, after the first two models* | [GitHub](https://github.com/MTG/mtg-jamendo-dataset), CC per track | ~18.5k tracks with 59 mood/theme tags; full audio; human re-labelled test subset | Audio-branch training (Stage 4), once the text branch and Prompt Encoder work |
| 6 | **DEAM** | [Website](https://cvml.unige.ch/databases/DEAM/), CC | ~1.8k songs, human valence/arousal (whole song and per second), audio | Human ground truth; intensity for the audio branch |
| 7 | **Song Describer** | [GitHub](https://github.com/mulab-mir/song-describer-dataset), CC BY-SA 4.0 | 1,106 human captions for 706 MTG-Jamendo tracks, audio | Stage 3 retrieval test; text–audio consistency check |
| 8 | **PMEmo** | [GitHub](https://github.com/HuiZhangDB/PMEmo), research | 794 chart pop songs, human valence/arousal, chorus clips | Checking the audio branch on mainstream songs (domain shift) |
| 9 | **Deezer / iTunes previews** | Free APIs, no key (tested) | 30s previews of mainstream songs | Running the audio branch on recent mainstream songs |

**Considered and rejected:** Spotify Artist Streaming Analytics 2020–2025 (Kaggle). It is **fully synthetic** (generated by the uploader's code), so no real patterns can be learned from it.
**For the baseline:** Spotify feature tables (Kaggle 114k / 1.2M), plus the ReccoBeats API for songs after 2022, give Spotify-style valence and energy per song. We use them for a simple valence × energy baseline that our emotion mapping must beat (M3), and for the MuSe vs Spotify valence check in the EDA. They are not used to train anything.
**Optional:** MusicCaps; Spotify Million Playlist Dataset (playlist titles as prompt–song pairs; no longer downloadable from AIcrowd).

## 8. Initial EDA / Preliminary Exploration

Full plan: [dataset_review.md §5](dataset_review.md#5-eda-plan). Summary:
- **Template Q1–Q8**, Phase 1 first: EmpatheticDialogues (prompt model), MuSe and PMEmo (song text model), then GoEmotions and Music4All. MTG-Jamendo and DEAM come after the first two models. Covers size, feature dictionary, missing values, duplicates and outliers, distributions, valence × arousal quadrants, MuSe vs Spotify valence.
- **Tag vocabulary (Stage 0):** tag frequencies, removing non-emotion tags, grouping synonyms, mapping onto GEMS / GoEmotions, finding themes the taxonomies miss.
- **Label quality:** MTG-Jamendo uploader tags vs the human re-labelled subset; DEAM annotator spread; GoEmotions label imbalance; Jev vs human labels.
- **Text:** how far training text (comments, tags, captions) is from what users type; vocabulary overlap between prompt-side and song-side text.
- **Audio:** sample rate, duration, loudness, zero-crossing rate, spectral centroid, signal-to-noise, clipping; indie CC vs mainstream audio (domain shift).
- **Coverage:** match rates between datasets and the final catalogue size.

**Findings so far:**
- No dataset has both mainstream songs and full-length audio. Music4All (30s clips) and Deezer/iTunes previews are the closest.
- MuSe's valence/arousal come from a word lexicon applied to tags, not from people rating the music.
- One candidate dataset turned out to be synthetic and was dropped.
- Spotify's API no longer gives new apps audio features. ReccoBeats returns Spotify-style valence/energy by track ID (tested 2026-10-05), which is enough for the comparison baseline. Deezer/iTunes give 30s previews (tested).

`[TODO]` Link the IEDA notebook once started.

## 9. Project Milestones

> Dates are placeholders. The programme runs **Winter + Term 2**. **[TODO]** Fill in the real DAP deadlines.

| Milestone | Main tasks | Success criterion / deliverable | Target |
|---|---|---|---|
| **M1: Data, EDA and dimensions** (Stage 0) | Download and clean core datasets, request Music4All, run the EDA, fix the dimension list and definitions, check Jev against human labels | Finished IEDA notebook. 10–20 named dimensions with definitions. Cleaned catalogue with measured match rates. Jev agreement report | `[TODO]` ~early Nov 2026 |
| **M2: Text encoders** (Stages 1–2) | Label songs and prompts with Jev, train the text branch and the Prompt Encoder; no-training lexicon baseline | Emotion vectors for every song with text. Prompt Encoder beats the lexicon baseline on held-out human labels | `[TODO]` ~Dec 2026 |
| **M3: Matching and validation** (Stage 3) | Build the test set (50–100 prompts + Song Describer captions), cosine search, calibration and consistency checks | precision@10 on the test set beats a keyword / valence-energy baseline | `[TODO]` ~Feb 2027 |
| **M4: Audio, fusion and prototype** (Stage 4) | Audio branch on spectrograms, fusion, audio vs text vs combined experiment, demo, small listening study | Working demo (prompt → ranked, explained songs). Comparison results. Final report and presentation | `[TODO]` ~Mar/Apr 2027 |

### Risks and mitigations
Full list with explanations: [project-idea.md](project-idea.md#risks-and-caveats).

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Jev's labels are biased or inconsistent, and the encoders learn the errors | Medium | High | Check Jev against human labels first; keep a human-labelled test set Jev never touched |
| Audio branch trained on indie CC music doesn't transfer to mainstream songs | High | Medium | Test on PMEmo and a hand-labelled mainstream sample; compare audio statistics in EDA |
| Noisy or circular labels (MTG-Jamendo uploader tags; MuSe lexicon scores) | High | Medium | Missing tag = unknown; use the human re-labelled subset and DEAM/PMEmo for checks |
| Cosine similarity ignores intensity ("a bit down" = "devastated") | Medium | Medium | Look at it in Stage 3; consider re-ranking by vector length |
| Music4All access not granted | Medium | Medium | Fall back to MuSe tags + MTG-Jamendo audio + Deezer/iTunes previews |
| No legal full-length mainstream audio | Certain | Medium | Use 30s previews/clips for analysis; Spotify playback control (Premium, ≤5 users) or CC audio for the demo |
| Spotify terms restrict ML training on its content | Medium | High | Train only on open datasets; Spotify used only for playback |
| "Matches my mood" is subjective, so hard to evaluate | High | Medium | Two test sets (team prompts + Song Describer), human-labelled data, small listening study (n ≥ 10) |
| Audio stage needs heavy downloads and GPU time | Medium | Low | Frozen pretrained backbones, a sample of the catalogue, Colab/Kaggle GPUs |
| Team availability over exams | Medium | Medium | Front-load data work in winter; buffer weeks before M4 |

---

## Open questions
1. How many dimensions, and do themes/situations (heartbreak, party, workout) count as dimensions?
2. Which catalogue do users get recommendations from: Music4All, MTG-Jamendo or MuSe?
3. Fusion weights: fixed (e.g. 0.6 audio / 0.4 text) or tuned on the Stage 3 test set?
4. Where do labels come from beyond Jev and the datasets above?
5. Who requests Music4All access, and who reads Spotify's developer terms?
6. Evaluation: offline metrics only, or a listening study too?

