# Emotion-Based Song Matching

## Idea

Map songs to emotions/themes so a user can describe how they feel in plain language (e.g. *"I'm going through a breakup"*) and get songs that match that feeling.

## Architecture

This is a **dual-encoder** design: two separate encoders turn prompts and songs into vectors in one shared space, and those vectors are then compared.

```
User prompt ──► Prompt Encoder ──► prompt vector ──┐
                                                    ▼
                                        Shared Emotion Space ──► Cosine Similarity Search ──► Ranked songs
                                                    ▲
Song (description / audio) ──► Song Encoder ──► song vector ──┘
```

Every encoder has two parts:

- **Backbone**: a large pretrained model (a CNN or transformer) that turns raw input into a general-purpose feature vector.
- **Head**: a small layer on top that converts those features into our N emotion scores (0–1 each). Usually only the head is trained.

### Prompt Encoder
Takes the user's free-text prompt and turns it into an **emotion vector**. Each value says how strongly the prompt expresses one emotion/theme (e.g. sadness 0.98, nostalgia 0.7).

- **Backbone:** transformer (sentence-embedding / BERT-style model)
- **Head:** small MLP → N emotion scores

### Song Encoder
Turns a song into an emotion vector of the same shape. It has two branches:

- **Text branch**: applies NLP to existing descriptions of the song (tags, human-written descriptions, other people's analysis).
  - **Backbone:** transformer
  - **Head:** small MLP → N emotion scores
- **Audio branch**: analyses the audio track directly. It covers songs that have no descriptions.
  - **Input:** mel-spectrogram (a 2D time × frequency "image" of the audio)
  - **Backbone:** CNN (pretrained)
  - **Head:** small MLP → N emotion scores

### Fusion
Combines the text-branch and audio-branch vectors into one song vector with a weighted average (e.g. `0.6·audio + 0.4·text`). If a song has no descriptions, Fusion uses the audio vector alone.

### Shared Emotion Space
Both encoders output scores for the **same named emotion dimensions**, so dimension *i* means the same emotion on both sides (e.g. dimension 0 = sadness for prompts *and* songs). The space is aligned **by design**: both sides are labelled with the same dimension definitions, so no prompt–song paired data is needed. Because every axis has a name, every match can be explained (e.g. "matched on sadness + nostalgia").

### Cosine Similarity Search
Computes the cosine similarity between the prompt vector and every song vector, then returns the highest-scoring songs as recommendations. Cosine similarity measures the angle between two vectors and ignores their length, so it compares the *shape* of the emotional profile rather than its overall intensity (1 = same profile, 0 = unrelated).

## Stages

### Stage 0: Define the dimensions (EDA)
Decide on the fixed set of named emotion/theme dimensions that every vector will use. All later stages depend on this stage.

1. **Start from existing taxonomies.**
   - **GEMS (Geneva Emotional Music Scale):** 9 music-specific emotions: wonder, transcendence, tenderness, nostalgia, peacefulness, power, joyful activation, tension, sadness.
   - **Valence/arousal:** how positive and how energetic (used by DEAM and MuSe).
   - **GoEmotions' 27 emotions:** shows which emotions appear in prompt-style text.
2. **EDA on tags.**
   - Collect the tag vocabulary from MuSe/Last.fm (and Music4All if we get access), count tag frequencies, and drop non-emotion tags ("seen live", "favourites").
   - Group synonyms (sad / melancholic / depressing → sadness).
   - Look for **themes and situations** the taxonomies miss (heartbreak, party, late night, workout, road trip) and decide which become dimensions.
3. **Fix the final list.** Aim for ~10–20 named dimensions, each with a short written definition. These definitions are also the instructions Jev uses for labelling, so songs and prompts are labelled consistently.

*Optional:* use a learned embedding space to discover dimensions (see [Optional Extensions](#optional-extensions)).

### Stage 1: Song descriptions → vector (Song Encoder, text branch)
- **Input:** song tags, descriptions and (where available) lyrics.
- **Labels:** emotion scores per song from Jev, checked against PMEmo's human valence/arousal ratings (PMEmo has lyrics and comments) and, weakly, MuSe's lexicon scores.
- **Model:** pretrained transformer backbone (frozen) + small trained head → N emotion scores.
- **Simple baseline without training:** score each tag against the dimension definitions using word embeddings or an emotion lexicon, then average the scores per song.
- **Output:** a `text_emotion_vector` column for every song that has descriptions.

### Stage 2: Prompt → vector (Prompt Encoder)
- **Input:** free-text prompts.
- **Labels:** GoEmotions and EmpatheticDialogues (mapped onto our dimensions) plus Jev-labelled example prompts.
- **Model:** pretrained transformer backbone (frozen) + small trained head → N emotion scores.
- **Output:** a prompt vector over the same N dimensions as the songs.

### Stage 3: Align and validate (Shared Emotion Space + Cosine Similarity Search)
- **Test set:** write 50–100 realistic prompts and have the team pick good matching songs for each (Jev can help draft the prompts).
- **Check retrieval:** run cosine similarity search for each test prompt and measure how often the good songs rank near the top (e.g. precision@10).
- **Check calibration:** make sure the score scales of the two encoders match, e.g. that the prompt encoder doesn't systematically output higher scores than the song encoder.
- **Check consistency:** on Song Describer tracks (text and audio for the same songs), confirm that the text-branch and audio-branch vectors broadly agree.

### Stage 4: Add audio (Song Encoder, audio branch + Fusion)
- **Input:** mel-spectrograms of each track.
- **Labels:** MTG-Jamendo mood/theme tags (mapped onto our dimensions) and DEAM valence/arousal ratings.
- **Model:** pretrained CNN backbone (frozen) + small trained head → N emotion scores.
- **Inference:** run the trained model on every track we have audio for, including untagged ones, and store the result as an `audio_emotion_vector` column.
- **Fusion:** combine `text_emotion_vector` and `audio_emotion_vector` into the final song vector; fall back to audio only when there are no descriptions.
- **Experiment:** compare audio only vs text only vs combined on the Stage 3 test set. If combined wins, that is our evidence for analysing songs "from both sides".

## Datasets

| Dataset | Contents | Used in | How we use it |
|---|---|---|---|
| **MuSe** | ~90k songs, each found on Last.fm by one or more of 276 mood tags, with lexicon valence/arousal/dominance and Spotify IDs (68%). No full tag list | Stage 0, 1 | Seed mood words for the dimension list; song list for the text branch, with full tags fetched from the Last.fm API |
| **Music4All** | ~109k songs with audio clips, lyrics, tags and metadata (access by request) | Stage 1, 4 | Has text *and* audio for the same songs, so we can train both branches and test Fusion on one catalog |
| **GoEmotions** | ~58k Reddit comments labelled with 27 emotions | Stage 0, 2 | Everyday emotional text, similar to user prompts; trains/tests the Prompt Encoder and informs the dimension list |
| **EmpatheticDialogues** | ~25k short first-person situations ("My dog died and I was heartbroken"), one of 32 emotions each | Stage 0, 2, 3 | Closest text to real prompts; trains/tests the Prompt Encoder with GoEmotions; source of realistic test prompts |
| **MTG-Jamendo (mood/theme)** | ~18k Creative Commons tracks with ~60 mood/theme tags (~55k tracks overall) | Stage 4 | Main training set for the audio branch. Planned for after the first two models (Stages 1–2) |
| **DEAM** | ~1.8k songs with continuous valence/arousal ratings | Stage 4 | Adds intensity ("how sad") to the audio branch, which tag-only data lacks |
| **PMEmo** | 794 chart pop songs with human valence/arousal ratings, lyrics, user comments and chorus clips | Stage 1, 4 | Human check on the text branch (lyrics + comments) in the first phase; domain-shift check on mainstream audio later |
| **Song Describer** | Free-text captions for ~700 MTG-Jamendo tracks | Stage 3 | Text and audio for the same tracks; used to check that the text and audio branches agree |

### Labelling

- Use **Jev** (the new System 1 model) to classify songs and prompts into our emotion dimensions, using the Stage 0 definitions, then train our encoders on those labels.
- Check Jev's labels against human-labelled data (GoEmotions, DEAM, MTG-Jamendo tags) before trusting them.

## Optional Extensions

Only if we have time or need more complexity.

### Learned joint embedding space
Instead of (or alongside) named dimensions, a model learns its own axes (typically 256–1024 unnamed dimensions). Text and songs that go together end up close. It is trained with **contrastive learning**: both encoders are trained together on matching (text, song) pairs, pulling matching pairs together and pushing non-matching pairs apart.

- **Pros:** no taxonomy to design; can capture nuance beyond emotions (genre, vibe, situations like "rainy Sunday drive").
- **Cons:** needs a lot of paired data; axes have no names, so matches can't be explained or easily debugged.

Ways we could use it:

1. **Dimension discovery (Stage 0).** Embed tags (sentence transformer) or songs (CLAP / audio embeddings), cluster them, and name each cluster. Clusters that don't match our chosen dimensions are candidates for new ones. Clusters often mix concepts (e.g. "sad *and* acoustic *and* slow"), so a person still has to interpret them.
2. **Baseline (Stage 3).** Run an off-the-shelf pretrained text–audio model (e.g. CLAP) on the same test prompts and compare it with our interpretable model.
3. **Playlist titles as training pairs.** A playlist titled "breakup songs" is effectively a prompt with matching songs. The Spotify Million Playlist Dataset (~1M playlists) could supply pairs for contrastive training.

### Train the audio CNN from scratch
Train a CNN end-to-end on MTG-Jamendo spectrograms instead of using a pretrained backbone, and compare its accuracy with the pretrained version.

## Risks and Caveats

Things that could break the idea or weaken the results, plus what we plan to do about each. More detail per dataset is in [dataset_review.md](dataset_review.md).

### Labels
- **MuSe's valence/arousal are not human ratings of the music.** They come from applying a word lexicon (Warriner et al.) to each song's Last.fm tags. Using them to check the text branch, which also reads tags, is partly circular. The MuSe authors also warn about many songs sharing the same score because they share one seed tag.
  → Use them as a weak sanity check only. Use DEAM / PMEmo human ratings for the real check.
- **MTG-Jamendo mood tags are noisy.** The artists tagged their own tracks (sometimes for marketing). A missing tag doesn't mean the mood is absent, and a few moods dominate.
  → Treat missing tags as "unknown", not "no". Evaluate on the human re-labelled test subset (Music Classification Annotations).
- **GoEmotions is Reddit comments, not music requests.** The labels are skewed (many "neutral"/"admiration"), and an outside audit reported a large share of mislabelled examples. Mapping 27 emotions onto our dimensions will also lose some meaning.
  → Add Jev-labelled, prompt-style examples. Report results per dimension, not just overall.
- **Jev's labels carry Jev's biases.** If the encoders are trained on Jev's labels, they learn whatever Jev gets wrong, and the errors may be consistent across songs and prompts (so retrieval looks fine while being wrong).
  → Check Jev against human labels before training (already planned). Keep a human-labelled test set that Jev never touched.
- **Human-labelled emotion data is small.** DEAM has ~1.8k songs and PMEmo 794.

### Model and design
- **Domain shift in the audio branch.** It would be trained on independent Creative Commons music (MTG-Jamendo, DEAM), which sounds different from mainstream pop.
  → Check it on a small hand-labelled sample of mainstream songs, and on PMEmo (chart pop with human ratings).
- **Cosine similarity ignores intensity.** "A bit down" and "devastated" point in the same direction, so they get the same songs. That partly cancels out why we added DEAM ("how sad").
  → Consider re-ranking the top results by vector length, or a distance that keeps magnitude. Decide in Stage 3.
- **All scores are 0–1, so every cosine similarity is positive.** Scores may bunch together, and a vague prompt ("play something") gives a near-zero vector whose direction is basically noise.
  → Look at the spread of scores in Stage 3. Detect weak prompts and fall back to something sensible (e.g. ask the user, or use popularity).
- **Text and audio can disagree on purpose.** For example, sad lyrics over an upbeat track. A weighted average blurs this into "neutral".
  → Keep both vectors and look at disagreements in the Stage 3 consistency check before deciding how to fuse them.
- **Named dimensions limit what users can ask for.** Prompts about genre, tempo or situations outside the list ("rainy Sunday drive") won't map well.
  → Stage 0 decides whether themes/situations become dimensions. The learned embedding extension covers the rest.

### Data access and legal
- **No mainstream songs with full audio.** Mainstream datasets (Spotify-feature tables, MuSe) have no audio. Datasets with audio are mostly indie CC music. Mainstream audio is available legally only as 30-second previews (Deezer, iTunes), and Music4All also has 30s clips.
  → Run the audio branch on 30s previews/clips, and accept that a clip may not represent the whole song.
- **Spotify's API has been cut back.** Since Nov 2024 new apps can't get audio features or recommendations. In Development Mode the owner needs Premium, with max 5 users per app. Spotify's developer terms may forbid training ML on Spotify content.
  → Do all ML on open datasets. Use Spotify only (optionally) to play the final list.
- **No YouTube downloading.** It's against YouTube's terms (this is what shut down the original Rythm bot). That rules out MusicCaps audio.
- **Music4All needs an access request**, and the Spotify Million Playlist Dataset is no longer downloadable from AIcrowd.
  → Request early. Have a plan that works without them.
- **Matching tracks across datasets loses data**, and Spotify-derived tables contain duplicates (the same song on several releases).
  → Measure the match rate and deduplicate during EDA.

### Evaluation and scope
- **"Matches my mood" is subjective.** Test prompts and "good matches" picked by the team reflect the team's taste.
  → Add Song Describer captions as a second test set (each caption has a known song), and run a small listening study (n ≥ 10).
- **Stage 4 (audio) is the heaviest stage**: large downloads, spectrograms, GPU time.
  → Use frozen pretrained backbones (Essentia models, trained on MTG-Jamendo, are a ready-made baseline), work on a sample of the catalogue, and use Colab/Kaggle GPUs.

## Open Questions

- How to label the data / where to get labelled data beyond the datasets above
- Which song catalog do users get recommendations from (Music4All vs MTG-Jamendo vs MuSe)?
- How many dimensions, and do we include themes/situations as well as emotions?
- Fusion weights: fixed (e.g. 0.6/0.4) or tuned on the Stage 3 test set?
