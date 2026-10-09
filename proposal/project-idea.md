# Mood-Based Song Matching

> **Current plan: [v2 (lyrics)](#v2-plan-lyrics).** The song model reads a song's lyrics, not its Last.fm tags. The [v1 plan (Last.fm tags)](#v1-plan-lastfm-tags-superseded) is kept below for history. Wherever v2 doesn't mention something, v1 still applies (prompt model, dimensions, matching).

## v2 plan (lyrics)

Decided 2026-10-09.

### Why we pivoted: no Last.fm track tags

Last.fm's `track.getTopTags` (and `track.getInfo`) now return an empty tag list for almost every song. Only very popular tracks still get tags.

- **Sample:** 506 random MuSe songs, stored in `data/raw/lastfm/sample_1000.jsonl`.
- **Result:** 500 of 506 (98.8%) came back with no tags. The other 6 are big hits (e.g. Radiohead "Fake Plastic Trees", Taylor Swift "Love Story"). Each of them got exactly 10 tags.
- **Not a bug in our code:** every song was found. There were no API errors, and listener counts came back as normal. Only the tag list is empty.

v1's song model had no input left for ~99% of MuSe songs, and the tag-based EDA and PMEmo check broke with it. MuSe itself is unaffected: its song list, seeds and IDs are still there.

**Options we rejected:**
- **Artist tags** (`artist.getTopTags` still works). They describe the artist, not the song, so they're too rough to use.
- **Scraping the Last.fm website.** Likely against its terms.

### What changes from v1

| Part | v1 (tags) | v2 (lyrics) |
|---|---|---|
| Song model input | Last.fm track tags, seed hidden | Song lyrics |
| Song model labels | MuSe seeds, grouped into dimensions | Same |
| Catalogue | Songs with Last.fm tags | English songs with lyrics |
| Song-model baseline | Group mood tags by rule | Zero-shot: cosine between the lyric embedding and each dimension's written definition |
| Song-side EDA | Tag frequencies, themes in tags | Lyrics coverage, length and themes in lyrics (see below) |
| Lyrics | Optional extension | Core input |
| Prompt model, dimensions, matching | (unchanged) | (unchanged) |

### Song model (v2)

- **Data:** MuSe's ~90k songs. Lyrics fetched by artist + title:
  1. **LRCLIB** ([lrclib.net](https://lrclib.net), free API, no key) first.
  2. **Kaggle "Genius Song Lyrics" dump** as a fallback, if LRCLIB coverage is too low. It was scraped from Genius, so research use only.
  3. **Music4All** (lyrics + pre-2020 Last.fm tags), if we get access. Request by emailing `contact4music4all@gmail.com`.
- **Catalogue filter:** keep English songs with lyrics. Drop instrumental and non-English songs, report how many, and state it as a limitation. (`all-MiniLM-L6-v2` is English-only.)
- **Label:** the dimension(s) of the song's MuSe seed(s), as in v1. **All seeds are kept**, including ones that describe sound (crunchy, slick). Sound and lyrics may correlate, so we let the results show which dimensions carry over, reported per dimension.
- **Seed words in the lyrics:** unlike tags, the seed was never chosen *from* the lyrics, so the circular leakage of v1 doesn't apply. A song seeded "lonely" may still say "lonely". **Train both ways** (seed and synonyms masked, and unmasked) and compare, to see how much the model just spots keywords.
- **Long lyrics:** `all-MiniLM-L6-v2` truncates at 256 word pieces (~1–2 verses). Split into stanzas, embed each, and **mean-pool**. **Max-pool** is an ablation, because one strong chorus may matter more than the average.
- **Model:** frozen sentence transformer + small trained head → N mood scores, as in v1.
- **Baseline (no training):** zero-shot. Score each dimension by the cosine between the lyric embedding and that dimension's written definition. The trained head has to beat it.
- **Later comparison:** Jev labels the lyrics of a subset of songs directly. Measure how often Jev and the seeds agree, and train a second head on Jev labels to compare. Where they disagree shows where listener mood and lyric meaning differ.
- **Copyright:** lyrics are for research only. Never republish them (in the repo, report or demo).

### Song-side EDA (v2)

Fetching lyrics is part of the Stage 1 EDA:
- Lyrics coverage per source (LRCLIB, then the fallbacks). **Result (2026-10-09, 1,000 random MuSe songs):** 59% have lyrics on LRCLIB and 54% have English lyrics, so roughly 49,000 songs in the full catalogue. The Genius fallback isn't needed for now. Coverage favours popular, pop/rock songs; ambient, jazz and electronic are mostly missing. 55% of songs are over the 256-token limit, and 14% have no stanza breaks, so those fall back to fixed blocks of lines. Details are in the lyrics section of `DAP_IEDA_Template.ipynb`; the cache is `data/raw/lrclib/sample_1000.jsonl`.
- Share of instrumental and non-English songs. **Result:** 3.5% instrumental; 8% of songs with lyrics aren't English.
- Lyric length in word pieces, against the 256 limit.
- Match quality. **Result:** a hand-check of 50 matched songs found 0 wrong (`data/interim/lyrics_match_check.csv`).
- Distinctive words (log-odds). Done per valence × arousal quadrant for now: the seed groups aren't fixed yet, and 1,000 songs are too few per seed. **Result:** a weak signal at this sample size; only negative / high energy is clear (*death, flesh, hell*). Redo per seed group after the full fetch.
- Situations mentioned in lyrics (breakup, late night, leaving home), by keyword. This is the song-side input to deciding which themes become dimensions. **Result:** love 46%, night 36%, home / road 26%, breakup / leaving 22%. Breakup / leaving is flat across quadrants (21–25%).

### Evaluation changes (v2)

- **Song model:** beats the zero-shot baseline on held-out songs (seed labels), per dimension.
- **PMEmo:** a rough check that our song vectors, projected onto valence/arousal, roughly match PMEmo's human ratings on its lyrics. Details later.
- The Stage 4 prompt → song evaluation is unchanged.

### Research questions (v2)

1. How useful can song labels and descriptions be in determining a song's relevance to a user's prompt?
2. To what extent can an ML system meet a user's subjective requirements from it?
3. Can people's own descriptions of how they feel be matched to songs through a small set of named moods, without any examples of which songs suit which prompts?

### Still open (v2)

- Music4All access.

---

## v1 plan (Last.fm tags, superseded)

Everything below is the original plan, written before Last.fm track tags turned out to be empty. Kept for history.

## Idea

Map songs to moods/themes so a user can describe how they feel in plain language (e.g. *"I'm going through a breakup"*) and get songs that match that feeling.

## Architecture

This is a **dual-encoder** design: two encoders turn prompts and songs into vectors in one shared space, and those vectors are then compared.

```
User prompt ──────────────► Prompt Encoder ──► prompt vector ──┐
                                                                ▼
                                                  Shared Mood Space ──► Cosine Similarity Search ──► Ranked songs
                                                                ▲
Song (Last.fm tags) ──────► Song Encoder ────► song vector ────┘
```

### Encoder, embedding and head

Both encoders are built the same way:

```
"rain, 3am, piano"   (or a prompt, or later lyrics)
        │
   ENCODER: pretrained sentence transformer, frozen      ← understands language in general
        │
   embedding: [0.12, -0.80, 0.33, … 384 numbers]          ← meaning, but no names
        │
   HEAD: small layer, trained by us                       ← our questionnaire
        │
   sadness 0.91, loneliness 0.74, joy 0.05, …             ← N named mood scores (0–1)
```

- **Encoder (backbone):** a pretrained transformer (e.g. sentence-transformers `all-MiniLM-L6-v2`). It turns *any* text into an **embedding**, a list of a few hundred numbers that represents what the text means. Texts with similar meaning get similar embeddings ("lonely, 3am, rain" sits near "empty room, can't sleep"). We keep it frozen.
- **Head:** a small layer or MLP that turns the embedding into our N named scores. It is the only part we train.
- **Why this split:** training only the head is cheap, works with thousands (not millions) of examples, and the same encoder can read tags, prompts or lyrics. That is what lets a head trained on tags later be applied to lyrics (see [Lyrics and descriptions](#lyrics-and-descriptions)).

### Prompt Encoder
Turns the user's free-text prompt into a **mood vector**. Each value says how strongly the prompt expresses one mood/theme (e.g. sadness 0.98, nostalgia 0.7). Trained on EmpatheticDialogues.

### Song Encoder
Turns a song into a mood vector of the same shape, from the song's **Last.fm tags** (words listeners attached to the song, with weights). Trained on songs whose mood we know from their MuSe seed tag, with that seed hidden from the input (see [Stage 2](#stage-2-song-model-lastfm-tags--vector)).

### Shared Mood Space
Both encoders output scores for the **same named mood dimensions**, so dimension *i* means the same mood on both sides (e.g. dimension 0 = sadness for prompts *and* songs). The space is aligned **by design**: both sides are labelled with the same dimension definitions, so no prompt–song paired data is needed. Because every axis has a name, every match can be explained (e.g. "matched on sadness + nostalgia").

*Variant to test:* since both encoders read text through the same frozen encoder, one shared head trained on tags *and* prompts could replace the two separate heads. Compare one shared model against two separate ones in Stage 4.

### Cosine Similarity Search
Computes the cosine similarity between the prompt vector and every song vector, then returns the highest-scoring songs as recommendations. Cosine similarity measures the angle between two vectors and ignores their length, so it compares the *shape* of the mood profile rather than its overall intensity (1 = same profile, 0 = unrelated).

## Stages

### Stage 1: Define the dimensions (EDA)
Decide on the fixed set of named mood/theme dimensions that every vector will use. All later stages depend on this stage. A dimension only works if people **ask for it** (prompt side) *and* songs **carry it** (song side), so we look at both.

1. **Start from existing vocabularies.**
   - **AllMusic moods (via MuSe):** MuSe's 276 seed words come from the mood list written by AllMusic.com's editors (aggressive, bittersweet, lonely, nostalgic, wistful…). Many are about sound or style rather than mood (crunchy, literate, slick) and get filtered out.
   - **AllMusic themes:** AllMusic also lists themes/situations. We copy these words **by hand** from the public list page (no scraping: AllMusic's terms forbid automated collection) to cover what the moods miss.
   - **GEMS (Geneva Emotional Music Scale):** 9 music-specific moods: wonder, transcendence, tenderness, nostalgia, peacefulness, power, joyful activation, tension, sadness.
   - **EmpatheticDialogues' 32 moods:** which moods people describe in their own words.
2. **EDA on the song side.**
   - Count songs per MuSe seed, and seeds per song (80% of songs have one).
   - Once we have a Last.fm API key: fetch the full tags for a sample of songs, count tag frequencies, and drop non-mood tags ("rock", "seen live", "favourites").
   - Look for **themes and situations** in the full tags that the seeds miss (breakup, late night, rainy day, workout, road trip).
3. **EDA on the prompt side.**
   - Label frequencies and situation lengths in EmpatheticDialogues; which situations come up most (loss, breakups, exams, jobs).
4. **Group and fix the list.**
   - Group synonyms (sad / melancholy / gloomy → sadness) and show the mapping table for both MuSe seeds and EmpatheticDialogues labels.
   - Count how many songs and how many prompts each candidate dimension covers. A dimension with too few of either can't be trained or tested.
   - Aim for **~10 named dimensions**, each with a short written definition.

*Optional:* use a learned embedding space to discover dimensions (see [Optional Extensions](#optional-extensions)).

### Stage 2: Song model (Last.fm tags → vector)
- **Data:** MuSe's ~90k songs, with each song's full tags fetched from the **Last.fm API** (`track.getTopTags`, free key) using its artist and title.
- **Input:** the song's Last.fm tags (as text, with their weights), **with its seed word and that seed's synonyms removed**.
- **Label:** the dimension(s) that the song's MuSe seed belongs to (e.g. seed "melancholy" → sadness).
- **Why hide the seed:** if the answer word is still in the input, the model just learns to spot "sad" and scores perfectly while learning nothing. With it hidden, the model has to learn which *other* tags go with each mood (e.g. "rain", "3am", "piano", "autumn" → sadness). Then a song tagged only "late night, piano, rainy day" still gets a sadness score.
- **Model:** frozen sentence-transformer encoder + small trained head → N mood scores.
- **Simple baseline without training:** group the song's mood tags into the dimensions by rule (the Stage 1 mapping table) and average them by tag weight.
- **Output:** an `mood_vector` for every song with tags. This is the recommendation catalogue.

### Stage 3: Prompt model (Prompt Encoder)
- **Input:** free-text prompts.
- **Training data:** **EmpatheticDialogues** situations (main), with their 32 moods mapped onto our dimensions. **GoEmotions** as a second, larger source.
- **One label per situation:** EmpatheticDialogues gives one mood per situation, but the model outputs a score for every dimension. Either train it as "this dimension is high" (not "the others are zero"), or use **Jev** (System 1 model) to give each situation full scores across all dimensions.
- **Model:** frozen sentence-transformer encoder + small trained head → N mood scores.
- **Testing:** on held-out EmpatheticDialogues situations *and* our own hand-written prompts. EmpatheticDialogues is life situations, not music requests, so our prompts are the real test.
- **Output:** a prompt vector over the same N dimensions as the songs.

### Stage 4: Match and evaluate
- **Test set:** 50–100 realistic prompts (drawn partly from EmpatheticDialogues situations), with the team picking good matching songs for each.
- **Retrieval:** run cosine similarity search for each test prompt and measure how often the good songs rank near the top (precision@10).
- **Baseline to beat:** a simple **valence × energy** system. Map the prompt to a point on 2 axes (a small word list is enough), give each song Spotify-style valence and energy (Kaggle Spotify tables, or the ReccoBeats API for newer songs), and return the nearest songs. If our ~10 dimensions win, that's evidence they capture more than 2 axes.
- **Human check on the song side:** compare our song vectors with **PMEmo**'s human valence/arousal ratings (fetch Last.fm tags for PMEmo's songs). Tags can't be used to judge tags, so this check has to come from people.
- **Untagged songs (simulated):** hide the tags of some test songs and see how well the [lyrics and descriptions extension](#lyrics-and-descriptions) recovers their vectors.
- **Calibration:** make sure the score scales of the two encoders match, e.g. that the prompt encoder doesn't systematically output higher scores than the song encoder.


## Datasets

| Dataset / source | Contents | Used in | How we use it |
|---|---|---|---|
| **EmpatheticDialogues** | ~25k short first-person situations ("My dog died and I was heartbroken"), one of 32 moods each | Stage 1, 3, 4 | Main training/test data for the Prompt Encoder; prompt-side vocabulary for the dimensions; source of realistic test prompts |
| **MuSe** | ~90k songs, each found on Last.fm by one or more of 276 AllMusic mood words (the "seeds"), with lexicon valence/arousal/dominance and Spotify IDs (68%). No full tag list | Stage 1, 2 | Song list (catalogue), seed words for the dimension list, and seed labels for the song model |
| **Last.fm API** | Free-text tags with weights for any song, applied by Last.fm users (free API key) | Stage 1, 2, 4 | Full tags for MuSe and PMEmo songs: the song model's input |
| **AllMusic mood and theme lists** | Mood and theme words written by AllMusic editors | Stage 1 | Vocabulary only, copied by hand (no scraping) |
| **GoEmotions** | ~58k Reddit comments labelled with 27 moods | Stage 1, 3 | Second, larger training source for the Prompt Encoder |
| **PMEmo** | 794 chart pop songs with human valence/arousal ratings, lyrics, user comments and chorus clips | Stage 4; lyrics extension | Human check on song vectors; tags → lyrics transfer test |
| **Spotify feature tables + ReccoBeats** | Spotify-style valence and energy per song | Stage 4 | Valence × energy baseline. Not used for training |
| **Music4All** | ~109k songs with lyrics, Last.fm tags, audio clips and metadata (access by request) | Lyrics extension | Lyrics already joined with tags; would also serve the audio extension |
| **Song Describer** | Free-text captions for ~700 MTG-Jamendo tracks | Lyrics extension | Descriptions as an extra input |

Audio datasets (MTG-Jamendo, DEAM, Deezer/iTunes previews) are only needed for the [audio extension](#audio-branch).

### Labelling

- **Song labels:** MuSe seed words, grouped into our dimensions. These are tags real Last.fm users applied, so they are a rough human signal.
- **Prompt labels:** EmpatheticDialogues moods (chosen by each situation's writer) and GoEmotions labels, mapped onto our dimensions.
- **Jev (optional):** give each EmpatheticDialogues situation full scores across all dimensions, fixing its one-label-per-situation limit. Jev can't label songs that have no text, and a model trained to copy Jev can't be judged against Jev, so evaluation stays on human data (PMEmo, our test prompts).

## Optional Extensions

Only if we have time.

### Lyrics and descriptions
The song model only needs text, so it can be pointed at text other than tags. This covers songs that are not popular on Last.fm.
- **Direct transfer:** run lyrics or descriptions through the *same* frozen encoder and the *same* head trained on tags. No songs with both tags and lyrics are needed for training.
- **Domain shift:** tag lists ("indie, sad, piano") and lyrics (long, poetic, often indirect) are very different text, so expect it to work less well. Reduce it by training the head on mixed text (tags + EmpatheticDialogues situations, later captions), and measure the drop on **PMEmo** (lyrics + human ratings). How much is lost going from tags to lyrics is a research question in itself.
- **Training the same head further:** one head can keep learning from lyrics and descriptions; we don't need a model per input. It needs lyrics/descriptions with labels: songs that have both a MuSe seed and lyrics (e.g. Music4All, or a strict artist + title match), or PMEmo's ratings mapped onto our dimensions.
  - **Train on a mix, not lyrics alone.** Training only on lyrics makes the head forget what it learned from tags. Mix tag examples and lyrics examples in every batch.
  - **Mark the input type** with a short prefix (`tags: …`, `lyrics: …`, `description: …`), so the one head can treat each kind of text slightly differently.
  - **Separate heads per input** only if the shared head does clearly worse on one input type. Compare both on the same test songs.
- **Long lyrics:** split into verses, embed each, average. Flag non-English lyrics.
- **Sources:** PMEmo and Music4All lyrics, Song Describer captions, Genius annotations (other people's analysis of a song).
- **Combining inputs:** if a song has several inputs, run each through the head and average the resulting vectors (weights tuned on a validation set). A song with no tags uses whatever it has.
- **Coverage ladder:** track tags → lyrics/descriptions → artist tags (Last.fm `artist.getTopTags`; rougher, an artist's mood isn't the song's) → nothing (leave the song out).

### Audio branch
Analyse the audio directly, so the system works on any song with a 30-second preview, even with no tags or lyrics.
- **Input:** mel-spectrograms (a 2D time × frequency "image" of the audio) of 30s clips.
- **Model:** pretrained CNN backbone (frozen; Essentia models trained on MTG-Jamendo are a ready-made option) + small head → N mood scores.
- **Labels:** our Last.fm-derived song vectors (on songs with a preview), MTG-Jamendo mood/theme tags and DEAM valence/arousal, all mapped onto our dimensions.
- **Audio sources:** MTG-Jamendo and DEAM (CC music), PMEmo chorus clips, Music4All clips, Deezer/iTunes 30s previews for mainstream songs.
- **Experiment:** tags only vs audio only vs combined on the Stage 4 test set.
- Optionally train the CNN from scratch on MTG-Jamendo and compare with the pretrained version.

### Learned joint embedding space
Instead of (or alongside) named dimensions, a model learns its own axes (typically 256–1024 unnamed dimensions). Text and songs that go together end up close. It is trained with **contrastive learning**: both encoders are trained together on matching (text, song) pairs, pulling matching pairs together and pushing non-matching pairs apart.

- **Pros:** no taxonomy to design; can capture nuance beyond moods (genre, vibe, situations like "rainy Sunday drive").
- **Cons:** needs a lot of paired data; axes have no names, so matches can't be explained or easily debugged.

Ways we could use it:

1. **Dimension discovery (Stage 1).** Embed tags with the sentence transformer, cluster them, and name each cluster. Clusters that don't match our chosen dimensions are candidates for new ones. Clusters often mix concepts (e.g. "sad *and* acoustic *and* slow"), so a person still has to interpret them.
2. **Baseline (Stage 4).** Run an off-the-shelf pretrained text–audio model (e.g. CLAP) on the same test prompts and compare it with our interpretable model.
3. **Playlist titles as training pairs.** A playlist titled "breakup songs" is effectively a prompt with matching songs. The Spotify Million Playlist Dataset (~1M playlists) could supply pairs for contrastive training.

## Risks and Caveats

Things that could break the idea or weaken the results, plus what we plan to do about each. More detail per dataset is in [dataset_review.md](dataset_review.md).

### Labels
- **Seed leakage.** If a song's seed word (or a synonym) stays in its input tags, the song model learns to copy it.
  → Remove the seed and every word in the same dimension group from the input. Spot-check a sample of inputs.
- **MuSe's song selection is biased.** Every song is in MuSe *because* it was strongly tagged with a mood, about 1,000 songs per mood whatever its real frequency. Mood-neutral songs are missing, and the songs reflect Last.fm's userbase (indie/rock/electronic, mostly Western).
  → Report genre and artist distributions in EDA. State it as a limitation. Check on PMEmo (chart pop, not chosen by mood word).
- **The seed vocabulary is AllMusic's, not ours.** It covers how music sounds and feels, not life situations (no heartbreak, breakup, party, workout).
  → Add situations from the full Last.fm tags, AllMusic's theme list and the prompt side (EmpatheticDialogues).
- **MuSe's valence/arousal are not human ratings of the music.** They are a word lexicon (Warriner et al.) applied to the songs' tags.
  → Don't use them as labels. At most a weak sanity check.
- **EmpatheticDialogues has one writer-chosen label per situation**, with no second rater, and about 9 of its 32 labels aren't music moods (guilty, ashamed, jealous…).
  → Treat labels as "this one is high", or add Jev scores. Map or drop non-music labels. Report results per dimension.
- **GoEmotions is Reddit comments, not music requests.** The labels are skewed (many "neutral"/"admiration"), and an outside audit reported a large share of mislabelled examples.
  → Use it as the second source, behind EmpatheticDialogues.
- **Human-rated song data is small.** PMEmo has 794 songs.

### Model and design
- **Last.fm tag coverage.** Popular and older songs are well tagged; recent and obscure songs have few or no tags.
  → Measure coverage on a random sample once we have an API key. Lyrics/descriptions (extension) and artist tags extend coverage; v1 only recommends songs that have tags.
- **Domain shift from tags to lyrics** (lyrics extension). A head trained on tag lists may do worse on lyrics.
  → Train on mixed text and measure the drop on PMEmo.
- **Cosine similarity ignores intensity.** "A bit down" and "devastated" point in the same direction, so they get the same songs.
  → Consider re-ranking the top results by vector length, or a distance that keeps magnitude. Decide in Stage 4.
- **All scores are 0–1, so every cosine similarity is positive.** Scores may bunch together, and a vague prompt ("play something") gives a near-zero vector whose direction is basically noise.
  → Look at the spread of scores in Stage 4. Detect weak prompts and fall back to something sensible (e.g. ask the user, or use popularity).
- **Named dimensions limit what users can ask for.** Prompts about genre, tempo or situations outside the list ("rainy Sunday drive") won't map well.
  → Stage 1 decides whether themes/situations become dimensions. The learned embedding extension covers the rest.

### Data access and legal
- **Last.fm API needs a key and polite request rates.** ~90k songs means ~90k tag requests, an overnight job.
  → Get the key early, cache every response, and read the API terms (non-commercial use).
- **No scraping AllMusic.** Its terms forbid automated collection, including for training AI/ML.
  → Copy only the mood/theme word lists by hand, and cite AllMusic.
- **Matching songs across sources by artist + title** can attach the wrong tags (same title, covers, live versions, remixes). A wrong match is worse than a missing one.
  → Use IDs where possible, normalise artist and title, drop ambiguous matches, and hand-check ~100 matches to report an error rate.
- **Spotify's API has been cut back.** Since Nov 2024 new apps can't get audio features or recommendations. In Development Mode the owner needs Premium, with max 5 users per app. Spotify's developer terms may forbid training ML on Spotify content.
  → Do all ML on open data. Use Spotify only (optionally) to play the final list.
- **Lyrics are copyrighted.**
  → Use them for research only and never republish them.
- **Music4All needs an access request.**
  → Request early. The core plan works without it.
- **No YouTube downloading.** It's against YouTube's terms (this is what shut down the original Rythm bot). That rules out MusicCaps audio.

### Evaluation and scope
- **"Matches my mood" is subjective.** Test prompts and "good matches" picked by the team reflect the team's taste.
  → Use PMEmo's human ratings for the song side, and run a small listening study (n ≥ 10).

## Open Questions

- How many dimensions (~10?), and which themes/situations count as dimensions?
- Do we use Jev for multi-dimension prompt scores, or train on single labels?
- One shared text head for songs and prompts, or two separate heads?
- How well tagged are songs on Last.fm outside MuSe (coverage on a random sample)?
- Do we get Music4All access in time for the lyrics extension?
