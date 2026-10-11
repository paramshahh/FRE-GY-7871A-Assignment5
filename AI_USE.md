# AI use disclosure

**Tools used:** Claude, run through Claude Code in a cloud session. No other AI tools.

---

**What I used it for.**

The implementation, essentially in full, and every data collection run. Claude wrote all the code:

* the data layer: `src/data_loader.py`, `src/prediction_markets.py`, `src/data_quality.py`,
  `audit_data.py`
* the text layer: the twitterapi.io client with its credit ledger, the GDELT client and the quality
  filters in `src/nlp_topics.py`
* the LSA engine in `src/nlp_svd_topics.py` (TF-IDF, truncated SVD, varimax, pole labelling,
  topic-conditioned sentiment, the covariance SVD)
* the econometrics in `src/granger_econometrics.py`
* the orchestration in `src/pipeline.py`
* the tables, the Streamlit dashboard, the report builder and the 54 unit tests

It ran every download (prices, FRED, Kalshi, Polymarket, PredictIt, GDELT and the paid tweet
collection). It wrote `BRAINSTORMING.md` and `DATA_AUDIT.md`, generated and executed
`assignment5_capstone.ipynb`, and drafted the report. The readings of the results were first put
by the model from the computed numbers:

* that issue sentiment on X does not lead the election odds at a daily horizon
* that the surviving Stage 2 results are consistent with slow pricing of regulatory risk
* that the Rigobon–Sack estimates are uninformative because the variance shift is too small

As in Assignment 1, I want to be plain about the division of labour: this was not AI producing a
first draft that I rewrote. It wrote the analysis and I directed it, set its constraints and
checked it at the points listed below.

**What I did myself.**

* **Real data.** When the cloud environment blocked every data host, the model began building a
  synthetic data generator. I stopped it and told it to find real sources instead. I then widened
  the environment's network allowlist so the APIs could be reached.
* **Data before analysis.** I asked for the data to be collected, cleaned and audited before any
  of the analysis was implemented, and the analysis was only built after that.
* **Topics.** I added homebuyer/housing affordability and healthcare affordability as topics of
  their own, which turned out to need a dedicated oversample to be measurable at all.
* **API budget.** I supplied the twitterapi.io key and set the budget: 100,000 of the account's
  115,000 credits, with 15,000 kept in reserve. The final spend was 93,450, and every call is
  recorded in a ledger.
* **Failures.** I asked for every failed task to be listed with how it was recovered, rather than
  retried silently. That review produced resumable GDELT caching, retries for empty tweet pages
  instead of caching them as complete, a guard on query length, and a second-source check on
  prices once Stooq blocked us.
* **Specification.** I revised the brief midway to an LSA/SVD design with a Democratic-sweep
  contract and a single-stock basket.
* **News source.** I chose GDELT over RSS feeds for news, since an RSS feed has no history to
  backtest on.
* **Topic review.** I reviewed the topic model (k = 30, varimax-rotated) before the daily series
  and the tests were built on it.
* **Report and repository.** I asked for the GDELT downloads to finish and for the news-salience
  check to be added. I had the report cut from 14 pages to 8, and moved the assignment into its own
  repository.

**What the model got wrong that had to be corrected.**

Eleven things surfaced during the build. The first seven are analysis or data errors that would
have changed what the report says. The last four are process failures.

1. **The first prediction-market series was timed against the wrong close.** Daily bars from
   Kalshi close at midnight ET and Polymarket's daily points land around 20:00 ET, both after the
   16:00 equity close, so a daily change in the probability overlapped two return days. This was
   replaced with hourly quotes, taking the last quote at or before 16:00 ET with its timestamp kept.
   Kalshi's median staleness is now zero hours and Polymarket's one hour.

2. **Polymarket first matched only 1–12 days of 155.** Its D and R quotes are stamped at different
   seconds, and they were joined on exact timestamps. Nothing failed: the series was simply mostly
   empty, and it was caught only because the audit counted the coverage.

3. **GDELT outage days were missed, then crashed the build.** The first outage rule flagged days
   with low article counts, but not days GDELT returned nothing for at all, nor a collapse at the
   very end of the sample. Undetected, those would have been read as quiet news days. Once the
   rule was fixed, a partially downloaded series (counts present, tone not yet) crashed the report
   build.

4. **The topic labelling had four separate errors.**
   * It forced component 0 to "General" on the textbook assumption that an uncentred LSA's first
     axis is the mean document. A test showed that axis can be a real issue, which would have
     silently deleted that topic.
   * It labelled poles by counting seed words, which put a single label on mixed
     housing-and-healthcare poles.
   * Linear topic weights let the tone of one issue leak into another's: in a test, housing
     sentiment read −0.32 where it should have been −0.95.
   * A hand-written stopword list let words like "as", "he" and "we" dominate several topics.

5. **Daily sentiment for thin topics was noise.** On days when a topic received a sliver of one
   tweet's loading, its sentiment read as a unanimous ±1. Energy had meaningful mass on only 55 of
   155 sessions. This was corrected by pooling over 5 sessions with a minimum-evidence rule.

6. **The first reading of a price anomaly was only half right.** On 26 June 2026 the S&P 500 index
   and SPY disagreed by 68bp, and the model concluded the index was stale and SPY correct. The
   check it then built showed that SPY, IVV and VOO also disagree with each other by 15–30bp a day
   that week. A sensitivity check followed: a median benchmark, or dropping the week, changes no
   conclusion.

7. **The README overstated a result.** It said the XLK response held in both halves of the sample.
   The robustness battery shows it rests mainly on the second half; only XLF and UNH hold in both.

8. **It did not stop to report the network block.** When every data host returned 403 in the
   first minutes, it mentioned this in one line and carried on building, starting on simulated
   data rather than asking. I had to interrupt it.

9. **It cached failed tweet pages as complete.** Three sessions returned zero tweets with no error,
   costing 15 credits each, and were saved as finished, so a re-run would never have retried them.
   All three came back full on retry.

10. **Avoidable or design-iteration spending on the paid API.** A 551-character query was rejected
    for exceeding the 512-character limit; that call was not billed. Separately, about 900 credits
    went on pilot pages that were discarded once it was clear "Top" ranking favoured large accounts.

11. **Two shell-level mistakes.**
    * A `pkill` command matched its own shell and killed it.
    * The first GDELT collector kept everything in memory until the end, so a restart lost about
      an hour of rate-limited downloading.

**A note on what I would and would not stand behind.**

* **Data.** I am confident in the data work. The prediction-market timing, the coverage and
  staleness checks, and the outage masking are verifiable, and `DATA_AUDIT.md` lets them be
  checked rather than taken on trust.
* **Stage 1.** The null result is the result I trust most among the tests: no sentiment or
  news-salience test survives the false-discovery correction, and the covariance-SVD factors do
  not beat their permutation null.
* **Stage 2.** I hold the Stage 2 findings more loosely. Five of 57 return tests pass every
  robustness check, but only XLF and UNH hold in both halves of the sample, and with one election
  cycle they are hypotheses rather than results.
* **The robustness rule.** When the formal battery passed two results (NVDA and META) that a hand
  check had called fragile, I kept the rule as written rather than tightening it after seeing what
  it let through, and the report says which results rest on one half of the sample.
* **The X sample.** It is politically engaged mid-size accounts, not the median voter, and nothing
  here should be read as public opinion.
