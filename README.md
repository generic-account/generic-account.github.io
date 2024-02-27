# Esperanto Morphological Tokenization

## Introduction
#### Esperanto Background
Esperanto is an agglutinative constructed international auxiliary language, boasting a unique and regular set of grammatical features along with the largest speaker base of any constructed language. While it has its quirks which we will soon note, its word structure is incredibly regular, fitting only a handful of common patterns, making it uniquely suited to morphological segmentation, tokenization, and subword modeling, the process of splitting up words based on their structure for use in natural language processing models. We investigate the impact of morphological tokenization on the translation quality of English to Esperanto translations, using Fairseq, a simple sequence modeling toolkit built by Facebook. [^1]

#### Subword Modeling

Subword modeling is the process of splitting up words or strings for use in language models to allow the model to represent any input text with a finite vocabulary. There are a few standard subword modeling methods we will explain here and compare to our Esperanto-specific morphological segmentation algorithm. Language models and neural networks do not understand language in terms of letters and words as we do, we must feed them vector-representations to train and run. As such, we must convert our words into subwords, and then assign each subword token to a vector that the model can manipulate.

The most basic scheme is character-level modeling, where words and sentences are split up by their unique characters which are each assigned a vector, which is then fed into the model. This is typically used for languages like Mandarin Chinese which have little internal word separation (most words are just one or two characters, and spacing is not used for word segmentation). In this case, the character-level subword modeling could approach word modeling, where each word is given its token. In the case of a language like English, it does the exact opposite, with a word like "snakes" being split into "s", "n", "a", "k", "e", and "s". The benefit of this approach is that no matter what input the model later receives (as long as those characters occur in the training data), it can be tokenized. A disadvantage of this is that individual characters in alphabetic languages like English or Esperanto rarely have meaning on their own.

Another scheme is subword modeling, where words are broken down into parts of words (called subwords). Take, for example, the word "snakes", which could be broken down as "sn", "ake", and "s". There are various ways of splitting the words, and the most prominent is BPE (byte pair encoding) tokenization. In this method, the most common pairs of bytes are grouped based on frequency in the corpus until the desired number of unique groupings is reached, with groups of bytes being combined into even larger groups by their frequency. The advantage of this is that all words can be tokenized (as long as we can default to a character-based encoding), and some semantic components that do not exist at the character level can be captured by the groupings. For example, BPE may lead to "es" as a token in English, which makes sense when it would be at the ends of words like "snakes" or "cakes", but not when in the middle or beginning of words like "corresponding". The morphological segmentation we will investigate here is a type of subword modeling, except we try to retain as many semantic components of the subwords as possible, first splitting off endings and affixes, then applying traditional BPE to the remaining roots.

#### Why Esperanto is particularly suited to morphological tokenization

In many languages, including English, word structure is extremely complex and inconsistent, with countless exceptions and historical spellings to complicate things. Take two examples: "judges" and "forgave". The first is relatively easily split up into morphemes, the smallest unit of meaning in language, split into bound (cannot exist as words on their own) and unbound (can exist as single words) types. It could be split into "judge" + "s", with "judge" being unbound and "s" being bound. On the other hand "forgave" presents much more of a problem, as it is only one morpheme. However, if one were to try to write an algorithm to split words into morphemes, it's easy to see how it would be difficult to prevent the said algorithm from splitting the word into "for" + "gave" without thousands of exceptions. Furthermore, various endings and grammatical inconsistencies would complicate any algorithmic attempt even more. At that point, the algorithm would more or less be regurgitating manual segmentations of the most common words.

Esperanto, however, is much more amenable to the algorithmic segmentation of its vocabulary. Every word is either part of a simple and very limited set of commonly used irregular words that include prepositions, pronouns, and correlatives/tables (e.g. "tie", meaning “there”), or the word fits the well-defined forms of a verb, noun, adjective, or adverb. All nouns end in "o", all adjectives end in "a", and both can have a "j" appended if the noun is plural or the adjective describes a plural noun, and an "n" appended if the noun is in the accusative or the noun described by the adjective is in the accusative. For example, "bona besto" would be "good animal" in the nominative, "bonan beston" would be the same in the accusative, "bonaj bestoj" would be "good animals", and "bonajn bestojn" would be plural in the accusative. Verbs, too, are simple. Verbs in the infinitive always end in "i", in the present tense end in "as", in the future end in "os", in the past end in "is", "u" in the jussive (for issuing orders), and "us" for conditional (e.g. would want = "volus"). Due to this ending regularity, it is very simple to algorithmically strip endings to get to the core roots of each word.

Furthermore, Esperanto also employs a relatively small set of affixes that can be applied before or after roots, but always before endings. For example, the affix “ig” means “to cause”, and it tends to come right before a verb ending, such as in “klarigi”, meaning “to make clear” (“klaro” = clear, “ig” = to make, “i” = verb ending). The simplest example is "mal-", indicating the opposite, which can be applied to any part of speech. "Nova" is new, so "malnova" is old. While this is a very simple example, the affix system is extensive yet simple and regular, allowing the expression of countless new words with only small tweaks. And because this set of affixes is relatively small and predefined (no new affixes are being created), it is much easier to algorithmically separate words into their morphemes, at least approximately.

---
## Esperanto Tokenization Algorithm

To tokenize the Esperanto sentences, the general pipeline is to first regularize the sentences, then to split the sentences into words to each be segmented. Each word will then be segmented and tagged, the unique tags will be reduced to a limited set, the segmentations will be simplified, root tags are tokenized with a BPE tokenizer, and special markers added.

#### Regularization
Regularizing the sentences is done using sacremoses, a commonly used text cleaning and segmentation tool.[^2] This separates words from special characters (like numbers and punctuation) by inserting spaces, and lowercases the text. For example, “Mi ŝatas hundidojn, kaj vi ankaŭ!” (“I like puppies, and you do to!”) will be converted to “mi ŝatas hundidojn , kaj vi ankaŭ !”. A further preprocessing step is done to change letters with circumflexes (“ĉ”, “ĝ”, “ĥ”, “ĵ”, and “ŝ”) to a standard form without circumflexes (“cx”, “gx”, “hx”, “jx”, and “sx”, respectively). The previous sentence would then be converted to “mi sxatas hundidojn , kaj vi ankaux !”.

#### Segmentation
The segmentation itself is done with EsperantoWordSegmenter, a tool that algorithmically segments and tags Esperanto words into morphemes using a Markov model and various hardcoded rules.[^3] The program takes in a training file of words, their segmented forms, and their tags, one per line (for example “krucxo krucx'o noun'nounEnding 1.0”, meaning “cross”), an input file (a list of Esperanto words), and a directory of known morphemes (special words and endings). The program then outputs a file of the segmented and tagged forms of each word. In using this program, we came across some words (namely numbers like “dektri”) that did not segment or tag properly, so these were added to the train.txt file with their correct segmentations and tags.

#### Tag Reduction
Tag reduction is a very important step, as the EsperantoWordSegmenter algorithm distinguishes from a number of grammatical tags that are not relevant to this project. The types of tags fall into four broad categories, standalone words (special words like correlatives, numbers, or pronouns), affixes (which the program divides into NounSuffix, NounPrefix, PeopleAnimalPrefix, AdjectiveSuffix, etc…), roots (including adjectives, nouns, and verbs), mid-word endings including “o”, a glue letter that serves to connect Esperanto words like in the case of “lampocilindro lamp'o'cilindr'o”, and endings which are split up into adjective endings, noun endings, pronoun endings, etc… The primary requirement for the tagging system is that it is unambiguous, such that a line of text can be segmented, tagged, and returned to its exact form with spaces inserted correctly. As such, we simply collapse the many tag distinctions of EsperantoWordSegmenter into only four types of tags, “SPECIAL”, “ROOT”, “AFFIX”, and “ENDING”. “SPECIAL” includes standalone words and punctuation, “ROOT” includes any root, “AFFIX” includes any prefix or suffix, and “ENDING” only includes the very small set of endings in Esperanto. To convert the tags from EsperantoWordSegmenter to this smaller set, we simply apply a mapping of the more complex tags to the distinct tags. Then, any repeated “ROOT” tags are collapsed into one “ROOT”, with (“ROOT”, ”AFFIX”, ”ROOT”) patterns also being reduced to “ROOT”, in order to aid the further segmentation of these subwords by the BPE tokenizer. Furthermore, aberrant patterns such as (“AFFIX”, “AFFIX”) or just one (“ENDING”,) are reduced to one “ROOT”, as these are often names and other unique words that did not segment properly. After this process is done, every word fits one of only a few segmentation patterns, seen below:

- ('SPECIAL',) &emsp; &emsp; ex: "mi" = "mi" (I or me)
- ('SPECIAL', 'ENDING') &emsp; &emsp; ex: "mia" = "mi_a" (my)
- ('ROOT',) &emsp; &emsp; ex: "lichtstein" = "lichtstein" (Lichtstein (a proper noun))
- ('ROOT', 'ENDING') &emsp; &emsp; ex: "bona" = "bon_a" (good)
- ('ROOT', 'AFFIX', 'ENDING') &emsp; &emsp; ex: "bonega" = "bon_eg_a" (great)
- ('AFFIX', 'ROOT', 'ENDING') &emsp; &emsp; ex: "malbona" = "mal_bon_a" (bad)
- ('AFFIX', 'ROOT', 'AFFIX', 'ENDING') &emsp; &emsp; ex: "malbonega" = "mal_bon_eg_a" (very bad)
- ('ROOT', 'AFFIX', 'AFFIX', 'ENDING') &emsp; &emsp; ex: "bonigilo" = "bon_ig_il_o" (an enhancer)
- ('AFFIX', 'AFFIX', 'ROOT', 'AFFIX', 'ENDING') &emsp; &emsp; ex: "malbonigilo" = "mal_bon_ig_il_o" (a detractor)
- ('AFFIX', 'ROOT', 'AFFIX', 'AFFIX', 'ENDING') &emsp; &emsp; ex: "miskomunistestro" = "mis_komun_ist_estr_o" (an anti-communist leader)
- ('AFFIX', 'AFFIX', 'ROOT', 'ENDING') &emsp; &emsp; ex: "fieksedzo" = "fi_eks_edz_o" (a wicked ex-husband)

Any tag labeled “ROOT” will get passed to a BPE tokenizer, to tokenize these core words in the same way the entire corpus would usually be tokenized. This allows for the model to gain some extra context about the content of these roots, and to reduce the number of tokens the model will have to deal with.

Ideally, every word would end in either an “ENDING” or “SPECIAL” tag, as then it would be easy to only put spaces after these tags, and not put them after any other tags. With this method, a sequence of tags (“SPECIAL”,”ROOT”,”ENDING”,”ROOT”,”ENDING”) corresponding to “mi havas demandojn” (meaning “I have questions”), could be separated into the words (“SPECIAL”,) for “mi”, (“ROOT”,”ENDING”) for “havas”, and (“ROOT”,”ENDING”) for “demandojn”.

However, this process introduces a problem regarding where spaces must be placed when reverting from a marked, tokenized sequence to the original raw text. This simple algorithm breaks down in two cases shown above, one where the word is tagged as (“SPECIAL, ”ENDING”), and one where the word is only tagged as (“ROOT”,). The first case is relatively easily fixed. While reforming the original words from a sequence of word segments and tags (e.g. (“ili’a”,”SPECIAL”,”ENDING”) -> “ilia”, meaning “their” [possessive]), we can iterate through each tag left to right, appending the word segment for every tag labeled “ROOT” or “AFFIX”. If the current tag is equal to “SPECIAL”, we check whether the next tag is “ENDING”, and if it is we do not add a space after the tag, but if it isn’t then we know a new word must have started and we do add a space. Hence, a sequence like (“SPECIAL”,”ENDING”) will get interpreted as one word, but (“SPECIAL”, “ROOT”, “ENDING”) will get correctly interpreted as two. The second case can be solved by passing words with the form (“ROOT”,) to the BPE tokenizer as the word, plus a single whitespace after it. This way, the tokenizer learns where to incorporate the whitespace by treating it as a normal character, no different from “a” or “b”.

At a large scale, it would be very computationally expensive to segment and tag every word in every sentence individually, as most words appear multiple times within the corpus. Hence, we first generate a file containing a set of every unique word, each word on a line, and we feed that into EsperantoWordSegmenter. In the simplification stage, as we scan through the corpus we slowly build up a dictionary of unique words and their segmentations and tags (e.g. {“sxatigi”: ((“sxat”, ”ig”, ”i”), (“ROOT”, ”AFFIX”, ”ENDING”))}, meaning “to begin liking” ), and if we later see that word, instead of simplifying its segmentation and tags again, we simply reference the dictionary and use that simplified segmentation and tags.

#### Combining the segmented and tagged forms and adding markers
Finally, the segmentations and tags for each sentence are collapsed into one, delimited by “点” and “分” (meaning dot (period) and divide (space) in Chinese, respectively). Every segment will be followed by a Chinese character representing its tag type (AFFIX, ENDING, and SPECIAL mapping to "接", "終", and "特", respectively). Roots, after being passed into the BPE tokenizer and segmented according to this algorithm, receive no marking. Roots or segments of roots followed by a space will be given the tag ”空" instead, to indicate where spaces should be placed in the un-segmentation process. This collapses the information previously stored in two lists (the list of the segments of a word, and a list of the corresponding tags) into one.

---

### Example Sentences

Below are three example sentences, with each step of the morphological tokenization process shown. The differences are bolded at each transition, except at the major steps between the cleaned sentences and the tagged and segmented ones, and between the tagged and segmented sentences and the marked sentences. I have also included the same sentences, as completely tokenized by the generic BPE tokenizer, for comparison.

| Esperanto Sentence                            | English Translation                                        | After Sacremoses Regularization                              | Cleaned and Circumflexes Removed                              |
| --------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------- |
| Ni provu ion!                                 | Let's try something!                                       | **n**i provu ion **!**                                               | ni provu ion !                                                |
| Mi devas enlitiĝi.                            | I have to go to sleep.                                     | **m**i devas enlitiĝi **.**                                          | mi devas enliti**gx**i .                                          |
| The knife is hard, the boiled potato is soft. | La tranĉilo estas malmola, la boligita terpomo estas mola. | **l**a tranĉilo estas malmola **,** la boligita terpomo estas mola **.** | la tran**cx**ilo estas malmola , la boligita terpomo estas mola . |

<br>

| Esperanto Sentence                            | Segmented and Tagged with EsperantoWordSegmenter                                                                                                                                                                                                                          |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ni provu ion!                                 | (ni,Pronoun), (prov_u,Verb_VerbEnding), (io_n,Table_TablePronounEnding)                                                                                                                                                                                                   |
| Mi devas enlitiĝi.                            | (mi,Pronoun), (dev_as,Verb_VerbEnding), (en_lit_igx_i,PrepPrefix_Noun_VerbSuffix_VerbEnding)                                                                                                                                                                              |
| The knife is hard, the boiled potato is soft. | (la,Article), (trancx_il_o,Verb_NounSuffix_NounEnding), (est_as,Verb_VerbEnding), (mal_mol_a,VerbPrefix_Adj_AdjEnding), (,), (la,Article), (bol_ig_it_a,Verb_Verb_TenseSuffix_AdjEnding), (ter_pom_o,Noun_Noun_NounEnding), (est_as,Verb_VerbEnding), mol_a,Adj_AdjEnding |

<br>

| Esperanto Sentence                            | Tags Simplified                                                                                                                                                                                                                           |
| --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ni provu ion!                                 | (ni,SPECIAL), (prov_u,**ROOT_ENDING**), (io_n,**SPECIAL_ENDING**), (!)                                                                                                                                                                            |
| Mi devas enlitiĝi.                            | (mi,**SPECIAL**), (dev_as,**ROOT_ENDING**), (en_lit_igx_i,**AFFIX_ROOT_AFFIX_ENDING**), (.)                                                                                                                                                           |
| The knife is hard, the boiled potato is soft. | (la,**SPECIAL**), (trancx_il_o,**ROOT_AFFIX_ENDING**), (est_as,**ROOT_ENDING**), (mal_mol_a,**AFFIX_ROOT_ENDING**), (,), (la,**SPECIAL**), (bol_ig_it_a,**ROOT_ROOT_AFFIX_ENDING**), (ter_pom_o,**ROOT_ROOT_ENDING**), (est_as,**ROOT_ENDING**), (mol_a,**ROOT_ENDING**), (.) |

<br>

| Esperanto Sentence                            | Roots Collapsed                                                                                                                                                                                                               |
| --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ni provu ion!                                 | (ni,SPECIAL), (prov_u,ROOT_ENDING), (io_n,SPECIAL_ENDING), (!)                                                                                                                                                                |
| Mi devas enlitiĝi.                            | (mi,SPECIAL), (dev_as,ROOT_ENDING), (en_lit_igx_i,AFFIX_ROOT_AFFIX_ENDING), (.)                                                                                                                                               |
| The knife is hard, the boiled potato is soft. | (la,SPECIAL), (trancx_il_o,ROOT_AFFIX_ENDING), (est_as,ROOT_ENDING), (mal_mol_a,AFFIX_ROOT_ENDING), (,), (la,SPECIAL), (**bolig**_it_a,**ROOT**_AFFIX_ENDING), (**terpom**_o,**ROOT**_ENDING), (est_as,ROOT_ENDING), (mol_a,ROOT_ENDING), (.) |

<br>

| Esperanto Sentence                            | With Markers                                                                          | With Roots BPE Tokenized                                                               | Fully BPE Tokenized (for comparison)                                            |
| --------------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Ni provu ion!                                 | ni特 prov u終 io n終 !空                                                                  | ni特 prov u終 io n終 !空                                                                   | ni終 provu終 ion終 !終                                                              |
| Mi devas enlitiĝi.                            | mi特 dev as終 en接 lit igx接 i終 .空                                                        | mi特 dev as終 en接 lit igx接 i終 .空                                                         | mi終 devas終 enliti gxi終 .終                                                       |
| The knife is hard, the boiled potato is soft. | la特 trancx il接 o終 est as終 mal接 mol a終 ,空 la特 bolig it接 a終 terpom o終 est as終 mol a終 .空 | la特 trancx il接 o終 est as終 mal接 mol a終 ,空 la特 **bol ig** it接 a終 terpom o終 est as終 mol a終 .空 | la終 trancxi lo終 estas終 mal mo la終 ,終 la終 bo li gita終 ter pomo終 estas終 mo la終 .終 |


---
## Experimental Setup

First, the Esperanto corpus is morphologically tokenized, according to the process described above. The corpus is also tokenized with a standard BPE tokenization algorithm to serve as a reference to compare our experimental model.

  

The building and training of the models themselves is done with Fairseq. The models are transformers, ubiquitous in NLP and other machine learning applications, which efficiently extract the most important pieces of context from their input data, in this instance Esperanto and English sentences. There are four models, two of which utilize the experimental morphological tokenizer, with the other two using the standard BPE tokenizer. For each type of tokenizer, one model is trained to translate from English to Esperanto, and the other translates from Esperanto to English.

To train the models, we use data from the English to Esperanto Tatoeba corpus, a dataset of about 300,000 parallel sentences (one sentence in English, and its translation in Esperanto), taken from the language learning site tatoeba.org.[^4] Users of the site can input their own new sentences and translations, which can sometimes introduce errors and unclean data (e.g. sentences in the Esperanto sentences file that are in English, Chinese, contain non-Esperanto characters, or are incorrect translations). One example of this is the English text "The bull is a strong animal" that was included among the Esperanto sentences. These out-of-place translations only account for an extremely tiny percentage of the entire data (there are only a handful of misplaced sentences), so they likely have almost no impact on the final results. It is also interesting to note that there are many repeated English sentences that have multiple different Esperanto translations, and vice-versa. This is useful as it helps the models generalize by seeing the many possible different translations.

As the initialization and training process is somewhat random, we trained and tested 3 models for each type of model. This allows us to measure and analyze the impacts of the tokenization algorithm itself while mitigating some of the randomness inherent to the process. However, this is a slight limitation of the project, as ideally, we would be able to run more tests to better filter out the effect of randomness.

---
## Results

#### Average BLEU Scores, Aggregated Data:

|| BPE Tokenizer       | Semantic Tokenizer|
|----------|---------------------|--------------------|
| **EN -> EO** | 51.69666667         | 51.42              |
| **EO -> EN** | 58.45               | 58.38              |

#### Raw Data:

| | Trial 1 (seed = 1000) BLEU Score | Trial 1 Training Time (sec) | Trial 1 Epochs | Trial 2 (seed = 1001) BLEU Score | Trial 2 Training Time (sec) | Trial 2 Epochs | Trial 3 (seed = 1002) BLEU Score | Trial 3 Training Time (sec) | Trial 3 Epochs | Average BLEU Score | Average Training Time (sec) | Average Epochs |  
| :-----------------------------: | :------------------------------: | :-------------------------: | :------------: | :------------------------------: | :-------------------------: | :------------: | :------------------------------: | :-------------------------: | :------------: | :----------------: | :-------------------------: | :------------: |  
| **EN -> EO BPE Tokenizer** | 51.65 | 2682.6 | 28 | 51.51 | 3554.6 | 37 | 51.93 | 3931.3 | 41 | 51.69666667 | 3389.5 | 35.33333333 |  
| **EN -> EO Semantic Tokenizer** | 51.52 | 3437.4 | 27 | 51.52 | 3293.8 | 31 | 51.22 | 3297.8 | 31 | 51.42 | 3343 | 29.66666667 |  
| **EO -> EN BPE Tokenizer** | 58.33 | 3931.8 | 41 | 58.53 | 6226.5 | 65 | 58.49 | 4762.1 | 50 | 58.45 | 4973.466667 | 52 |  
| **EO -> EN Semantic Tokenizer** | 58.47 | 6256.4 | 60 | 58.01 | 3868.7 | 37 | 58.66 | 5093.1 | 49 | 58.38 | 5072.733333 | 48.66666667 |

#### Analysis
After training and validating each type of model (English to Esperanto with morphological tokenizer, English to Esperanto with BPE tokenizer, Esperanto to Esperanto with morphological tokenizer, and Esperanto to English with BPE tokenizer), we found no statistically significant difference in the performance of the morphologically tokenized models compared to the BPE tokenized models, all outputting about the same BLEU score (a measure of how well the model's output translations match up with the official outputs from the Tatoeba corpus, with higher being better).[^5] For English to Esperanto (EN -> EO), both average BLEU scores were about 51 plus or minus less than one BLEU, and for Esperanto to English (EO -> EN), both average scores were about 58, plus or minus less than one BLEU. It is interesting to note that our semantic tokenizer performed very slightly worse in these tests, however this is about what one would expect from the random influences of which seeds we used and is not statistically significant.

 ---
## Conclusion

The results of this experiment hint at the robustness of the BPE tokenization algorithm, the unintuitiveness of tokenization and how it impacts models, but also the need for more study in this area. With a simple algorithm that requires little to no specific tuning for the target language, BPE performs similarly to an algorithm specially designed to fit the morphology of Esperanto, reinforcing the validity of this often-used algorithm in NLP.

It seems quite intuitive that developing an algorithm to specifically morphologically segment one particular language would lead to models being able to better understand grammatical structures and context downstream, but it did not play out like this in our experiment. In the world of NLP tokenization, our intuition goes out the window and we must rely on other metrics to guide the way, not just our preconceptions. This also calls into question what other ideas, accepted as “intuitive” and made widespread, are in fact not optimal and should be investigated further.

There are countless other variables that could impact the results of this experiment, from the language pair to the dataset size and the training time, plus a whole range of model hyperparameters. While the benefit of morphological tokenization to Esperanto transformer models may be relatively small, it is entirely possible that morphological tokenization would yield better results when applied to English or German, even though it would be harder to program such tokenizers for less regular languages. Additionally, it’s likely that the morphological tokenization has some effect, good or bad, on the final BLEU scores, but that we were not able to uncover it with our limited resources. Perhaps training more models or training them for longer would unveil some different BLEU upper-bound. In addition to this there are many hyperparameters, high-level knobs and levers we can tweak when training the models, which may affect the effectiveness of morphological tokenization. Without more experimentation, it’s not possible to know.

One particular weakness of our algorithm is that we only extract a small amount of the easily accessible semantic content of words, limited to mostly stripping off affixes and endings along with identifying special words and letting a BPE tokenizer handle the rest. In reality, these simple patterns are quickly picked up by a BPE tokenizer, which we can see in the few example sentences above. The BPE tokenizer learned pronouns, some endings, and even an affix ("mal", meaning "not"). More complex words or very irregular words are exactly those which we would likely get the most benefit out of by morphologically tokenizing, but our algorithm collapses them all to "ROOT" (in the case of very irregular words), or combines all "ROOT" tags into one single "ROOT" for the BPE tokenization step (in the case of large compound words). One specific future experiment or series of experiments could focus on these difficult compound words and irregular forms, maximally accurately morphologically segmenting them and making minimal use of the BPE tokenizer.

---
We would like to thank Marco Cognetta for his amazing help and guidance throughout this project. Without him, this likely would not have been possible.

## References
[^1]: [fairseq: A Fast, Extensible Toolkit for Sequence Modeling](https://aclanthology.org/N19-4009)
[^2]:[https://github.com/hplt-project/sacremoses](https://github.com/hplt-project/sacremoses)
[^3]:[https://github.com/tguinard/EsperantoWordSegmenter](https://github.com/tguinard/EsperantoWordSegmenter)
[^4]:[https://opus.nlpl.eu/Tatoeba/en&eo/v2023-04-12/Tatoeba]([https://opus.nlpl.eu/Tatoeba/en&eo/v2023-04-12/Tatoeba)
[^5]: [BLEU: a Method for Automatic Evaluation of Machine Translation](https://aclanthology.org/P02-1040.pdf)
