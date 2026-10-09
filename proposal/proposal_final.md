# DAP Project Proposal: MoodMix

> Paste-ready text for `DAP_Project_Proposal_Template.docx`, section by section.
> `[TODO]` = still to fill in. Working notes stay in [proposal_framework.md](proposal_framework.md).
> Describes the v2 plan (lyrics as the song model's input). Why we changed from Last.fm tags: [project-idea.md](project-idea.md#why-we-pivoted-no-lastfm-track-tags).

---

## 1. Team

`[TODO]` Names, year, course, background, commitments, individual learning objectives.

## 2. Group Learning Objectives

1. **Understand the full data workflow, end to end:** collect raw data from several sources, clean it, and shape it into a form a model can learn from.
2. **Learn how to distil music into emotions:** decide which emotions matter for music, and teach a model to read them from a song's lyrics.

## 3. Mentor Preference

`[TODO]`

---

## 4. Project Idea

### Project title
**MoodMix**

### 4.1 Problem statement and motivation
`[TODO]` To be rewritten. Angle: tools that turn a mood into songs exist, but they don't do it well, and we want to do it better.

### 4.2 Project description / proposed approach

MoodMix lets a user describe how they feel in their own words (*"I'm going through a breakup"*) and returns songs that match that feeling, along with the moods each song was matched on.

*[architecture figure]*

**How it works**
- **A shared list of moods.** We pick about 10 named moods (e.g. sadness, nostalgia, tension, joy). The user's prompt and every song each get a score from 0 to 1 for every mood. Because both sides are scored on the same named moods, they line up without needing examples of which songs suit which prompts, and every result can be explained ("matched on sadness + nostalgia").
- **Prompt model.** Reads the user's text and gives its mood scores. It is trained on EmpatheticDialogues: about 25,000 short descriptions of real-life situations (*"My dog died and I was heartbroken"*), each labelled with the emotion its writer felt.
- **Song model.** Reads a song's lyrics and gives its mood scores. It is trained on MuSe: about 90,000 songs, each found on Last.fm through a mood word that listeners gave it, such as *"melancholy"*. That mood word is the label; the lyrics are the input.
- **How both models are built.** A pretrained language model (a sentence transformer) turns text into an embedding, a list of numbers that captures what the text means. A small layer that we train on top turns that embedding into the mood scores. Only the small layer is trained, so a few thousand examples are enough, and the same setup reads both prompts and lyrics.
- **Matching.** We compare the prompt's mood scores with every song's scores (cosine similarity) and return the closest songs.

**What lyrics can and can't tell us.** The mood words in MuSe come from listeners, who react to the whole song: its sound as well as its words. A song can sound upbeat and still have sad lyrics. So we expect some moods to be easy to read from lyrics (sadness) and others hard (tension, power). Measuring which ones is our third research question.

**Labels.** EmpatheticDialogues gives one emotion per situation, but a breakup is sad *and* lonely *and* maybe nostalgic. Optionally, Jev gives each situation a score for every mood.

**Workflow**

| Stage | What we do | Output |
|---|---|---|
| 1. Choose the moods | Explore both datasets, fetch lyrics for a sample of songs, group similar mood words (sad / melancholy / gloomy → sadness), check each mood has enough songs and prompts | ~10 named moods, each with a short definition |
| 2. Song model | Fetch lyrics for the catalogue, keep English songs with lyrics, train the model | Mood scores for every song with lyrics |
| 3. Prompt model | Map EmpatheticDialogues' 32 emotions onto our moods, train the model | Mood scores for any prompt |
| 4. Match and evaluate | Build a test set of 50–100 realistic prompts with songs the team picks as good matches; check how many of them appear in the top 10 | Results for our research questions |

**Final output.** A demo where the user types how they feel and gets a ranked list of songs, each with the moods it matched on, which they can save as a Spotify playlist.

**If time allows.** Read the audio as well as the lyrics, to cover instrumental songs and moods that live in the sound.

### 4.3 Key research questions
1. Can people's own descriptions of how they feel be matched to songs through a small set of named moods, without any examples of which songs suit which prompts?
2. Do about 10 named moods match prompts to songs better than only scoring how positive and how energetic they are?
3. How much of a song's listener-perceived mood can its lyrics alone predict?

### 4.4 Intended users / beneficiaries and expected value

| Stakeholder / user | How they may use or benefit from the project |
|---|---|
| Casual listeners | Get songs for how they feel right now, described in their own words, with a reason for each pick |
| Researchers and students | Study how emotions map onto music (and back), using an open method where every match can be explained |

---

## 5. Machine Learning Techniques Required

- **NLP:** pretrained sentence transformers to turn prompts and song lyrics into embeddings.
- **Transfer learning:** keep the pretrained model fixed and train only a small layer on top.
- **Multi-label classification:** one score per mood for each prompt or song.
- **Recommendation / retrieval:** rank songs by how similar their mood scores are to the prompt's.
- **Label design:** grouping mood words into named moods, and using listeners' mood words as labels for lyrics.
- **Evaluation:** how many good songs appear in the top 10 results (precision@10) on our own test prompts; the song model compared with a no-training baseline.
- *Optional:* LLM-assisted labelling (Jev), audio models on spectrograms.

## 6. Research Done

| No. | Link / source | What the research is about | How it informs our project |
|---|---|---|---|
| 1 | Zentner, Grandjean & Scherer (2008), *Emotions evoked by the sound of music* (GEMS). [doi:10.1037/1528-3542.8.4.494](https://doi.org/10.1037/1528-3542.8.4.494) | Nine emotions people actually feel from music, such as nostalgia, wonder and tenderness | Starting list for our moods. Shows that two axes miss feelings that matter in music |
| 2 | Eerola & Vuoskoski (2011), *A comparison of the discrete and dimensional models of emotion in music*. [doi:10.1177/0305735610362821](https://doi.org/10.1177/0305735610362821) | Compares named emotions with the two-axis view, which scores only how positive and how energetic music is | The two-axis view is the usual way music data describes mood (e.g. Spotify's "valence" and "energy" scores). This study asks the same question as our second research question |
| 3 | Warriner, Kuperman & Brysbaert (2013), *Norms of valence, arousal, and dominance for 13,915 English lemmas*. [doi:10.3758/s13428-012-0314-x](https://doi.org/10.3758/s13428-012-0314-x) | Over 1 million crowd ratings of how positive, how energetic and how in-control about 14,000 English words feel, on a 1–9 scale | The source of MuSe's mood scores: MuSe looks up each song's mood words in this list and averages them. So the scores rate the words, not the music, which is why we don't use them as labels. It could also place prompts and lyrics on the two axes for our second research question |

## 7. Datasets / Data Sources to be Used

| No. | Dataset / API / source | Link / access method | Key information available | Planned usage in project |
|---|---|---|---|---|
| 1 | MuSe | [Kaggle](https://www.kaggle.com/datasets/cakiki/muse-the-musical-sentiment-dataset), free download (CC BY 4.0) | 90,001 songs, each found on Last.fm through one or more of 276 mood words; Spotify IDs for 68% | The song catalogue, and the mood words as labels for the song model |
| 2 | LRCLIB | [lrclib.net](https://lrclib.net), free API, no key | Song lyrics, searchable by artist and title; flags instrumental songs | The song model's input |
| 3 | Genius Song Lyrics *(fallback, tentative)* | [Kaggle](https://www.kaggle.com/datasets/carlosgdcj/genius-song-lyrics-with-language-information), free download, research use only | Lyrics for ~5M songs, with a language label | Extra lyrics for songs LRCLIB doesn't have, if needed |
| 4 | Music4All *(tentative, access by request)* | Email request to the dataset authors | ~109k songs with lyrics and Last.fm tags | Extra lyrics, and tags collected before Last.fm stopped returning them |
| 5 | EmpatheticDialogues | [GitHub](https://github.com/facebookresearch/EmpatheticDialogues), free download (CC BY-NC 4.0) | 24,850 short first-person situations, each labelled with one of 32 emotions | Training and testing the prompt model; a source of realistic test prompts |

## 8. Initial EDA / Preliminary Exploration

**Notebook / repository / link:** `[TODO]`

**Key observations, feasibility checks and early findings**

*EmpatheticDialogues (prompt side)*
- 24,850 situations, 32 emotions, fairly balanced (478 to 1,279 per emotion), so no resampling is needed.
- Situations are short (median 16 words), and 76% are written in the first person. This is the closest public text we found to what a user would type.
- Cleaning needed: 21 situations contain no words at all, and 78 repeated situations carry conflicting emotion labels. Four situations appear in both the training and test files, so we move them into one.
- Only 116 situations mention music, and about 9 of the 32 emotions (e.g. guilty, jealous) are not music moods, so these need mapping or dropping. Our own hand-written prompts are the real test.

*MuSe (song side)*
- 90,001 songs from 26,012 artists, mostly indie, rock and electronic, which reflects who uses Last.fm.
- MuSe has no tags or lyrics for each song, only the mood word that found it. We planned to fetch each song's tags from the Last.fm API, but a check on 506 random songs found **500 with no tags**: Last.fm now returns tags only for very popular songs. So the song model reads lyrics instead.
- MuSe's own mood scores come from a word list applied to the tags, not from people rating the music. 48% of songs share their exact score with another song, so we don't use these scores as labels.
- Mood words are unevenly used: median 290 songs each, and 91 of the 276 have fewer than 100 songs. Some words describe sound rather than mood (e.g. "crunchy", "slick"). We keep them and check which moods lyrics can actually predict.
- 1,863 rows are the same song listed twice with slightly different titles, so the catalogue needs de-duplicating.

*Lyrics (LRCLIB, a random sample of 1,000 MuSe songs)*
- 59% of the songs have lyrics on LRCLIB, and 54% have English lyrics. If that holds for the whole catalogue, about 49,000 songs are usable, far more than the model needs. (On the same songs, Last.fm had tags for 6 of 506.)
- We checked 50 matched songs by hand, and all 50 were the right song.
- Coverage leans towards popular songs with vocals: about 87% of pop and rock songs have lyrics, but only 4% of ambient songs and 44% of electronic ones. Calm and dreamy moods will therefore have fewer songs.
- 55% of the lyrics are longer than the language model reads in one go, so we split each song into verses and average them.

*Both datasets: a first grouping of mood words*
- A first grouping of all 276 MuSe mood words and 32 EmpatheticDialogues emotions gives 16 candidate moods. Five of them (e.g. *ethereal*, *romance*) have songs but no situations, and *shame* has only 55 songs. Merging these down to about 10 moods, each with enough songs and situations, is the next step of Milestone 1.

## 9. Project Milestones

| Milestone | Main goals / tasks | Success criteria / key deliverable | Target completion |
|---|---|---|---|
| Milestone 1: Choose the moods | Finish the EDA on both datasets; fetch lyrics for a sample of songs and measure coverage; group similar mood words | ~10 named moods with definitions, and enough songs and prompts for each | |
| Milestone 2: Song model | Fetch lyrics for the catalogue; keep English songs with lyrics; train the song model | Mood scores for every song with lyrics; the trained model beats a no-training baseline on songs it hasn't seen | |
| Milestone 3: Prompt model | Map EmpatheticDialogues' emotions onto our moods; train the prompt model | Mood scores for any prompt; accuracy per mood on held-out situations and our own prompts | |
| Milestone 4: Matching, evaluation and demo | Build the 50–100 prompt test set; match prompts to songs; answer the research questions; build the demo with Spotify playlist export | Top-10 results measured on the test set; a working demo; final report and presentation | |

### Risks and mitigations

| Risk | Mitigation |
|---|---|
| Listeners' mood words partly describe how a song sounds, which lyrics can't show | Report results per mood; this is what our third research question measures |
| Many songs have no lyrics available, and some are instrumental or not in English | The EDA sample found English lyrics for 54% of songs, which is enough. Keep English songs with lyrics, state it as a limitation, and add a second lyrics source if a mood ends up with too few songs |
| Lyrics are copyrighted | Use them for research only; never republish them in the repository, report or demo |
| MuSe's songs lean towards Last.fm's listeners (indie, rock, Western) and leave out songs with no clear mood | Report the genre spread; state it as a limitation |
| EmpatheticDialogues gives only one emotion per situation, and some emotions aren't music moods | Treat the label as "this mood is high" rather than "the others are zero", or add Jev scores; map or drop non-music emotions |
| Whether a song "matches my mood" is subjective | Build a test set of realistic prompts, and check the results with a small group of listeners |
