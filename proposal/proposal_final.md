# DAP Project Proposal: MoodMix

> Paste-ready text for `DAP_Project_Proposal_Template.docx`, section by section.
> `[TODO]` = still to fill in. Working notes stay in [proposal_framework.md](proposal_framework.md).

---

## 1. Team

`[TODO]` Names, year, course, background, commitments, individual learning objectives.

## 2. Group Learning Objectives

1. **Understand the full data workflow, end to end:** collect raw data from several sources, clean it, and shape it into a form a model can learn from.
2. **Learn how to distil music into emotions:** decide which emotions matter for music, and teach a model to read them from how people describe songs.

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
- **Song model.** Reads the tags Last.fm listeners have attached to a song (*"rainy day, piano, late night"*) and gives its mood scores. It is trained on MuSe: about 90,000 songs, each found on Last.fm through a mood word such as *"melancholy"*.
- **How both models are built.** A pretrained language model (a sentence transformer) turns text into an embedding, a list of numbers that captures what the text means. A small layer that we train on top turns that embedding into the mood scores. Only the small layer is trained, so a few thousand examples are enough, and the same setup can read prompts, tags or, later, lyrics.
- **Matching.** We compare the prompt's mood scores with every song's scores (cosine similarity) and return the closest songs.

**A problem we designed around.** Each MuSe song comes with the mood word that found it. If that word is still among the song's tags, the model just learns to copy it and learns nothing else. So we hide that word and its synonyms from the input. The model then has to learn which *other* tags go with each mood (*"rain, 3am, piano"* → sadness), which is what lets it score songs that were never tagged with a mood word.

**Labels.** EmpatheticDialogues gives one emotion per situation, but a breakup is sad *and* lonely *and* maybe nostalgic. Optionally, Jev gives each situation a score for every mood.

**Workflow**

| Stage | What we do | Output |
|---|---|---|
| 1. Choose the moods | Explore both datasets, group similar mood words (sad / melancholy / gloomy → sadness), check each mood has enough songs and prompts | ~10 named moods, each with a short definition |
| 2. Song model | Fetch each song's Last.fm tags, hide the mood word, train the model | Mood scores for every tagged song |
| 3. Prompt model | Map EmpatheticDialogues' 32 emotions onto our moods, train the model | Mood scores for any prompt |
| 4. Match and evaluate | Build a test set of 50–100 realistic prompts with songs the team picks as good matches; check how many of them appear in the top 10 | Results for our research questions |

**Final output.** A demo where the user types how they feel and gets a ranked list of songs, each with the moods it matched on, which they can save as a Spotify playlist.

**If time allows.** Read lyrics or audio as well as tags, to cover songs that few people have tagged.

### 4.3 Key research questions
1. Can people's own descriptions of how they feel be matched to songs through a small set of named moods, without any examples of which songs suit which prompts?
2. Do about 10 named moods match prompts to songs better than only scoring how positive and how energetic they are?

### 4.4 Intended users / beneficiaries and expected value

| Stakeholder / user | How they may use or benefit from the project |
|---|---|
| Casual listeners | Get songs for how they feel right now, described in their own words, with a reason for each pick |
| Researchers and students | Study how emotions map onto music (and back), using an open method where every match can be explained |

---

## 5. Machine Learning Techniques Required

- **NLP:** pretrained sentence transformers to turn prompts and song tags into embeddings.
- **Transfer learning:** keep the pretrained model fixed and train only a small layer on top.
- **Multi-label classification:** one score per mood for each prompt or song.
- **Recommendation / retrieval:** rank songs by how similar their mood scores are to the prompt's.
- **Label design:** grouping mood words into named moods, and hiding the answer word from the input so the model can't copy it (preventing label leakage).
- **Evaluation:** how many good songs appear in the top 10 results (precision@10) on our own test prompts, and accuracy per mood.
- *Optional:* LLM-assisted labelling (Jev), clustering to discover moods, audio models on spectrograms.

## 6. Research Done

| No. | Link / source | What the research is about | How it informs our project |
|---|---|---|---|
| 1 | Russell (1980), *A circumplex model of affect*. [doi:10.1037/h0077714](https://doi.org/10.1037/h0077714) | Describes all emotions on two axes: how positive and how energetic | The usual way music data describes mood (e.g. Spotify's "valence" and "energy" scores). Our second research question tests whether ~10 named moods do better |
| 2 | Zentner, Grandjean & Scherer (2008), *Emotions evoked by the sound of music* (GEMS). [doi:10.1037/1528-3542.8.4.494](https://doi.org/10.1037/1528-3542.8.4.494) | Nine emotions people actually feel from music, such as nostalgia, wonder and tenderness | Starting list for our moods. Shows that two axes miss feelings that matter in music |
| 3 | Eerola & Vuoskoski (2011), *A comparison of the discrete and dimensional models of emotion in music*. [doi:10.1177/0305735610362821](https://doi.org/10.1177/0305735610362821) | Compares named emotions with the two-axis view for music | The same question as our second research question, asked by music psychologists |

## 7. Datasets / Data Sources to be Used

| No. | Dataset / API / source | Link / access method | Key information available | Planned usage in project |
|---|---|---|---|---|
| 1 | MuSe | [Kaggle](https://www.kaggle.com/datasets/cakiki/muse-the-musical-sentiment-dataset), free download (CC BY 4.0) | 90,001 songs, each found on Last.fm through one or more of 276 mood words; Spotify IDs for 68% | The song catalogue, and the mood words as labels for the song model |
| 2 | Last.fm API | [last.fm/api](https://www.last.fm/api), free API key | The tags listeners have attached to any song, with weights | The song model's input |
| 3 | EmpatheticDialogues | [GitHub](https://github.com/facebookresearch/EmpatheticDialogues), free download (CC BY-NC 4.0) | 24,850 short first-person situations, each labelled with one of 32 emotions | Training and testing the prompt model; a source of realistic test prompts |

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
- MuSe doesn't include each song's full tag list, only the mood word that found it, so we fetch full tags from the Last.fm API.
- MuSe's own mood scores come from a word list applied to the tags, not from people rating the music. 48% of songs share their exact score with another song, so we don't use these scores as labels.
- Mood words are unevenly used: median 290 songs each, and 91 of the 276 have fewer than 100 songs. Some words describe sound rather than mood (e.g. "crunchy", "slick") and get dropped.
- 1,863 rows are the same song listed twice with slightly different titles, so the catalogue needs de-duplicating.

## 9. Project Milestones

| Milestone | Main goals / tasks | Success criteria / key deliverable | Target completion |
|---|---|---|---|
| Milestone 1: Choose the moods | Finish the EDA on both datasets; get a Last.fm API key and fetch tags for a sample of songs; group similar mood words | ~10 named moods with definitions, and enough songs and prompts for each | |
| Milestone 2: Song model | Fetch Last.fm tags for the catalogue; hide the mood word; train the song model | Mood scores for every tagged song; the model gives sensible scores on songs it hasn't seen | |
| Milestone 3: Prompt model | Map EmpatheticDialogues' emotions onto our moods; train the prompt model | Mood scores for any prompt; accuracy per mood on held-out situations and our own prompts | |
| Milestone 4: Matching, evaluation and demo | Build the 50–100 prompt test set; match prompts to songs; answer the research questions; build the demo with Spotify playlist export | Top-10 results measured on the test set; a working demo; final report and presentation | |

### Risks and mitigations

| Risk | Mitigation |
|---|---|
| The song model copies the mood word instead of learning from other tags | Hide the mood word and its synonyms from the input; spot-check a sample |
| Many songs, especially recent or obscure ones, have few Last.fm tags | Measure tag coverage on a sample early; only recommend songs that have tags |
| MuSe's songs lean towards Last.fm's listeners (indie, rock, Western) and leave out songs with no clear mood | Report the genre spread; state it as a limitation |
| EmpatheticDialogues gives only one emotion per situation, and some emotions aren't music moods | Treat the label as "this mood is high" rather than "the others are zero", or add Jev scores; map or drop non-music emotions |
| Whether a song "matches my mood" is subjective | Build a test set of realistic prompts, and check the results with a small group of listeners |
