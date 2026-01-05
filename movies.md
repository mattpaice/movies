# Project: Canonical Top 250 Films Dataset

A single Markdown spec Codex can follow to build the dataset in Python:

**Goal:** produce an Excel/CSV file with ~250 "best ever" films, combining rankings and flags from multiple sources.

## Target output

A single table with **one row per film** and columns:

- `title`
- `year`
- `imdb_id` (if available)
- `imdb_top250_rank` (integer, or blank if not in IMDb Top 250)
- `bfi_rank` (BFI / Sight & Sound 2022 rank, or blank)
- `oscars_best_picture_winner` (bool)
- `oscars_best_picture_nominee` (bool)
- `bafta_best_film_winner` (bool)
- `afi_100_rank` (AFI 100 Years…100 Movies rank, or blank)[1][2]
- `bbc_21st_century_rank` (BBC Culture "21st Century's 100 Greatest Films" rank, or blank)[3]
- `rt_top100_rank` (Rotten Tomatoes "Top 100 movies" style rank, or blank)[4][5]
- `other_critics_flags` (optional string, e.g. "listed_best_in_national_poll")[6]
- `composite_score` (float, computed)
- `composite_rank` (integer ranking by composite_score)

**Final deliverable:**
- `top_250_canon.csv`
- Optional: `top_250_canon.xlsx` (same data, Excel‑friendly).

## Source lists

Use **public/static datasets or scraped tables**, not paid APIs.

1. **IMDb Top 250**
   - Use an existing CSV of the IMDb Top 250 (Kaggle or GitHub).[7][8]
   - Required fields: title, year, rank, rating, and preferably IMDb ID.

2. **BFI / Sight & Sound 2022 "Greatest Films of All Time"**
   - Source: BFI's official list page for the 2022 poll.[9][10]
   - Extract at least top 100 (or full list if easy).
   - Fields: title, year, BFI rank.

3. **Academy Awards – Best Picture**
   - Use a CSV of Oscar Best Picture winners and nominees.[11][12]
   - Fields: year, film title, nominee/winner flag.
   - Create booleans: `oscars_best_picture_winner`, `oscars_best_picture_nominee`.

4. **BAFTA – Best Film**
   - Use a table or list of BAFTA Best Film winners (and nominees if available).
   - Fields: year, film title, winner flag.
   - Boolean: `bafta_best_film_winner`.

5. **AFI 100 Years…100 Movies**
   - Source: AFI's 100 greatest American movies list.[2][1]
   - Fields: AFI rank, title, year.

6. **BBC Culture: "21st Century's 100 Greatest Films"**
   - Source: BBC Culture critics' poll.[3]
   - Fields: BBC rank, title, year.

7. **Rotten Tomatoes "Top 100 movies" (or similar)**
   - Use a stable RT "Top 100 movies of all time" list (or similar canonical RT list).[5][4]
   - Fields: RT rank, title, year.

8. **Other critic/national polls (optional)**
   - Use Wikipedia's consolidated "List of films voted the best" page to flag films that were voted "best ever" in national or major polls.[6]
   - Create a simple boolean or string flag, e.g. `other_critics_flags = "voted_best_in_X"`.

## Normalisation and merging

High‑level steps Codex should implement in Python (likely using `pandas`):

1. **Load all source tables**
   - Read each CSV, or scrape HTML tables and convert to DataFrames.
   - Standardise column names immediately.

2. **Title/year standardisation**
   - Create normalised keys, e.g.:
     - `title_norm`: lowercased, stripped of punctuation/extra spaces.
     - Use `year` as a secondary key to disambiguate.
   - Where available, keep and propagate `imdb_id` from sources that include it (e.g. IMDb Top 250 dataset, some Oscars datasets).[7][11]

3. **Merge into master film list**
   - Start from the union of all titles appearing in:
     - IMDb Top 250
     - BFI list
     - Oscar Best Picture winners/nominees
     - BAFTA Best Film winners
     - AFI 100
     - BBC 21st Century list
     - Rotten Tomatoes top list
     - Optional "voted best" lists.[1][5][9][11][3][6][7]
   - Merge on `title_norm` + `year` (and `imdb_id` when available).

4. **Create rank/flag columns**
   - For each list:
     - If list is ranked (IMDb, BFI, AFI, BBC, RT): store that rank as an integer column.
     - If list is award‑based (Oscars, BAFTAs): store boolean flags for winner/nominee.

5. **Handle duplicates and conflicts**
   - If multiple rows refer to same film (same `imdb_id` or same `title_norm` + `year`), consolidate into one row and combine all rank/flag information.

## Composite scoring logic

Define a function to compute `composite_score` per film, based on what data is available. This is a suggested schema (adjustable):

- **Base from IMDb Top 250:**
  - If `imdb_top250_rank` present:
    - `score_imdb = 300 – imdb_top250_rank` (so rank 1 gets 299, rank 250 gets 50).
  - Else: `score_imdb = 0`.

- **BFI weight:**
  - If `bfi_rank` present:
    - `score_bfi = 300 – bfi_rank`.
  - Else: `score_bfi = 0`.

- **AFI weight:**
  - If `afi_100_rank` present:
    - `score_afi = 200 – afi_100_rank`.
  - Else: `score_afi = 0`.

- **BBC 21st‑century weight:**
  - If `bbc_21st_century_rank` present:
    - `score_bbc = 200 – bbc_21st_century_rank`.
  - Else: `score_bbc = 0`.

- **Rotten Tomatoes weight:**
  - If `rt_top100_rank` present:
    - `score_rt = 150 – rt_top100_rank`.
  - Else: `score_rt = 0`.

- **Awards and "voted best" bonuses:**
  - `+60` if `oscars_best_picture_winner` is true.[11]
  - `+25` if `oscars_best_picture_nominee` is true (non‑winners).[11]
  - `+40` if `bafta_best_film_winner` is true.
  - `+30` if film appears as "voted best ever" in any national/major poll from the Wikipedia compilation.[6]

**Composite score:**
```
Composite_score = score_imdb + score_bfi + score_afi + score_bbc + score_rt + award_bonuses
```

(Exact weights/offsets are configurable; Codex just needs to implement them cleanly.)

**Then:**
- Sort films descending by `composite_score`.
- Assign `composite_rank` as 1, 2, 3, … based on that sort.
- Keep only the top 250 rows to form the final "Top 250 canon".

## Output

- Save to `top_250_canon.csv` and (optionally) `top_250_canon.xlsx`.
- Ensure UTF‑8 encoding and safe column names (no spaces).
- Optional: also output a small text/JSON summary with counts (e.g. number of films that are Oscar winners, etc.).

**Codex should implement:**
- Data ingestion from these sources.
- Normalisation & merging.
- Composite scoring function.
- Export of the final 250‑row dataset.

## References

[1] https://prdaficalmjediwestussa.blob.core.windows.net/images/2019/08/movies100.pdf
[2] https://en.wikipedia.org/wiki/AFI's_100_Years...100_Movies
[3] https://www.bbc.com/culture/article/20160819-the-21st-centurys-100-greatest-films
[4] https://en.wikipedia.org/wiki/List_of_films_with_a_100%25_rating_on_Rotten_Tomatoes
[5] https://www.imdb.com/list/ls033935095/
[6] https://en.wikipedia.org/wiki/List_of_films_voted_the_best
[7] https://www.kaggle.com/datasets/rajugc/imdb-top-250-movies-dataset
[8] https://github.com/itiievskyi/IMDB-Top-250/blob/master/imdb_top_250.csv
[9] https://www.bfi.org.uk/news/revealed-results-2022-sight-sound-greatest-films-all-time-poll
[10] https://www.bfi.org.uk/greatest-films-all-time
[11] https://www.opendatabay.com/data/ai-ml/95aaaf6f-1f6d-4c76-9099-ff74340982bc
[12] https://www.reddit.com/r/datasets/comments/1n2nba5/a_clean_combined_dataset_of_all_academy_award/
