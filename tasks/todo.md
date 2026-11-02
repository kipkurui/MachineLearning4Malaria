# OpenAlex rebuild: Africa vs Global North, deeper + latest snapshot

## Context
Previous Africa-vs-Global-North analysis regex-parsed PubMed abstract text for
countries (fragile, needed several bug fixes). User now wants: (1) the latest data
snapshot, (2) a deeper analysis. Agreed approach: switch to the OpenAlex API, which
gives structured ISO2 country codes per author directly — no regex, no pycountry
aliases — plus citations, and cheap aggregate counts via `group_by` for denominators.
Confirmed via live probe: `filter=title_and_abstract.search:...` does AND search,
`authorships[].countries` gives ISO2 codes directly, `group_by=authorships.countries`
gives per-country totals without fetching every record.

Enrichments the user selected, all in scope for this pass:
- Africa share over time (not just one snapshot)
- Country-level breakdown within Africa (not one bucket)
- International co-authorship (Africa+Global-North collab vs Africa-only)
- True adoption rate (AI papers ÷ all papers, per region) — previously skipped
- Citation-impact comparison (Africa-led vs Global-North-led papers)
- Broaden to TB, alongside malaria and HIV

## Before starting
- [x] Get a free OpenAlex API key (raises the shared daily request budget from
      $0.10 to $1.00/day — we need ~50-100 requests). Requires POSTing the user's
      email to OpenAlex and them clicking a confirmation link. **Confirming with
      user before doing this** (external service + their email). User confirmed,
      key requested and activated.

## Part 1: Fetch pipeline (OpenAlex)
- [x] `fetch_term_disease(term, disease)`: paginated fetch via
      `title_and_abstract.search` filter, term × disease pairs:
      {artificial intelligence, deep learning} × {malaria, HIV, TB}
- [x] Per paper, keep: year, cited_by_count, first-author country code + raw
      affiliation text, full set of distinct author countries (for collaboration)
- [x] Save to `Data/openalex_ai_diffusion_detail.csv` (per-paper) +
      `Data/openalex_disease_denominators.csv` (exact per-region disease totals)

## Part 2: Denominators for adoption rate
- [x] Switched from a `group_by` sum (confirmed to double-count papers with >1
      co-author in the same region — inflated Global North ~1.46x vs Africa
      ~1.25x) to an OR-combined country filter's exact `meta.count`, for both
      numerator and denominator
- [x] Noted in the notebook: this is "any co-author in region" (presence),
      different from the first-author-based "authorship share" metric

## Part 3: Analysis + plots (dataviz skill, same palette as before)
- [x] Updated Africa/Global North authorship-share stacked bar (6 pairs now, was 3)
- [x] Africa-share-over-time trend (year bins, with 95% Wilson CI bands), split by
      disease, both full corpus and PubMed-indexed-only
- [x] Top African vs top Global-North countries by paper count (bar)
- [x] Collaboration breakdown: Africa-only / Global-North-only / collab / other (bar)
- [x] Citation comparison by first-author region, restricted to a fixed 2020-2022
      maturity window (not "all pre-2023"), medians only
- [x] Adoption-rate comparison by region, per disease/term, with two-proportion
      z-tests marking which differences are actually distinguishable

## Part 4: Notebook + write-up
- [x] Added new notebook section for the OpenAlex pipeline + plots (existing
      PubMed delay/trend analysis untouched)
- [ ] Update the existing Claude Doc with the new findings and plots (same doc,
      not a new one) — confirm with user this is wanted before publishing

## Notes / open questions
- OpenAlex's corpus is broader than PubMed's (indexes conference proceedings etc.),
  so raw counts will differ from the old PubMed numbers — that's expected, not a bug.
- Deep learning × HIV/TB are new pairs never fetched before (old pipeline only did
  deep learning × malaria) — first-ever data point for those, not a re-derivation.

## Review

### What changed
- **New OpenAlex pipeline** (notebook section "Deeper dive: OpenAlex, latest snapshot
  (Nov 2026)", + `Data/openalex_*.csv`): replaces regex-parsed PubMed text with
  OpenAlex's structured per-author ISO2 country codes for 6 term×disease pairs
  ({artificial intelligence, deep learning} × {malaria, HIV, tuberculosis}), current
  through Nov 2026.
- **6 new analyses**: authorship-share summary (full corpus + PubMed-indexed-only, side
  by side), a time trend with confidence intervals, top countries within each region,
  international co-authorship, a true adoption rate (AI papers ÷ all disease papers per
  region), and a citation comparison.
- **6 new plots** in `Plots/`: `openalex_summary_stacked.png`, `openalex_trend.png`,
  `openalex_top_countries.png`, `openalex_collaboration.png`, `openalex_adoption_rate.png`.
- Got a free OpenAlex API key (sent to `.env`, gitignored, read via `Authorization: Bearer`
  header — never as a URL param, so it can't leak into a logged URL or saved notebook output).

### Bugs/biases caught before they reached any conclusion (2 advisor rounds)
This was a much higher-stakes round than last session's — several of the headline numbers
the first pass produced were real artifacts, not findings:
1. **Adoption-rate denominator mismatch**: numerator (AI papers) was almost all 2018+, but
   the first denominator (all-time disease papers) spanned decades — inflated the apparent
   "Africa adopts AI faster" gap. Fixed by windowing both to 2015-2026.
2. **`group_by` double-counts**: summing OpenAlex's per-country `group_by` buckets counts a
   paper once per co-author country, and Global North papers (more international
   co-authorship) were inflated more (~1.46x) than Africa's (~1.25x) — would have biased
   the region comparison. Fixed by switching to an OR-combined country filter's exact
   `meta.count` for both numerator and denominator.
3. **Corpus shift unexplained**: OpenAlex's full corpus initially looked wildly different
   from the published doc's PubMed-based numbers (malaria+AI: ~parity vs the doc's "~2.5x
   Global North"). Validated by rebuilding the PubMed-indexed-only subset, which reproduces
   the original finding's direction — the difference is real and explained (OpenAlex's
   non-PubMed venues, e.g. CS conference proceedings, have a different regional mix and a
   much higher missing-affiliation rate) — now both corpora are shown side by side.
4. **Trend could have been a venue-mix artifact**: the "Africa's share rising in 2023+"
   finding was checked against the PubMed-only subset specifically because 2023+ is
   dominated by non-PubMed venues; it holds (z=2.0-2.5) for malaria/TB, confirmed with
   two-proportion z-tests, not just eyeballed off a bar chart.
5. **Missing-affiliation bias direction, checked not assumed**: ~30% "Missing" (vs 5-9%
   with the old PubMed-only pipeline) turned out to be mostly OpenAlex's own
   institution-matcher failing on smaller/regional institutions, not a geocoding gap -
   confirmed by eyeball-checking raw affiliation text, then recovered ~28% of them with the
   same regex fallback as the PubMed section.
6. **NaN round-trip bug** (caught by my own dry-run validation, not the advisor): a
   co-author-country column with no data serializes to an empty string, but `pd.read_csv`
   reads that back as `NaN` - a naive `str(x).split(";")` on real NaN produces the literal
   string `"nan"`, which then got misclassified as region "Other" instead of "Missing".
   Fixed with an explicit `pd.isna()` guard. Confirmed via a dry run of the exact ported
   notebook code (not just the scratch scripts) against the cached CSVs before trusting it.

### Headline findings (with the caveats above already applied)
- **Adoption rate** (AI-term papers ÷ all disease papers, same region): Africa is higher
  for "artificial intelligence" across all 3 diseases, but the gap is small and only
  malaria/TB clear p<0.05; "deep learning" specifically flips to Global-North-favoring or
  parity. Framing matters - "AI" picks up more general/review content than "deep learning".
- **Trend**: Africa's first-author share roughly doubled from 2020-2022 to 2023+ for
  malaria and TB (statistically distinguishable, holds in the PubMed-only subset too); HIV's
  rise is weaker and concentrated in non-PubMed venues.
- **Collaboration**: true Africa+Global-North co-authorship is modest (~5-8% of papers);
  most non-Global-North-only output is from elsewhere entirely (India, China, Brazil), not
  from Africa or from Africa-GN partnerships.
- **Citations**: no consistent region gap once restricted to a fixed 2020-2022 maturity
  window - mixed direction across the 3 diseases, small samples (n=13-26 for Africa).

### Not done
- The write-up (Claude Doc) has not been updated yet with these new findings - checking
  with the user first since publishing it is a separate, visible action.
