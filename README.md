
***Esperanto Morphological Tokenizer***

  

Introduction

  

- Esperanto Background

Esperanto is an agglutinative constructed international auxiliary language, boasting a unique and regular set of grammatical features along with the largest speaker base of any constructed language. While it has its quirks which we will soon note, its word structure is incredibly regular, fitting only a handful of common patterns, making it uniquely suited to morphological segmentation, tokenization, and subword modeling, the process of splitting up words based on their structure for use in natural language processing models.

Specifically, we are investigating the impact of this morphological tokenization on the output of English to Esperanto translations, using JoeyNMT, a simple neural machine translation toolkit built for educational purposes

- Subword Modeling

  

Subword modeling is the process of splitting up words or strings for use in language models to allow the model to represent any input text with a finite vocabulary. There are a few basic standard subword modeling methods we will explain here and compare to our Esperanto-specific morphological segmentation algorithm. Language models and neural networks do not understand language in terms of letters and words as we do, we must feed them vectors to train and run. As such, we must convert our words into subwords, and then assign each token to a vector that the model can manipulate.

  

The first is character-level subword modeling. This is where words and sentences are split up by their unique characters and assigned a vector, which is then fed into the model. This is typically used for languages like Mandarin Chinese with little internal word separation (most words are just one or two characters). In this case, the character-level subword modeling could approach word modeling, where each word is given its token. In the case of a language like English, it does the exact opposite, with a word like "snakes" being split into "s", "n", "a", "k", "e", and "s". The benefit of this approach is that no matter what input the model later receives (as long as those characters occur in the training data), it can be tokenized. A disadvantage of this is that individual characters in English or Esperanto rarely have meaning on their own, The disadvantage of this is that some semantic components of the words are lost, with an "s" at the beginning of a word and part of the root looking exactly like the "s" at the end indicating a plural.

  

The second is subword modeling, where words are broken down into parts of words (called subwords). Here a word like "snakes" might be "sn", "ake", or "s". There are various ways of splitting the words, and the most prominent is BPE (byte pair encoding) tokenization. In this method, the most common pairs of bytes are grouped based on frequency in the corpus until the desired number of unique groupings is reached, with groups of bytes being combined into even larger groups by their frequency. The advantage of this is that all words can be tokenized (as long as we can default to character-based encoding), and some semantic components that do not exist at the character level can be captured by the groupings. For example, BPE may lead to "es" as a token in English, which makes sense when it would be at the ends of words like "snakes" or "cakes", but not when in the middle or beginning of words like "corresponding". The morphological segmentation we will investigate here is a type of subword modeling, except we try to retain as many semantic components of the subwords as possible, first splitting off endings and affixes, then applying traditional BPE to the remaining roots.

- Why Esperanto is particularly suited to morphological tokenization

  

In many languages, including English, word structure is extremely complex and inconsistent, with countless exceptions and historical spellings to complicate things. Take two examples: "judges" and "forgave". The first is relatively easily split up into morphemes, the smallest unit of meaning in language, split into bound (cannot exist as words on their own) and unbound (can exist as single words) types. It could be split into "judge" + "s", with "judge" being unbound and "s" being bound. On the other hand "forgave" presents much more of a problem, as it is only one morpheme. However, if one were to try to write an algorithm to split words into morphemes, it's easy to see how it would be difficult to prevent the said algorithm from splitting the word into "for" + "gave" without thousands of exceptions. Furthermore, various endings and grammatical inconsistencies would complicate any algorithmic attempt even more. At that point, the algorithm would more or less be regurgitating manual segmentations of the most common words.

  

  

Esperanto, however, is much more amenable to the algorithmic segmentation of its vocabulary. Every word is either part of a simple and very limited set of commonly used irregular words that include prepositions, pronouns, and correlatives/tables (e.g. "tie"), or the word fits the forms of a verb, noun, adjective, or adverb. Nouns in "o" and adjectives end in "a", with a "j" appended if the noun is plural or the adjective describes a plural noun, and an "n" appended if the noun is in the accusative or the noun described by the adjective is in the accusative. For example, "bona besto" would be "good animal" in the nominative, "bonan beston" would be the same in the accusative, "bonaj bestoj" would be "good animals", and "bonajn bestojn" would be plural in the accusative. Verbs, too, are simple. Verbs in the infinitive always end in "i", in the present tense end in "as", in the future end in "os", in the past end in "is", "u" in the jussive (for issuing orders), and "us" for conditional (e.g. would want = "volus"). Due to this ending regularity, it is very simple to algorithmically strip endings to get to the core roots of each word.

  

Furthermore, Esperanto also employs a relatively small set of affixes that can be applied before or after roots, but always before endings. The simplest example is "mal-", indicating the opposite, which can be applied to any part of speech. "Nova" is new, so "malnova" is old. While this is a very simple example, the affix system is extensive yet simple and regular, allowing the expression of countless new words with only a small tweak. And because this set of affixes is relatively small and predefined (no new affixes are being created), it is much easier to algorithmically separate words into their morphemes, at least approximately.

  
  

Esperanto Tokenization Algorithm

  
  
  

- EsperantoWordSegmenter

- Moses

- enterify

- get a dictionary of every word, segment it

- collapsing large set of tags to ("ROOT","SPECIAL","ENDING","AFFIX")+ reasoning, also collapse the underlying

- forms of words, frequency, & counter.py:

- ('ROOT', 'ENDING') 8929

('SPECIAL',) 8194

('ROOT', 'AFFIX', 'ENDING') 788

('SPECIAL', 'ENDING') 578

('AFFIX', 'ROOT', 'ENDING') 534

('ENDING',) 237

('AFFIX', 'ROOT', 'AFFIX', 'ENDING') 126

('ROOT',) 107

('ROOT', 'AFFIX', 'AFFIX', 'ENDING') 23

('AFFIX', 'AFFIX', 'ROOT', 'ENDING') 4

('AFFIX', 'AFFIX', 'ROOT', 'AFFIX', 'ENDING') 4

('AFFIX', 'ROOT', 'AFFIX', 'AFFIX', 'ENDING') 2

('AFFIX', 'AFFIX', 'AFFIX', 'ENDING') 1

- recombine words into sentences

  

Experimental Setup

  

- joeynmt

- preprocessing tokenization

  

Results

  

Conclusion

  

- strange things I noticed in the EsperantoWordSegmenter

  

References

  

- github.com/tguinard/EsperantoWordSegmenter

- https://github.com/joeynmt/joeynmt

- https://web.archive.org/web/20230701203847/https://opus.nlpl.eu/Tatoeba.php
