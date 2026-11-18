# tweets-to-news

Extract news article links from a file of tweets and retrieve the corresponding article texts for text analysis.

## Project overview

This project uses Python to identify news articles shared in tweets, retrieve their original web addresses and extract their text.

The notebook processes the links in several steps: it extracts URLs from tweets, follows shortened links, identifies news websites, and retrieves each distinct article. When a website refuses automated access, it also checks whether a copy of the article is available in the Internet Archive (Wayback Machine).

The resulting dataset contains the article URLs, titles, publication dates, extracted texts and information about the retrieval process. It can then be used for further text analysis.

## 1. Results on the sample

The sample contains 36 public tweets, mostly in French. In the test run, the notebook identified 36 links.

| Result | Number |
|---|---:|
| Tweets | 36 |
| Links found | 36 |
| Links matching the news website list | 29 |
| Distinct news links processed | 29 |
| Articles retrieved from the original websites | 10 |
| Articles retrieved from the Internet Archive | 13 |
| Articles successfully retrieved in total | 23 |
| Articles not retrieved | 6 |

Of the 29 distinct news links, 23 articles were successfully retrieved: 10 from the original websites and 13 from archived copies.

The 6 remaining articles could not be retrieved. In the sample run, the original websites refused access and the Internet Archive had no available copy. All six were published in the two days before the test and had not been archived yet.

The results also show that the Internet Archive can be useful for recovering articles from websites that refuse automated access, including Le Monde, RFI, Le Parisien and Le Point.

This work describes a particular sample and test run. They do not represent the expected retrieval rate for every collection of tweets.

## 2. Installation and execution

Install the required Python libraries:

```bash
pip install -r requirements.txt
```

Open the notebook:

```bash
jupyter notebook tweets_to_news_articles.ipynb
```

Run the cells from top to bottom. The notebook explains each step and displays the intermediate tables and results.

## 3. Input data

The input is a CSV file containing tweets, with at least two columns:

| Column | Description |
|---|---|
| `tweet_id` | Unique identifier of the tweet |
| `text` | Text of the tweet |

The file paths and other settings are defined in the setup section of the notebook.

## 4. Extract URLs from tweets

The notebook uses a regular expression to find URLs beginning with `http://` or `https://` in the text of each tweet.

It also removes punctuation at the end of a URL and keeps each link only once within a tweet.

## 5. Clean URLs and identify news websites

The same article can be shared with different tracking information in its URL, such as `utm_source` or `fbclid`. The notebook removes these tracking parameters and URL fragments so that different versions of the same article address can be treated as one.

It then extracts the website domain and compares it with the list of news websites in `news_domains.txt`.

The list contains 42 domains in the sample run. It can be updated to add or remove websites. Subdomains are also recognised: for example, `sport.francetvinfo.fr` matches `francetvinfo.fr`.

Links to websites that are not in the list are kept in the results, with `is_news = False`. This makes it possible to inspect other domains and add news websites that may have been missed.

## 6. Resolve shortened links

Tweet links often pass through several addresses before reaching the original article. For example, a link may first use the Twitter/X shortener `t.co` and then a news website's own shortener, such as `go.rfi.fr` or `l.lefigaro.fr`.

The notebook follows this chain of redirects to identify the final address. It handles both standard HTTP redirects and redirects implemented within a webpage, including HTML meta refresh tags and JavaScript redirects. It also checks certain page metadata when a shortener returns an almost empty page.

Each original URL is resolved only once, even if it appears in several tweets.

If a website refuses the request or returns an HTTP error, the notebook keeps the address reached and records the corresponding status. This can still help identify the website, even when the page cannot be accessed.

Two links using the `ebx.sh` shortener remained at that address in the sample run and could not be resolved to their final article addresses.

## 7. Build the links dataset

The notebook creates `output/links.csv`, with one row for each link found in a tweet.

| Column | Description |
|---|---|
| `tweet_id` | Identifier of the tweet containing the link |
| `raw_url` | Original link as it appears in the tweet |
| `final_url` | Address reached after following redirects, with tracking information removed |
| `resolve_status` | `ok`, or the reason the link could not be followed successfully (`http_404`, `http_403`, `Timeout`, `too_many_redirects`, etc.) |
| `domain` | Domain of the final address |
| `is_news` | `True` if the domain belongs to a website listed in `news_domains.txt` |

The `resolve_status` column helps distinguish links that were successfully followed from those that returned an error or could not be resolved.

## 8. Retrieve article texts

The notebook retrieves each distinct link classified as a news website only once.

It uses [Trafilatura](https://trafilatura.readthedocs.io) to extract the article title, publication date and text from the webpage. Trafilatura helps separate the main article from other page elements, such as menus, advertisements and footers.

Before keeping the extracted text, the notebook removes certain boilerplate lines and checks that the text meets a minimum character length defined in the settings.

### Internet Archive

When a website refuses automated access or the original page cannot be downloaded, the notebook checks whether a copy is available in the Internet Archive (Wayback Machine).

If an archived copy can be retrieved, the notebook extracts the article text from that copy. It does not attempt to bypass the website's access restrictions.

### Article retrieval status

Each article receives a status indicating the result of the retrieval process:

| Status | Description |
|---|---|
| `ok` | Article text successfully extracted from the original website |
| `ok_archive` | Article text successfully extracted from an archived copy |
| `no_text` | No article text could be extracted, for example from a video page or homepage |
| `http_404` | Page not found |
| `http_403` | Access refused |
| Other values | Other errors, such as `Timeout` or `http_402` |

The `paywall` column indicates whether the extracted text contains a message associated with a subscription requirement. An article flagged this way may contain only the freely available part of the text.

The `archive` column records the result of the Internet Archive search. For example, `used` means that an archived copy was retrieved and used; `no_copy` means that no available copy was found. Other values can indicate errors when accessing the archive.

### Output: articles.csv

The notebook saves the extracted articles in `output/articles.csv`, with one row per distinct news link.

| Column | Description |
|---|---|
| `url` | Cleaned address of the article |
| `status` | Result of the retrieval process |
| `title` | Article title, when available |
| `date` | Publication date, when available |
| `n_characters` | Number of characters in the extracted text |
| `paywall` | `True` if a subscription-related message was detected in the extracted text |
| `archive` | Result of the Internet Archive search |
| `text` | Extracted article text |

When the notebook is run again, previously retrieved articles are kept, while articles that failed are tried again. This avoids downloading articles that have already been successfully extracted.

## 9. Check the retrieval results

The notebook provides summary tables to assess the results, including:

- The total number of links found in the tweets.
- The number of links pointing to websites in the news domain list.
- The most frequent domains.
- The number of distinct news articles identified.
- The number of articles retrieved from the original websites and from the Internet Archive.
- The number of articles that could not be retrieved.
- The number of articles flagged as potentially behind a paywall.

It also displays the first lines of an extracted article to allow a quick check of the retrieved text.

## 10. First look at the articles

Once the article texts have been retrieved, the notebook provides a first descriptive overview of the corpus.

First, it counts the words in each article and compares the average article length across news websites.

It then checks how many articles mention a selection of news topics, including Ukraine, Russia, the United States, Trump, Gaza, Israel, Palestine, China, Niger, Burkina Faso and Mali. The topic list can be modified to reflect the content of the corpus.

Finally, it selects articles mentioning a chosen topic and displays the 20 most frequent words in their texts. Common French and English words are excluded to make the results easier to interpret.

This section provides an initial overview of the topics and vocabulary represented in the corpus. It is a descriptive step before more detailed text analysis.

## 11. Choices and limitations

- **News website list:** The list of news websites is maintained in `news_domains.txt`. It can be updated as new domains are identified.
- **Request frequency:** The notebook includes a pause between requests to avoid sending too many requests to the same websites.
- **Access restrictions:** Some large news websites refuse automated access, returning errors such as `http_403` or `http_402`. The notebook does not attempt to get around these protections. Instead, it checks the Internet Archive for an available copy.
- **Archived articles:** Recent articles may not have been archived yet. The Internet Archive is therefore more useful for older articles, although availability depends on whether a copy was saved.
- **Paywalls:** Articles behind a paywall may return only the freely available part of the text. These articles are flagged in the `paywall` column.
- **Non-article pages:** Some links may lead to videos, homepages, error pages or pages listing several articles rather than to one specific article.
- **Unavailable links:** A link may no longer work even if it was available when the tweet was published.
- **Shortened links:** Some shorteners may not expose the final article address through the redirects or page information checked by the notebook.
- **Research databases:** For a research corpus, access to a licensed press database such as Europresse or Factiva would be a useful complement, particularly for articles that cannot be retrieved from the original websites or the Internet Archive.
- **Twitter/X API:** If the tweets come from the Twitter/X API, the `expanded_url` field may already contain the original address behind a `t.co` link and can replace part of the URL resolution process.


## 12. Files

| File | Description |
|---|---|
| `tweets_to_news_articles.ipynb` | Jupyter notebook containing the code, explanations and results |
| `news_domains.txt` | List of news websites, one domain per line |
| `data/sample_tweets.csv` | Sample input file |
| `requirements.txt` | Python libraries required to run the notebook |

The notebook creates `output/links.csv` and `output/articles.csv` when it is run. They are not included in the repository because `articles.csv` contains the full text of press articles.

