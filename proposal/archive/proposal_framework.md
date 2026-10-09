# DAP Proposal Framework: Mood-Based Song Matching

> The "trunk" draft of the proposal. Content is agreed here first, then copied into `DAP_Project_Proposal_Template.docx`.
> Sections follow the template (1–9). The EDA is in `DAP_IEDA_Template.ipynb` (one notebook, a section per dataset); the failed Last.fm tag fetch is kept in `FAILED_edas.ipynb`.
> Sources: the idea comes from [project-idea.md](project-idea.md) (where we iterate). Dataset details and the full EDA plan are in [dataset_review.md](dataset_review.md).
> Items marked `[TODO]` or `[DECIDE]` still need a team decision.
>
> **v2 (2026-10-09):** the song model now reads **lyrics**, not Last.fm tags. Last.fm no longer returns track tags (500 of 506 sampled MuSe songs came back empty). Full v2 plan: [project-idea.md](project-idea.md#v2-plan-lyrics).

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
2. Learn transfer learning for text: a frozen pretrained sentence transformer with a small trained head, applied to two kinds of text (song lyrics and user prompts).
3. Design an interpretable label space (named mood dimensions) from data and existing vocabularies.
4. Build our own evaluation set and measure retrieval quality with standard metrics (precision@k) against a simple baseline.
5. Deploy a working prototype that takes a prompt and returns matching songs with an explanation.

## 3. Mentor Preference

| Preference | Mentor names | Reason / relevant expertise |
|---|---|---|
| 1 | `[TODO]` | Ideally NLP or recommender systems |
| 2 | `[TODO]` | Ideally music information retrieval |
| 3 | `[TODO]` | |

---

## 4. Project Idea

### Project title
`[DECIDE]` **MoodMix: songs that match how you feel.** (Placeholder name.)

### 4.1 Problem statement and motivation
- Music apps let you pick playlists by genre, activity or a few fixed moods ("chill", "happy"). People describe how they feel in their own words, e.g. *"I'm going through a breakup"* or *"nostalgic but hopeful"*, and there's no good way to turn that into songs.
- Existing recommenders (e.g. Spotify DJ) are black boxes: they don't say *why* a song was picked.
- Spotify stopped giving new apps its audio features and recommendations in Nov 2024, so students can't simply build on Spotify's numbers. The mood of a song has to be worked out from open data: its lyrics, labelled with the mood words listeners gave it.
- The problem can be investigated with data: there are public datasets of songs with listener mood words (MuSe), open lyrics (LRCLIB), and everyday first-person situations labelled with moods (EmpatheticDialogues).

### 4.2 Proposed approach
A **dual-encoder** system. Both the prompt and each song are turned into a vector of scores over the **same named mood dimensions** (e.g. sadness, nostalgia, tension, joy), then compared with cosine similarity.

```
User prompt ──────────► Prompt Encoder ──► prompt vector ──┐
                                                            ▼
                                              Shared Mood Space ──► Cosine Similarity Search ──► Ranked songs
                                                            ▲
Song (lyrics) ────────► Song Encoder ────► song vector ────┘
```

- **Both encoders** are a pretrained sentence transformer (e.g. `all-MiniLM-L6-v2`, frozen) plus a small trained head that outputs N named mood scores (0–1). Only the head is trained, so thousands of examples are enough, and the same encoder reads both prompts and lyrics.
- **Prompt Encoder:** trained on **EmpatheticDialogues** situations, with their 32 moods mapped onto our dimensions. GoEmotions is a second, larger source.
- **Song Encoder:** reads each song's **lyrics** (LRCLIB, with the Kaggle Genius dump and Music4All as fallbacks), split into stanzas and mean-pooled. It is trained on MuSe songs, labelled with the dimension of their MuSe **seed** mood word. Catalogue: English songs with lyrics. We train with the seed words in the lyrics both masked and unmasked, and compare.
- **Why named dimensions:** the two sides line up by design, so no prompt–song paired data is needed, and every match can be explained ("matched on sadness + nostalgia").
- **Labels:** MuSe seeds (song side) and EmpatheticDialogues moods (prompt side), both chosen by people. `[DECIDE]` Optionally, Jev (System 1 model) gives each situation full scores across all dimensions, fixing EmpatheticDialogues' one-label-per-situation limit.

**Workflow (stages):**
| Stage | What | Output |
|---|---|---|
| 1 | Define the dimensions (EDA): start from AllMusic moods (via MuSe seeds), AllMusic themes, GEMS and EmpatheticDialogues' 32 moods; run EDA on both sides; group synonyms; check coverage | ~10 named dimensions with written definitions and a mapping table |
| 2 | Song model: lyrics → mood vector. Zero-shot baseline: cosine between the lyric embedding and each dimension's written definition | `mood_vector` per English song with lyrics (the catalogue) |
| 3 | Prompt model: EmpatheticDialogues situations → mood vector | prompt vector; tested on held-out situations and our own prompts |
| 4 | Match and evaluate: cosine search, test set of 50–100 prompts, PMEmo human check, calibration | precision@10 on the test set |

**Final output:** a demo where a user types a prompt and gets a ranked list of songs, each with the moods it matched on. `[DECIDE]` Optionally play the list through Spotify (playback control only).

**Optional extensions (only if time):**
- **Descriptions:** run captions (Song Describer) through the same encoder and head.
- **Audio branch:** pretrained CNN on mel-spectrograms of 30s clips + head, for instrumental songs and moods that live in the sound.
- **Learned joint embedding:** clustering to discover dimensions, or CLAP as a baseline.

### 4.3 Key research questions
1. How useful can song labels and descriptions be in determining a song's relevance to a user's prompt?
2. To what extent can an ML system meet a user's subjective requirements from it?
3. Can people's own descriptions of how they feel be matched to songs through a small set of named moods, without any examples of which songs suit which prompts?

### 4.4 Intended users and expected value
| User | How they benefit |
|---|---|
| Casual listeners | Songs for how they feel right now, described in their own words, without curating a playlist |
| People working through an emotion (breakup, stress, nostalgia) | Music that fits the feeling, with a reason given for each pick |
| Hosts, cafés, small events | Background music with a consistent mood from a one-line description |
| Content creators | Finding songs with a given mood for videos |

---

## 5. Machine Learning Techniques Required

- **NLP:** pretrained sentence-embedding transformers (frozen) for prompts and song lyrics.
- **Transfer learning:** frozen backbone with small trained heads (linear / MLP).
- **Multi-label classification:** predicting N mood scores (0–1) per input, trained from single positive labels ("this dimension is high").
- **Label design:** grouping vocabularies into dimensions; using listener mood words (MuSe seeds) as labels for lyrics.
- **Retrieval:** cosine similarity search over vectors, ranked results.
- **Evaluation:** precision@k, per-dimension accuracy, song model vs a zero-shot baseline, rough check against PMEmo, small listening study.
- *Optional:* weak supervision with an LLM (Jev, also compared against the seed labels); clustering of lyric embeddings for dimension discovery; audio CNNs on mel-spectrograms; contrastive learning (CLAP).

## 6. Research Done

| No. | Source | What it's about | How it informs the project |
|---|---|---|---|
| 1 | [MuSe (Akiki & Burghardt)](http://ceur-ws.org/Vol-2723/short26.pdf) | ~90k songs found on Last.fm by 276 AllMusic mood words, scored with a word lexicon | Song catalogue, seed labels for the song model, starting vocabulary for the dimensions; also shows the limits of lexicon-based scores |
| 2 | [EmpatheticDialogues (Rashkin et al., ACL 2019)](https://arxiv.org/abs/1811.00207) | ~25k first-person situations, each with one of 32 moods | Main training/test data for the Prompt Encoder; prompt-side vocabulary; pool of realistic test prompts |
| 3 | GEMS (Geneva Emotional Music Scale), Zentner et al. 2008 `[TODO link]` | 9 emotions specific to music | Vocabulary for the dimension list (Stage 1) |
| 4 | AllMusic mood and theme lists `[TODO link]` | Editor-written mood and theme words | Source of MuSe's seeds; theme words cover situations the moods miss. Copied by hand (no scraping) |
| 5 | [LRCLIB](https://lrclib.net) | Open lyrics API, searchable by artist and title | The song model's input |
| 6 | [GoEmotions paper (ACL 2020)](https://arxiv.org/abs/2005.00547) | 27 emotions in ~58k Reddit comments | Second training source for the Prompt Encoder |
| 7 | [PMEmo](https://github.com/HuiZhangDB/PMEmo) | 794 chart pop songs with human valence/arousal ratings and lyrics | Rough human check on the song vectors |
| 8 | [sentence-transformers](https://www.sbert.net/) | Pretrained sentence embedding models | Frozen encoder backbone for both sides |
| 9 | [Spotify Web API changes (Nov 2024)](https://community.spotify.com/t5/Spotify-for-Developers/Web-API-Get-Track-s-Audio-Features-403-error/m-p/6778841/highlight/true) · [Feb 2026 update](https://developer.spotify.com/blog/2026-02-06-update-on-developer-access-and-platform-security) | New apps lose audio features and recommendations; Dev Mode limits | Why we use open data and only use Spotify for playback |
| 10 | [Spotify DJ](https://support.spotify.com/bs/article/dj/) | The existing product | Motivation: no free-text mood input, no explanations |
| 11 | [CLAP (LAION)](https://github.com/LAION-AI/CLAP) | Pretrained joint text–audio embeddings | Optional baseline against our interpretable model |

`[TODO]` Fill in the missing links. The template table has 4 rows; pick the most relevant and add rows.

## 7. Datasets / Data Sources

Full review, including rejected sources: [dataset_review.md](dataset_review.md).
The core plan uses **MuSe + lyrics** (song side) and **EmpatheticDialogues** (prompt side). Everything else is for evaluation, baselines or the optional extensions.

| No. | Dataset / source | Link / access | Key information | Planned usage |
|---|---|---|---|---|
| 1 | **MuSe** | [Kaggle](https://www.kaggle.com/datasets/cakiki/muse-the-musical-sentiment-dataset), CC BY 4.0 | 90,001 songs, one or more of 276 AllMusic seed mood words each (80% have one), lexicon-based valence/arousal/dominance, Spotify IDs for 68%. No full tag list | Stage 1 vocabulary; song catalogue and seed labels for the song model (Stage 2) |
| 2 | **LRCLIB** | [lrclib.net](https://lrclib.net), free API, no key | Lyrics by artist + title; instrumental flag | The song model's input (Stages 1, 2). Fallback: [Kaggle Genius Song Lyrics](https://www.kaggle.com/datasets/carlosgdcj/genius-song-lyrics-with-language-information) (~5M songs, scraped, research use only) |
| 3 | **EmpatheticDialogues** | [GitHub](https://github.com/facebookresearch/EmpatheticDialogues) / HuggingFace, CC BY-NC 4.0 | 24,850 first-person situations (median 16 words), one of 32 moods each, fairly balanced | Stage 1 prompt-side EDA; Prompt Encoder training and testing (Stage 3); pool for Stage 4 test prompts |
| 4 | **AllMusic mood and theme lists** | Public list pages, copied by hand | Editor-written mood and theme words | Stage 1 vocabulary only |
| 5 | **GoEmotions** | [GitHub](https://github.com/google-research/google-research/tree/master/goemotions) / HuggingFace | ~58k Reddit comments, 27 emotions + neutral | Second training source for the Prompt Encoder (Stage 3) |
| 6 | **PMEmo** | [GitHub](https://github.com/HuiZhangDB/PMEmo), research | 794 chart pop songs, human valence/arousal, lyrics, comments, chorus clips | Rough human check on the song vectors (Stage 4) |
| 7 | **Music4All** ⚠️ *access pending* | Email `contact4music4all@gmail.com` ([page](https://sites.google.com/view/contact4music4all)), research use | ~109k songs with lyrics, Last.fm tags (collected before Last.fm emptied them), 30s clips | Extra lyrics (and the audio extension). The core plan works without it |
| 8 | **Song Describer** | [GitHub](https://github.com/mulab-mir/song-describer-dataset), CC BY-SA 4.0 | 1,106 human captions for 706 tracks | Descriptions extension |

**Audio extension only:** MTG-Jamendo, DEAM, Deezer/iTunes 30s previews. Not downloaded unless we reach the audio branch.
**Considered and rejected:** Spotify Artist Streaming Analytics 2020–2025 (Kaggle). It is **fully synthetic** (generated by the uploader's code), so no real patterns can be learned from it. MusicCaps (audio only via YouTube, which is against YouTube's terms).
**Optional:** Spotify Million Playlist Dataset (playlist titles as prompt–song pairs for the learned-embedding extension; no longer downloadable from AIcrowd).

## 8. Initial EDA / Preliminary Exploration

The EDA is Stage 1: it decides the mood dimensions. A dimension only works if people **ask for it** (prompt side) *and* songs **carry it** (song side), so both datasets get the template Q1–Q8 treatment. Full plan: [dataset_review.md §5](dataset_review.md#5-eda-plan).

- **Song side (MuSe and LRCLIB lyrics), sections of `DAP_IEDA_Template.ipynb`:**
  - Q1–Q8: size, feature dictionary, missing values, duplicates, valence/arousal/dominance distributions, valence × arousal quadrants, genre distribution.
  - Songs per seed, seeds per song, seed co-occurrence.
  - Sort the 276 seeds into mood words vs sound/style words (crunchy, literate, slick). All are kept as labels; results are reported per dimension.
  - Fetch lyrics for a sample of songs: coverage per source, share of instrumental and non-English songs, lyric length against the 256 word-piece limit, distinctive words per seed group, and situations mentioned in lyrics (breakup, late night, leaving home) that the seeds miss.
- **Prompt side (EmpatheticDialogues), section of `DAP_IEDA_Template.ipynb`:**
  - Q1–Q8: size, label frequencies, situation lengths, duplicates and conflicting labels.
  - Most common situations (loss, breakups, exams, jobs) and top words per mood.
  - Which of the 32 moods are music moods, and which (guilty, ashamed, jealous…) need mapping or dropping.
- **Combining both:** a mapping table (dimension → MuSe seeds → ED moods → GEMS), songs and prompts per candidate dimension, and the final ~10 dimensions with written definitions.

**Findings so far:**
- MuSe has **no full tag list**, only the seed words and a count of emotion tags. Last.fm's `track.getTopTags` returned **no tags for 500 of 506** sampled MuSe songs (2026-10-09), so the song model reads lyrics instead.
- MuSe's valence/arousal/dominance come from a word lexicon applied to tags, not from people rating the music. **48% of songs (43,468) share their exact valence/arousal/dominance point with another song** (99.8% of songs with a single emotion tag), which confirms the authors' warning. We don't use these scores as labels.
- MuSe has no duplicate (artist, track) pairs. Genre has 811 distinct values with a long tail, mostly indie/rock/electronic, so the catalogue reflects Last.fm's userbase.
- MuSe caps each seed at 1,000 songs, but most seeds have far fewer (median 290; 91 of 276 seeds have under 100). Mood-neutral songs are missing.
- 1,863 MuSe rows share a Spotify ID with another row (the same song with a punctuation difference), so the catalogue needs de-duplicating.
- EmpatheticDialogues has 24,850 situations (24,503 unique), 32 fairly balanced moods (478–1,279 each), median 16 words, 76% in the first person: the closest public text to real prompts. About 9 of its 32 moods aren't music moods.
- One candidate dataset turned out to be synthetic and was dropped.
- **Lyrics (LRCLIB, 1,000 random MuSe songs):** 59% have lyrics and 54% have English lyrics (~49,000 songs if the rate holds), so the Genius fallback isn't needed for now. A hand-check of 50 matches found 0 wrong songs. Coverage favours popular, vocal songs (pop/rock ~87%; ambient 4%, jazz 38%, electronic 44%). 55% of songs are over the 256 word-piece limit and 14% have no stanza breaks (fixed blocks of lines for those).
- **Draft dimensions:** a first grouping of the 502 candidate terms (`data/interim/mood_clusters.csv`, classified with Claude Opus 5.5) gives 16 moods. Five (ethereal, playful, romance, dread, jaded) have songs but no ED situations, and shame has only 55 songs, so the list still needs merging to ~10.

`[TODO]` Add the link to the EDA notebook (repo URL).

## 9. Project Milestones

> Dates are placeholders. The programme runs **Winter + Term 2**. **[TODO]** Fill in the real DAP deadlines.

| Milestone | Main tasks | Success criterion / deliverable | Target |
|---|---|---|---|
| **M1: EDA and dimensions** (Stage 1) | EDA on MuSe and EmpatheticDialogues, fetch lyrics for a sample and measure coverage, group synonyms, fix the dimension list | Finished EDA notebooks. ~10 named dimensions with definitions and a mapping table. Songs and prompts counted per dimension | `[TODO]` ~early Nov 2026 |
| **M2: Song model** (Stage 2) | Fetch and cache lyrics for MuSe (overnight job), keep English songs with lyrics, train the head; zero-shot baseline | Mood vector for every song with lyrics. Trained model beats the zero-shot baseline on held-out songs | `[TODO]` ~Dec 2026 |
| **M3: Prompt model** (Stage 3) | Map ED moods onto the dimensions, train the head (optionally with GoEmotions / Jev scores) | Prompt vectors. Per-dimension accuracy on held-out ED situations and our own prompts | `[TODO]` ~Jan 2027 |
| **M4: Matching, evaluation and prototype** (Stage 4) | Test set of 50–100 prompts, cosine search, PMEmo check, calibration, demo, small listening study; extensions if time | precision@10 on the test set. Working demo (prompt → ranked, explained songs). Final report and presentation | `[TODO]` ~Mar/Apr 2027 |

### Risks and mitigations
Full list with explanations: [project-idea.md](project-idea.md#risks-and-caveats).

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Seed labels describe how listeners hear a song, partly its sound, which lyrics can't show | High | High | Report results per dimension |
| The model spots seed words in the lyrics ("lonely") instead of learning mood | Medium | Medium | Train with seed words masked and unmasked, and compare |
| MuSe's selection is biased (chosen by mood word, up to 1,000 per mood, Last.fm's indie/rock userbase) | Certain | Medium | Report genre and artist distributions in the EDA; state it as a limitation; check on PMEmo (chart pop) |
| The seed vocabulary covers sound and feel, not life situations (breakup, party, workout) | High | Medium | Add situations from lyrics, AllMusic themes and EmpatheticDialogues |
| EmpatheticDialogues has one writer-chosen label per situation, and ~9 labels aren't music moods | Certain | Medium | Train as "this dimension is high" or add Jev scores; map or drop non-music labels; report per dimension |
| **Happened (2026-10-09):** Last.fm returns no track tags. 500 of 506 sampled MuSe songs came back empty | Certain | High | Pivoted: the song model reads lyrics, with MuSe seeds kept as labels. See [project-idea.md](project-idea.md#v2-plan-lyrics) |
| Many songs have no lyrics available, and some are instrumental or not in English | Certain | Medium | Measured in the EDA: 54% of sampled songs have English lyrics on LRCLIB. Keep English songs with lyrics and state it as a limitation; add the Kaggle Genius fallback (or Music4All if granted) only if a dimension runs short of songs |
| Lyrics are copyrighted | Certain | Medium | Research use only; never republish them in the repo, report or demo |
| Wrong matches across sources by artist + title (covers, live versions, remixes) | Low | Medium | Exact match on normalised artist + title; no match counts as no lyrics. Hand-check in the EDA: 0 of 50 wrong (error rate likely under ~6%) |
| Cosine similarity ignores intensity ("a bit down" = "devastated"); vague prompts give noisy vectors | Medium | Medium | Look at score spread in Stage 4; consider re-ranking by vector length; fallback for weak prompts |
| Human-rated song data is small (PMEmo, 794 songs) | Certain | Medium | Use it for evaluation only, plus our own test set and a listening study |
| Spotify terms restrict ML training on its content | Medium | High | Train only on open data; Spotify used only for playback |
| "Matches my mood" is subjective, so hard to evaluate | High | Medium | PMEmo human ratings for the song side; small listening study (n ≥ 10) |
| Team availability over exams | Medium | Medium | Front-load data work in winter; buffer weeks before M4 |

---

## Open questions
1. How many dimensions (~10?), and which themes/situations (heartbreak, party, workout) count as dimensions?
2. Do we use Jev for multi-dimension prompt scores, or train on single labels?
3. One shared text head for songs and prompts, or two separate heads?
4. ~~LRCLIB lyrics coverage on MuSe~~ Answered by the EDA (2026-10-09): 59% have lyrics, 54% English (~49,000 songs). The Genius fallback isn't needed for now. Hand-check: 50 of 50 matches correct (error rate likely under ~6%).
5. Do we get Music4All access in time? It has lyrics and pre-2020 Last.fm tags.
6. Who reads the Spotify terms, and the terms of whichever lyrics source we choose?
7. Evaluation: offline metrics only, or a listening study too?
