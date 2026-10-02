# Customer Feedback Analysis & Automated Response

Imarticus Data Science Internship Assessment: Python Foundations & Gen AI

A retail company receives thousands of customer reviews a week. This project:

1. cleans the review data,
2. finds the *critical* reviews with plain Python rules (no machine learning) and the most common complaints,
3. uses a Generative AI model (Google Gemini) to draft personalised apology emails for the 3 most critical reviews.

**Dataset:** Women's E-Commerce Clothing Reviews (23,486 reviews, 1 to 5 star ratings).
**Notebook:** `Customer_Feedback_Analysis.ipynb`

---

## Approach

### 1. Data cleaning

| Problem | Decision | Why |
|---|---|---|
| `Unnamed: 0` column | Dropped | Leftover row number from when the file was saved. |
| Missing `Review Text` | **Row dropped** | No text means nothing to analyse or reply to. Filling it would invent data. |
| Missing `Title` | Filled with `""` | Optional field; the review body carries the content. |
| Missing `Division / Department / Class Name` | Filled with `"Unknown"` | Keeps the row; these are only labels. |
| Duplicate rows | Dropped (`drop_duplicates`) | Safety step so no review is counted twice. |

**Text cleaning** (`clean_text` function), applied to a *copy* of the review in a new `clean_text` column:

1. lowercase everything,
2. remove apostrophes first (`didn't` becomes `didnt`, so contractions stay one word),
3. replace every non-letter (digits, punctuation, emojis) with a space using a regular expression,
4. collapse repeated spaces.

The original `Review Text` is kept untouched. The cleaned version is only used for counting words. The Gen AI step receives the **original** text, because punctuation and capitalisation carry tone.

### 2. Rule-based logic (no machine learning)

* **Critical review:** a review with a star rating of **1 or 2** (`CRITICAL_MAX_RATING = 2`). This is a simple boolean filter. Critical reviews are saved to `outputs/critical_reviews.csv` as a hand-off file for the support team.
* **Complaint keywords:** the most common words in critical reviews, counted with `collections.Counter`. Two kinds of words are filtered out:
  * *stop words* (grammar words such as `the`, `was`, `didnt`),
  * *domain words* (words common in all clothing reviews, positive or negative, such as `dress`, `love`, `retailer`). These were chosen after reading a first pass of the output.
* **Complaint themes:** hand-written keyword lists group complaints into six themes: *Sizing & fit, Fabric & material, Look vs. photo, Defects & stitching, Price & value, Returns & service*. A review can belong to several themes. Common phrases such as *see through* and *runs small* are also counted.
* **Selecting the 3 reviews for outreach:** the rule is **1-star rating first, then the longest review text** (`word_count`). A long 1-star review gives the AI the most specific complaints to respond to.

### 3. Gen AI emails

For each selected review the notebook builds a prompt with:

* a **role** (customer support agent for an online women's clothing store),
* the **task** (a short, empathetic apology email),
* the customer's **original review**,
* **rules**: mention at least one concrete detail from the review, sound warm and human, stay under 120 words, do **not** promise refunds, discounts, compensation or deadlines, invite the customer to reply, include a subject line, and sign off as *Customer Care Team*.

Models are tried in order (`GEMINI_MODEL` environment variable, then `gemini-2.5-flash`), with one retry each. **If no API key or package is found, the notebook does not crash.** It falls back to a clearly labelled *offline template* (not Gen AI output), and every email in the results states which method produced it.

---

## How to run

### Requirements

* Python 3.9 or newer (or Google Colab)
* `pandas`, `matplotlib`, `ipython` (the rest is the standard library)
* `google-genai`, only needed for the Gen AI step

```bash
pip install pandas matplotlib ipython google-genai
```

### Steps

1. Download the dataset *Women's Clothing E-Commerce Reviews* (CSV).
2. Open the notebook and set the data location in the **Settings** cell (Step 0):
   ```python
   DATA_PATH = Path("/content/Womens Clothing E-Commerce Reviews.csv")   # change to your path
   ```
   The default is the Google Colab path. On your own machine use something like `Path("Womens Clothing E-Commerce Reviews.csv")`.
3. Supply the API key (see below).
4. Run all cells from top to bottom (**Runtime > Run all** in Colab, or **Run All** in Jupyter).

The other settings in that cell are `CRITICAL_MAX_RATING` (default 2) and `N_EMAILS` (default 3).

### Supplying the Gemini API key

The key is **never written in the code or committed to git.** The notebook reads it from the environment variable `GEMINI_API_KEY` (`GOOGLE_API_KEY` also works), or from a local `.env` file. Get a free key at <https://aistudio.google.com/apikey>, then use **one** of these:

**Option A: environment variable**

```bash
# macOS / Linux
export GEMINI_API_KEY="your-key-here"

# Windows PowerShell
$env:GEMINI_API_KEY="your-key-here"
```

**Option B: `.env` file** next to the notebook (add `.env` to `.gitignore`):

```
GEMINI_API_KEY=your-key-here
```

**Option C: Google Colab:** add the key under the key icon (**Secrets**) and run this in a cell before Step 8:

```python
import os
from google.colab import userdata
os.environ["GEMINI_API_KEY"] = userdata.get("GEMINI_API_KEY")
```

Optionally set `GEMINI_MODEL` to choose a different Gemini model, for example if a model name stops working.

---

## Outputs

All files are written to the `outputs/` folder:

| File | Contents |
|---|---|
| `critical_reviews.csv` | All 1 and 2 star reviews, for the support team |
| `emails.json` | The 3 drafted emails with rating, review text and generation source |
| `emails.md` | The same emails as a readable report |

The notebook also shows charts for ratings, top complaint keywords and complaint themes, and a calculated insights summary (Step 6).

---

## Limitations and next steps

* Keyword lists were chosen by hand and can miss synonyms or negation (for example "not too small").
* Reviews are English only.
* AI-drafted emails should be reviewed by a human before sending.
* At larger scale the email loop would need batching, retries and a queue.

---

## Optional: live web version

The folder `zara-womens-wear/` turns the same email rules into a small website. A customer picks a star rating, enters an email and a review, and receives a drafted reply by email. It reads the same `GEMINI_API_KEY` from the server's environment, plus Gmail SMTP settings (`SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `FROM_EMAIL`). See that folder's `README.md` for deployment on Render.
