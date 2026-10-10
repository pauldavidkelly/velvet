# How Keybr-style tutors unlock keys and score

Research for [#5](https://github.com/pauldavidkelly/velvet/issues/5), part of map [#1](https://github.com/pauldavidkelly/velvet/issues/1).

**Question:** How do keybr.com and similar tutors order letters, decide when to unlock the next key, weight practice towards weak keys, generate words from a limited letter set, and calculate scores? What should Velvet adopt?

## Sources

All claims below come from source code read at these commits, unless marked otherwise.

| Source | What it is | Version read |
|---|---|---|
| [aradzie/keybr.com](https://github.com/aradzie/keybr.com) | keybr.com's own source (the main source) | `05a37bc` (2026-09-28) |
| [ranelpadon/ngram-type](https://github.com/ranelpadon/ngram-type) | ngram-type's source and README | `b31347e` (2023-08-01) |
| [monkeytypegame/monkeytype](https://github.com/monkeytypegame/monkeytype) | Monkeytype's source | `cbaa65c` (2026-10-08) |
| keybr's own UI text | The author's explanations, in `packages/keybr-intl/translations/en.json` | same as keybr |

**beybr.com is not a typing tutor.** On 2026-10-10, `https://beybr.com` has an expired TLS certificate and serves a parked Chinese clinic-advert iframe. A web search for "beybr" typing finds nothing but keybr. The map's "beybr.com" almost certainly means **keybr.com**, and everything about gradual unlocking and score below comes from keybr.

In this note, keybr paths are relative to `packages/`.

---

## 1. keybr.com: the guided lesson

The whole algorithm sits in `keybr-lesson/lib/guided.ts` (`GuidedLesson.update` and `GuidedLesson.generate`). The code re-runs `update(keyStatsMap)` from all stored results before each lesson (`page-practice/lib/practice/state/lesson-state.ts:63`). It keeps no "unlocked" state: the unlocked set is **derived** each time from per-key history.

### 1.1 Letter ordering

- **Default: language letter frequency**, most frequent first (`Letter.frequencyOrder`, `keybr-phonetic-model/lib/letter.ts:127`). The frequencies come from the language's phonetic model (a transition table built from a corpus). For English the order starts e, n, i, t, r, l, s, a, o, u… (`page-help/lib/english.ts` shows the frequencies).
- **Option "keyboard order"** (`lesson.guided.keyboardOrder`, `guided.ts:131-141`). Letters are sorted by a key-position weight first, then by frequency. The weight is 1 for home row, 2 for top row and 1000 for everything else (`keybr-keyboard/lib/keyboard.ts:122-129`, `weight()` at line 144). The UI text says this suits Colemak and Dvorak, and that on QWERTY it forces more pseudo-words because of the few home-row vowels (`setting.keyboardOrder.description`).

### 1.2 Starting set and the unlock rule

From `guided.ts:41-93`. The loop walks letters in order and decides whether each is included:

1. **Minimum alphabet of 6.** The first 6 letters are always included (`minSize = 6`).
2. **Manual extra letters.** `maxSize = 6 + round((N - 6) * alphabetSize)`, where `alphabetSize` is a 0–1 slider that defaults to 0. Letters up to `maxSize` are "forced" in.
3. **Confident keys stay in.** Any letter with `bestConfidence >= 1` is always included, so a key the user has ever mastered is never re-locked.
4. **The next letter unlocks when every included key is confident:**
   - default (`recoverKeys = false`): every included key has *ever* reached the target (`bestConfidence >= 1`);
   - "recover keys" option: every included key is *currently* at the target (`confidence >= 1`).

   The loop goes on to the next letter in the same pass, so several letters can unlock at once if the history already supports it.

The author describes the default rule this way: "you unlock a new key by raising only the focused key above the target speed", with recover keys being harder but making "forgetting old keys harder" (`settings.recoverKeys.description`).

### 1.3 Per-key confidence: speed only

- **Per lesson, per key:** `Histogram.from` (`keybr-textinput/lib/histogram.ts:52-93`) records `hitCount`, `missCount` and a mean `timeToType` for each code point. The **mean counts only keystrokes that weren't typos**. A key that needed a correction counts as a miss, and its time is left out. Samples faster than 40 ms or slower than 12 s are thrown out (`validateSample`). The first keystroke of a lesson is ignored (`stats.ts`, `steps.slice(1)`).
- **Across lessons:** `MutableKeyStats.append` (`keybr-result/lib/keystats.ts:131-155`) runs each key's per-lesson time through an **exponential moving average with α = 0.1** (`makeFilter(0.1)`, `keybr-math/lib/filter.ts`). The first sample sets the starting value. `timeToType` is the current smoothed value. `bestTimeToType` is the minimum smoothed value ever reached.
- **Confidence** (`keybr-lesson/lib/target.ts`): `confidence = targetTimeToType / timeToType`, where `targetTimeToType = 60000 / targetSpeedCPM`. **1.0 means "at target speed".** The default target is **175 CPM (35 WPM)**, adjustable from 75 to 750 (`keybr-lesson/lib/settings.ts:60`), so a key must average about 343 ms or less.
- **Accuracy does not gate unlocking.** `missCount` is stored but `Target.confidence` never reads it. Errors only matter indirectly: those keystrokes are left out of the time average, and they count against the lesson score.
- **No minimum sample count.** Because the EMA starts at the first raw sample, a key can reach `bestConfidence >= 1` in its very first lesson if that lesson was fast. The UI just marks keys with no data as "not calibrated" (`lesson.indicator.notCalibrated`).
- **Time to unlock.** Shown but not used to decide anything. `keybr-lesson/lib/learningrate.ts` fits a polynomial (linear, quadratic or cubic depending on sample count) to the last 30 smoothed speeds in the latest unbroken session. If R² ≥ 0.5 it shows "≈ N lessons remaining to unlock".

### 1.4 Weighting practice towards weak keys

After deciding what's included, keybr **focuses exactly one key**: the included key with the lowest confidence below 1 (`guided.ts:95-105`). That usually means the newest letter. Focus does two things:

- **Every generated word must contain the focused letter.** For dictionary words, `Dictionary.find` (`keybr-lesson/lib/dictionary.ts:35-44`) starts from the words containing that code point and then keeps those made only of included letters. For pseudo-words, `PrefixList.findPrefixes` (`keybr-phonetic-model/lib/phoneticmodel.ts`) starts each word from a short prefix of up to 3 letters that contains the focused letter.
- The author's tour text says the frequency of a new letter "is artificially increased so that it will be included in every generated word", and that the algorithm "can also artificially rearrange letter frequencies, placing an emphasis on the letters with the worst time-to-type metric" (`m_tour04`). In the current code the only weighting mechanism is the single focused key. There's no graded weighting across several weak keys.

### 1.5 Generating words from a limited letter set

`GuidedLesson.generate` (`guided.ts:110-161`) and `keybr-lesson/lib/text/*`:

- **"Natural words" (default on):** the built-in dictionary is filtered to words made only of included letters, which contain the focused letter and are longer than 2 characters, capped at 1000. If fewer than 15 match, the list is **topped up with pseudo-words**. Words are drawn uniformly at random from the list.
- **Pseudo-words:** a Markov chain over letter transitions (`TransitionTable`, `makePhoneticModel` in `phoneticmodel.ts`). At each step, candidate next letters outside the filter are dropped. Space (end of word) isn't allowed until 3 letters, and its weight is boosted by `1.3^length` to keep words short. Words are capped at about 10 letters, with up to 5 retries on dead ends. The result is pronounceable nonsense that keeps realistic bigrams.
- `uniqueWords` avoids the same word twice in a row. `generateFragment` builds about 100–200 characters (`lesson.length`), with an optional `repeatWords`.
- **Capitals and punctuation aren't unlocked.** They're added on top as probabilities (`lesson.capitals` and `lesson.punctuators`, both 0–1). `mangledWords` capitalises a word or wraps it with punctuation (`word,` `word.` `"word"` `word-word` …). The punctuation is chosen with weights `, 9  . 8  ! 2  ? 2`, and 1 each for `; : ' " -` (`Letter.punctuators`). The UI recommends raising these only once all letters reach the target.

### 1.6 Scoring

From `keybr-result/lib/result.ts:55-67`:

```
speed      = length / (time_ms / 1000) * 60          // characters per minute
accuracy   = (length - errors) / length
complexity = number of distinct characters typed      // histogram.complexity
score      = speed * complexity / (errors + 1) * (length / 50)
```

- Score grows linearly with speed, alphabet size and text length, and is divided by `errors + 1`, so one error halves the score. The author: "rewards a faster speed, longer text and a larger alphabet, but punishes the number of errors" (`page.highScores.content`).
- A result counts only if length ≥ 10, time ≥ 1 s, complexity ≥ 3 distinct characters, and every per-key sample is valid (`Result.validate`).
- WPM = CPM / 5 (`keybr-result/lib/speedunit.ts`).
- There's also a daily goal in minutes of practice (`keybr-lesson/lib/dailygoal.ts`, default 30).

### 1.7 Symbols and other text in keybr

The guided lesson covers **letters only**: it unlocks `model.letters`, which come from the language's phonetic model. Numbers and code are **separate, non-guided lesson types** with no unlocking (`keybr-lesson/lib/numbers.ts`, `code.ts`). keybr has frequency tables Velvet could reuse, though: `Letter.programming` gives symbol frequencies in code (`( ) { } " ' , ; [ ] : . = + - * / % | & ! ? < > _ # @ $ ^ ~ \ \``), with paired brackets at half weight, and `Letter.digits` follows Benford's law (`letter.ts:25-117`).

---

## 2. ngram-type

Source: `app.js` and `README.md`.

- **No per-key unlocking.** The user picks a source (top bigrams, trigrams, tetragrams, words or custom). Each lesson is `combination` items repeated `repetition` times, for example "the and the and the and" (`generatePhrases`, `app.js:261`).
- **Gate per phrase, not per key.** A phrase passes only if `rawWPM >= minimumWPM` **and** `accuracy >= minimumAccuracy`. The defaults are **40 WPM and 100 % accuracy**, set per source (`app.js:24-27`). If either check fails, the phrase restarts (`app.js:351-363`). WPM = (correct + wrong keystrokes) / 5 / minutes. Accuracy = correct / (correct + wrong).
- The average WPM is taken over passed phrases and reset each round, so the user can raise the threshold (README "Features").
- The README advises building difficulty by raising `combination` and lowering `repetition`, aiming for 100 % accuracy because "correcting mistakes is costly".
- **Worth taking:** an **accuracy gate next to speed**, and **repeated n-gram drills** for a key that is new or weak.

## 3. Monkeytype

Monkeytype is a typing test, not a tutor. It has no unlocking and no weak-key weighting in its core flow. Its metrics are a useful reference:

- `wpm = correctWordChars / 5 / minutes`, counting only characters in correctly typed words. `raw = (allCorrect + incorrect + extra) / 5 / minutes` (`frontend/src/ts/test/test-logic.ts:795-798`, `calculateWpm` in `frontend/src/ts/utils/numbers.ts:153`).
- Accuracy = correct keypress events / all keypress events (`getAccuracy`, `frontend/src/ts/test/events/stats.ts:550`). Corrected mistakes still count against it.
- **Consistency** = `kogasa(stddev / mean)` of per-second raw speed, where `kogasa(c) = 100 · (1 − tanh(c + c³/3 + c⁵/5))` (`packages/util/src/numbers.ts:80`, used at `test-logic.ts:760`). This gives a 0–100 score for rhythm. A "key consistency" works the same way on the gaps between keypresses.

---

## 4. What carries over to non-letter keys (layers, symbols, thumb keys)

keybr's model is keyed by **code point**, which fits Velvet's "app sees only characters" constraint. The key ideas carry over with these changes:

| keybr idea | Carries over? | Change needed for Velvet |
|---|---|---|
| Per-key EMA of time-to-type and confidence = target / time | Yes | Key the stats by **binding** (key + layer), worked out from the Keymap, not just by code point. A character reachable two ways needs a rule for which binding gets the credit. |
| Time-to-type for a key | Partly | A layer symbol costs a layer-key hold plus the key. Time from the previous character includes reaching for the layer thumb key, so layer keys will be slower. Consider a **lower target per layer** or a layer-switch allowance. Otherwise unlocking stalls on symbols. |
| Errors excluded from the time average; accuracy not gating | Rethink | For layers, a common failure is the **wrong layer**: right physical key, wrong character, such as `[` instead of `(`. That's an accuracy problem speed can't see. Consider gating on accuracy too (ngram-type style), and recording which wrong character was produced so the app can tell "wrong layer" apart from "wrong key". |
| Frequency ordering | Yes, per group | Letters: language frequency. Symbols: `Letter.programming` code frequencies. Digits: Benford. Within a layer, a keyboard-order variant ranked by finger or row effort from the Keymap is an alternative. |
| Always keep the first 6, unlock one at a time | Yes | Probably unlock by **group**: all letters on the base layer first, then a layer's keys a few at a time, then the layer key itself. |
| Focused key must appear in every word | Yes | For symbols, "words" are code snippets or symbol-wrapped words, generated as keybr's `mangledWords` does for punctuation, from a template set filtered to unlocked characters. |
| Dictionary first, Markov pseudo-words as fallback | Letters only | Symbols need a different generator: code-token templates or a symbol n-gram model filtered to unlocked characters. The ticket on practice text generation should cover this. |
| Capitals and punctuation as probability sliders | Rethink | For Velvet these *are* keys to teach (shift, home-row mods, the symbol layer), so they should go through the unlock path, not sit outside it. |
| Score = speed × distinct chars / (errors + 1) × length / 50 | Yes | Works the same over any character set. `complexity` naturally rewards having more layers unlocked. |
| Combos and home-row mods | Not covered | No tutor studied handles these. A combo shows up as one character from two simultaneous keys. A home-row mod held as a modifier shows up as a shortcut or a capital. These need their own detection and timing rules. |

---

## 5. Recommendations for Velvet

1. **Adopt keybr's derived-unlock model.** Store the per-key, per-lesson samples (hits, misses, mean time); don't store an "unlocked" flag. Re-derive the unlocked set and the focused key before each lesson. This makes Keymap changes and target changes cheap to handle: re-derive and you're done.
2. **Confidence = target time / EMA(time-to-type), α ≈ 0.1**, with 1.0 meaning mastered. Use keybr's default rule (`bestConfidence` ≥ 1 for every included key unlocks the next one). Offer the stricter "recover keys" rule as a setting.
3. **Add what keybr lacks:**
   - a **minimum sample count** per key (for example ≥ 2 lessons or ≥ N hits) before it can count as confident;
   - an **accuracy floor** per key (for example miss rate ≤ 5 %) next to speed. This matters for wrong-layer errors.
4. **Focus one weak key per lesson and put it in every word.** This is simple and proven. Later, consider sampling 2–3 weak keys in proportion to (1 − confidence).
5. **Words:** filter a dictionary to the unlocked letters and require the focused letter. Fall back to pseudo-words or n-gram drills (ngram-type style repetition) when fewer than about 15 words match. That happens early on, especially with home-row-first ordering on a custom layout.
6. **Score:** use keybr's formula as-is (`cpm × distinct / (errors + 1) × length / 50`). Show WPM (CPM / 5) and accuracy next to it. Monkeytype's `kogasa` consistency is a cheap optional extra.
7. **Default target:** 35 WPM (175 CPM) for base-layer letters, as keybr does. Give layer keys a separate, lower target or a layer-switch allowance. The right value is unknown, so it needs testing.

## 6. Open questions surfaced

- How should time-to-type and the target be set for layer keys, given the extra layer-key press? Is it one target per layer, or a fixed allowance per extra key held?
- How should a character be credited when the Keymap can produce it from more than one binding, such as `-` on two layers?
- How do we detect and teach **combos** and **home-row mods**, which keybr-style per-character timing can't see directly?
- How do we classify wrong-layer errors (right physical key, wrong layer) separately from wrong-key errors, and should they hold back unlocking?
- What text generator should drive symbol and code practice from a limited symbol set? keybr's code lessons don't unlock anything.
