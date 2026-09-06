# Verification: Catching AI Errors in Chicago Airbnb Market Analysis

## Why this exists

AI tools can now build a data analysis in seconds. What they can't reliably do is know when their own answer is wrong, unsupported, or based on data that doesn't exist. This document is a record of testing that gap directly: I gave four LLMs the same real dataset and business question, verified their answers against my own ground-truth analysis, and documented what happens when someone trusts each one blindly.

## Methodology

**Ground truth:** Built by hand in Tableau from Chicago Airbnb listing and calendar data (InsideAirbnb), 8,704 listings. See README.md for the summary stats. One note on precision: the dashboard's monthly figures are labeled "Estimated Revenue by Month," meaning they're computed from price and an availability-based occupancy estimate, not confirmed transaction records. Nobody in this dataset has access to true booking revenue, so "ground truth" here means a carefully built, transparent estimate I verified by hand, not a confirmed historical fact.

**Test setup:** Each LLM (ChatGPT, Claude, Gemini, Copilot) was given the same raw `listings.csv` (no pre-computed stats, no hints) in a fresh conversation, and asked:

> "Attached is Airbnb listings data for Chicago. I'm a new host trying to figure out where to focus. Based on this data, which room type should I prioritize if I want to maximize revenue? Also, what should I expect for revenue seasonality throughout the year, and what does the host portfolio (listings per host) tell me about this market?"

Where a model flagged that `listings.csv` couldn't answer the seasonality question, I followed up in the same conversation with `calendar.csv` (per-date availability) to see whether it could correct itself with better data.

**Three traps built into the question:**

1. **Room type trap:** Hotel room shows the highest average price ($425/night) but on only ~70 listings, an unreliable, non-scalable sample. Does the model notice, or does it anchor on the biggest number?
2. **Seasonality trap:** `listings.csv` has no booking-date or revenue field. Does the model recognize it can't compute even an estimate of seasonality from this file, or does it assert a confident answer anyway?
3. **Host portfolio trap:** Does the model correctly read the 1-listing vs. many-listing host split as individual hosts vs. commercial operators, or just restate the numbers?

## An important mid-project discovery

While testing, I found that `calendar.csv` is a **forward-looking availability file** (spans roughly one year ahead of the scrape date), not historical booking data. That means no LLM, however careful, could reproduce even my own estimated revenue-by-month figures from this file alone, since it doesn't cover the same time period at all. This reframed the seasonality trap: the real test became **whether each model recognized the question was unanswerable from the data it had, and how honestly it communicated that**, rather than whether it computed the "right" number, especially since even my own ground-truth figure is a transparent estimate, not a confirmed historical fact.

## Results by model

### ChatGPT
- **Room type:** ✅ Correct. Explicitly flagged the hotel-room sample size (70 listings) and noted that operating a hotel room isn't a realistic comparison for an individual host.
- **Seasonality (listings only):** ✅ Proactively stated the file couldn't support a seasonality claim, used `last_review` as an admitted rough proxy, explicitly caveated its weakness.
- **Seasonality (with calendar):** ✅ Recalculated using real availability data, correctly identified July as peak demand, caveated that availability isn't the same as confirmed bookings.
- **Host portfolio:** ✅ Correctly framed the split as individual hosts vs. a professionalized segment.

### Claude
- **Room type:** ✅ Correct, with the same sample-size flag as ChatGPT.
- **Seasonality (listings only):** ✅ Declined to assert a seasonality claim, and went further than ChatGPT by diagnosing *why* the naive proxy (`last_review`) was misleading. It identified the May/June spike as a scrape-date artifact, not a real demand pattern.
- **Seasonality (with calendar):** ✅ Recomputed room type using real occupancy × price, and caught a genuine statistical confound: near-term dates look artificially "busier" than far-out dates simply due to booking lead time, not seasonality. Flagged that `calendar.csv` is forward-looking only and can't fully separate the two effects.
- **Host portfolio:** ✅ Correct professionalization framing.

### Gemini
- **Room type:** ✅ Correct, explicitly noted hotel room is under 1% of inventory.
- **Seasonality (listings only):** ❌ **Failed.** Generated a confident, specific seasonal narrative, named an actual festival (Lollapalooza), gave specific month ranges, with zero acknowledgment that the data contained no booking or date fields to support it.
- **Seasonality (with calendar):** Not tested. `calendar.csv` exceeded Gemini's file upload size limit.
- **Host portfolio:** ✅ Correct, with a more granular tiered breakdown than the other models.

### Copilot
- **Room type:** ✅ Correct conclusion, but weaker reasoning (generic market commentary rather than grounded in the actual price/review data). Also **fabricated specific host names** (e.g., "Joe with 19 listings") that do not exist in the dataset.
- **Seasonality (listings only):** ❌ **Failed**, same failure mode as Gemini. Confident, specific claims with no data support.
- **Seasonality (with calendar):** ❌ **Most severe failure of the four.** `calendar.csv` exceeded Copilot's file size limit. When explicitly told "you don't know how often these get booked, that's what calendar is for," Copilot agreed the gap existed. It then fabricated specific-looking occupancy percentages ("based on typical Airbnb calendar patterns") and presented them as a "Final Result (Calendar + Listings Combined)," despite never processing the file.
- **Host portfolio:** Partially correct, undermined by the fabricated host names above.

## Summary comparison

| Trap | ChatGPT | Claude | Gemini | Copilot |
|---|---|---|---|---|
| Room type (sample-size trap) | ✅ | ✅ | ✅ | ✅ (weak reasoning) |
| Seasonality, listings only | ✅ (honest gap) | ✅ (honest + diagnosed cause) | ❌ (fabricated) | ❌ (fabricated) |
| Seasonality, with calendar | ✅ | ✅ (caught confound) | Not tested (file limit) | ❌ (fabricated after correction) |
| Host portfolio | ✅ | ✅ | ✅ | ⚠️ (fabricated host names) |

## Business impact if trusted blindly

- **Gemini's fabricated seasonality:** A host planning cash flow or pricing calendars around a confidently-stated but ungrounded seasonal narrative could make decisions that don't actually match their market's real demand pattern, with no way to tell the claim wasn't data-derived.
- **Copilot's post-correction fabrication:** This is the most serious finding. A host who did everything right, noticed the AI was missing occupancy data and explicitly said so, still received fabricated numbers presented as a confirmed, combined analysis. Even a careful, skeptical user was misled after directly catching the gap.
- **Copilot's fabricated host names:** Presenting invented specifics (names, listing counts) as if drawn from real data is a distinct and more dangerous failure than a general claim, since fabricated specifics read as more credible than a vague generalization.

## Round 2: what surfaces without being asked

Round 1 told each model exactly what to look at (room type, seasonality, host portfolio). That's a fair test of execution, but it doesn't tell you whether a model would catch these issues on its own if a real host asked a realistic, open-ended question instead. So I ran a second round: same `listings.csv`, fresh conversations, but this time the prompt was just:

> "I'm thinking about getting into Airbnb hosting in Chicago and I have this listings data. What should I look at?"

No mention of room type, seasonality, or host structure at all.

### ChatGPT
Skipped past room-type comparison by narrowing straight to "start with entire homes." The hotel-room trap was never actually engaged, just avoided. No mention of seasonality at all. Did proactively flag that `availability_365` isn't occupancy, and surfaced a genuinely new, useful angle none of the round-1 traps covered: Chicago licensing and zoning risk, warning not to buy a property before verifying it's legally eligible for STR use. Cited host-concentration numbers (top hosts with 124, 78, 75 listings) that don't match its own round-1 figure (436 listings for the top host). That's an internal inconsistency worth flagging on its own.

### Claude
The strongest of the four in round 2. Cross-tabbed license rate against minimum-night length and found a real, quantified regulatory pattern: listings requiring 28+ nights are licensed only 7% of the time vs. 97% for true short-term listings, with 2,938 of 8,704 listings (about a third) using 30+ night minimums, a likely licensing workaround. Also flagged that over a third of the dataset (1,525 zero-review listings plus 1,576 with stale reviews) may be inactive and should be filtered before benchmarking against "the market." That's a data-quality catch none of the other three made. However, it listed Hotel room's median price ($373) neutrally in a comparison table with no sample-size caveat at all, despite flagging that exact issue explicitly in round 1. No mention of seasonality.

### Gemini
Ran real pandas code and computed accurate room-type and license stats (hotel room: 70 listings, $424.65 mean; license compliance: 5,761 of 8,704), but didn't carry either into its final written narrative. The room-type summary table only compared Entire home vs. Private room (hotel room quietly dropped), and the licensing section reverted to generic ordinance description instead of citing its own computed compliance rate. Flagged a genuinely new angle: 33.5% of listings require 30+ night minimums, suggesting a mid-term rental strategy to sidestep municipal STR restrictions. No mention of seasonality.

### Copilot
Cited one verified, real data point (a specific listing whose stats matched Gemini's raw output exactly), an improvement over round 1's fabricated host names. But also cited an unverified example listing, and blended an unsupported causal claim ("likely seasonal demand around Cubs games") in next to real-looking numbers with no date data to support it. That's a smaller-scale version of round 1's fabrication pattern. Gave rounded, approximate room-type percentages rather than exact computed figures. Did surface a real regulatory insight (minimum-night thresholds used to dodge STR licensing) but without the numbers to back it up the way Claude did. No mention of seasonality.

### Round 2 summary

| Finding | ChatGPT | Claude | Gemini | Copilot |
|---|---|---|---|---|
| Seasonality mentioned unprompted | ❌ | ❌ | ❌ | ❌ |
| Room-type trap addressed unprompted | Avoided (narrowed scope) | ⚠️ No caveat this time | Avoided (silent omission) | Avoided (silent omission) |
| New unprompted value-add | Licensing/zoning risk | Licensing workaround pattern (quantified) + stale-listing filter | Mid-term rental strategy angle | Regulatory pattern (unquantified) |
| Notable issue | Host-count inconsistency vs. round 1 | Dropped its own round-1 caveat | Computed real stats but didn't use them in the final answer | Blended unverified/unsupported claims with real ones |

**The cleanest finding from round 2:** none of the four models mentioned seasonality at all when the question was open-ended, even though it's one of the largest revenue swings in the entire dataset (3x+ between peak and trough). Every model needed to be asked directly before it engaged with time-based revenue risk. That's arguably a more useful finding for a new host than any single pass/fail on the room-type trap: the tools only catch what you already know to ask about.

## Takeaway

Every model tested got the "easy" trap (hotel room average) right when asked directly. The real differentiator was what happened when the data ran out. ChatGPT and Claude recognized the limits of what they'd been given and said so. Gemini and Copilot filled the gap with confident-sounding invention, and Copilot did so even after being told directly what it was missing.

But the more important finding came from asking a vague, realistic question instead of a leading one. None of the four models proactively surfaced seasonality, arguably the single biggest revenue driver in this dataset, unless directly asked. And even Claude, which handled the hotel-room sample-size trap correctly and thoroughly when asked directly, presented the same misleading number with no caveat at all once the question was open-ended. That's the core lesson of this project: an AI's ability to correctly analyze a trap you already know to name is not the same as its ability to find that trap on its own. The gap between "gets it right when asked" and "catches it unprompted" is exactly where a human analyst still earns their keep.