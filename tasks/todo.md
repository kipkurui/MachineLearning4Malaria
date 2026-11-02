# Write-up + Africa vs Global North authorship comparison

## Context
Last session added HIV and an `ai_agentic` term list to the diffusion analysis, with
delay tables and 3 trend plots (`Plots/malaria_artificial_intelligence.png`,
`Plots/hiv_artificial_intelligence.png`, `Plots/malaria_deep_learning.png`).

Two new asks:
1. Turn the figures + insights into a written report.
2. Compare who is actually publishing this AI-diffusion research: African-affiliated
   authors vs Global North-affiliated authors, for the malaria/HIV + AI terms.

## Part 1: Country-of-authorship comparison (new analysis)
- [x] Batch-fetch abstracts for the 3 term+disease pairs already plotted
- [x] Reuse the pycountry-matching idea, fixed up (see Review) and generalized to
      disease+term
- [x] Classify each detected country as Africa / Global North / Other using two short
      hardcoded name lists
- [x] Compute % of papers per bucket for each of the 3 term+disease pairs, plot as a
      100%-stacked bar chart, save to `Plots/`
- [x] Add these as new notebook cells, same pattern as the existing country section

## Part 2: Write-up
- [x] Compile a report covering: the original diffusion-delay finding, the HIV extension,
      the AI/agentic-term findings (corrected - see Review), the 3 trend plots, and the new
      Africa-vs-Global-North authorship comparison
- [x] Publish it as a Claude Doc: [AI Diffusion to Malaria and HIV Research](https://claude.ai/artifact/R7ADHaYoh8EqKb8TJ38VUY)

## Notes / open questions
- "Global North" and "Africa" will be two fixed country-name lists I hardcode (standard
  groupings) — not a precise or contested economic definition, just enough to answer
  "who's publishing this."
- Scope stays on the 3 AI-term plots already generated last session (not re-deriving country
  data for the original `ml`/`other_tech` lists across all 4 diseases — that would be a much
  bigger re-fetch for a question the user didn't ask about).

## Review

### What changed
- **`paper_retriever()` fixed**: `efetch(..., rettype="gb")` no longer returns any content
  from PubMed (confirmed empty response, even single-id) - switched to `rettype="abstract"`,
  which still produces the "Author information:" / "PMID:" layout the parser expects.
- **New country-classification pipeline** (notebook + `Data/africa_vs_global_north*.csv`):
  fetches abstracts in batches (not one `efetch` per paper), classifies each paper's first
  author's country into Africa / Global North / Other via `pycountry` + a short alias list
  (real affiliations use "Tanzania"/"South Korea", not pycountry's formal ISO names), and
  joins against each paper's publication year.
- **New plot**: `Plots/africa_vs_global_north.png`, a 100%-stacked bar.
- **Write-up published**: [AI Diffusion to Malaria and HIV Research](https://claude.ai/artifact/R7ADHaYoh8EqKb8TJ38VUY)
  (Claude Doc) - background + HIV extension, AI/agentic findings (corrected), the 3 trend
  plots, the Africa-vs-Global-North comparison, and a methodology/caveats section.

### Bugs caught before they reached the write-up (worth knowing about)
An advisor review caught several real problems in my first pass that would have produced
wrong numbers:
1. **Country parsing picked the wrong country.** The original cell's logic iterated
   `pycountry.countries` and took whichever matched first (alphabetical-ish ISO order), not
   the country actually named in the affiliation - a US-or-UK affiliation with any other
   country word in the address got mis-credited. Fixed: match everything, take the **last**
   match in the first author's affiliation segment (addresses end with the country).
2. **Line-wrap bug**: PubMed abstracts wrap at ~80 characters, which was splitting
   "South Africa" and "United Kingdom" across a line break and losing the match -
   undercounting Africa specifically. Fixed by normalizing whitespace before matching.
2. **PubMed's `esearch` 9999-ID cap** (new since the notebook was first written) made
   "first published" dates wrong, not just missing, for high-volume terms - `get_count()`
   now reports them as unknown instead of silently reading off the wrong paper. This also
   fixed a bug in `clean_df()` where an unparseable PubMed date string ("2003 Feb-Apr")
   crashed the notebook outright (`errors='coerce'` added).
3. **"AI agents" + cancer acronym collision**: "AI" is oncology's standard abbreviation for
   *aromatase inhibitor*. The oldest "AI agents + cancer" hit I'd initially reported (2015)
   was actually about breast-cancer endocrine therapy costs, not artificial intelligence.
   "AI agents" and "autonomous agents" are now excluded from delay calculations and reported
   as counts only.
4. **Right-censoring**: terms coined in 2022-2024 (LLMs, generative AI) can't show more than
   a 2-4 year "delay" no matter how slowly they're actually diffusing - the write-up now
   states this explicitly rather than implying fast diffusion.

### Not done
- No population-normalized "adoption rate" (AI-papers / total-papers per region) - would
  need ~12 more affiliation-restricted PubMed queries. The write-up reports authorship
  *share* and says so explicitly, not "adoption rate."
