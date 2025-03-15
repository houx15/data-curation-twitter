# data-curation-twitter

- [data-curation-twitter](#data-curation-twitter)
  - [Logs](#logs)
    - [Mar.15, 2025](#mar15-2025)
  - [Solution](#solution)
    - [File Format](#file-format)
    - [Parquet Engine](#parquet-engine)
    - [Parquet Compression Schema](#parquet-compression-schema)
    - [Indexing System](#indexing-system)
      - [Tweet Text: Year + User ID Range](#tweet-text-year--user-id-range)
      - [User Profile (Embedded) (TODO)](#user-profile-embedded-todo)
      - [User Profile (TODO)](#user-profile-todo)
      - [User Network (TODO)](#user-network-todo)
  - [Data Format (Curated Data)](#data-format-curated-data)
  - [Data Format (Original Data)](#data-format-original-data)
    - [Twitter Text](#twitter-text)
      - [**Pickle file**: pandas DataFrame](#pickle-file-pandas-dataframe)
      - [**iPickle file**: dictionary](#ipickle-file-dictionary)
    - [User Profile (TODO)](#user-profile-todo-1)
    - [User Network (TODO)](#user-network-todo-1)

## Logs
### Mar.15, 2025
initialize

## Solution
### File Format
|Plan|Strength|Weakness|Selected|
|--|--|--|--|
|Database (MongoDB+Neo4J)|1.flexible schema; 2.high query efficiency; 3. scalable and distributed;|need to set up a running service (not suitable on public servers)|-|
|Embedded DB (SQLite/RocksDB)|1. lightweight and fast; 2. relatively high query efficiency | limited scalability | -|
|Flat Files (JSON/CSV)|easy to use and human readable;|poor performance for large-scale data|-|
|Columnar Storage|1.efficient for analysis; 2.compression; 3.optimized for column-based queries; 4. scalable and distributed;|1. poor random access performance; 2. can not make incremental writes;|✅|
|Hierarchical Storage (HDF5)|1. efficient for large-scale scientific data; 2. better random access performance and incremental writes;|1. poor distributed system support; 2. not ideal for tabular data| -|

### Parquet Engine
|engine|read|write|compression|partioning|
|--|--|--|--|--|
|Fastparquet|slower for large datasets|faster|none by default|basic|
|✅Pyarrow|faster for large datasets|slower|snappy by default|advanced|


### Parquet Compression Schema
|Schema|compression ratio|compression speed|decompression speed|Selected|
|--|--|--|--|--|
|None|None|Fastest|Fastest|-|
|Snappy|Medium|Fast|Fast|✅|
|Gzip|High|Slow|Slow|-|
|Brotli|Very high| Slowest|Slowest|-|
|zstd|High|Fast|Medium|-|
|lz4|Low|Fast|Fast|-|

### Indexing System
The indexing system is designed based on the following typical use cases:
1. Analyzing all tweets within a specific time range for a given topic.
2. Retrieving a user's information based on their user ID.
3. Fetching all tweets posted by a specific user through one's user ID


#### Tweet Text: Year + User ID Range
**Directory Structure**
- Tweets are partitioned by year.
- Within each year, tweets are further divided into sorted blocks based on `user_id`.
- Each file name explicitly records the starting and ending `user_id` for that file.

Example Directory Structure:
```
/data-tweet-curated/
  ├── 2006/
  │     ├── bucket_0_1000.parquet       # user_id range: 0 to 1000
  │     ├── bucket_1001_2000.parquet    # user_id range: 1001 to 2000
  │     └── ...
  ├── 2007/
  │     ├── bucket_0_1500.parquet       # user_id range: 0 to 1500
  │     ├── bucket_1501_3000.parquet    # user_id range: 1501 to 3000
  │     └── ...
  └── ...
```

**Partition Strategy**
- Sort by `user_id`: First, sort all tweets by `user_id` within each year.
- Divide into ranges: Divide the sorted data into ranges of `user_id`. Each range corresponds to one file, and the file name explicitly records the range (e.g., bucket_0_1000.parquet for `user_id` range 0–1000).
- Ensure user uniqueness: A single `user_id` should only appear in one file. All tweets from the same user will be stored in the same file.
- File size control: one parquet file contain no more than 100,000 lines.


**Advantages**
- Query Efficiency: For a specific user in a given year, only one bucket file needs to be loaded.
- Storage Optimization: Using Parquet with compression reduces storage size while enabling fast queries.

**Future Enhancements**
- Metadata Index: A global metadata file (e.g., JSON or SQLite) to map user_id to the specific year and bucket. This would allow for faster lookups across multiple years.

#### User Profile (Embedded) (TODO)



#### User Profile (TODO)



#### User Network (TODO)

## Data Format (Curated Data)

## Data Format (Original Data)
### Twitter Text

Each day (`yyyy-mm-dd`) has several integer indices (0-99). Each index has a pickle file and an ipickle file.

**File name format:**
1. `yyyy-mm-dd_yyyy-mm-dd_{topic}_keywords_alpha_beta_{index}.pickle4`
2. `yyyy-mm-dd_yyyy-mm-dd_{topic}_keywords_alpha_beta_{index}.ipickle4`

---

#### **Pickle file**: pandas DataFrame

| Variable                        | Format        | Content                                                                                                                                                                                                                  |
|---------------------------------|---------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `conversation_id`               | str           | The conversation this tweet belongs to. A tweet might be part of a conversation (e.g., replies or quoted tweets). All related tweets share the same `conversation_id`.                                                  |
| `lang`                          | str           | Language.                                                                                                                                                                                                               |
| `created_at`                    | str           | Creation time of the tweet in the format `YYYY-MM-DDTHH:mm:ss.sssZ`.                                                                                                                                                    |
| `id`                            | str           | Tweet ID.                                                                                                                                                                                                               |
| `edit_history_tweets`           | list of str   | A list used to track all edit histories of this tweet.                                                                                                                                                                 |
| `referenced_tweets`             | list of object| An array indicating the IDs and types of tweets referenced by this tweet. Types include: 1. `replied_to`: the tweet is a reply; 2. `quoted`: the tweet quotes another; 3. `retweeted`: the tweet is a retweet.           |
| `text`                          | str           | The text content of the tweet.                                                                                                                                                                                          |
| `author_id`                     | str           | The ID of the author of the tweet.                                                                                                                                                                                      |
| `public_metrics.retweet_count`  | int           | Number of retweets.                                                                                                                                                                                                    |
| `public_metrics.reply_count`    | int           | Number of replies.                                                                                                                                                                                                     |
| `public_metrics.like_count`     | int           | Number of likes.                                                                                                                                                                                                       |
| `public_metrics.quote_count`    | int           | Number of quotes.                                                                                                                                                                                                      |
| `public_metrics.impression_count` | int         | Number of impressions.                                                                                                                                                                                                 |
| `in_reply_to_user_id`           | str           | The ID of the user being replied to.                                                                                                                                                                                   |

---

#### **iPickle file**: dictionary

```python
{
    "users": pandas DataFrame,
    "tweets": pandas DataFrame
}
```


**`users` pandas DataFrame**: Embedded information of a user. The information captures the user's status at the time of crawling.

| Variable                       | Format    | Content                                                                                 |
|--------------------------------|-----------|-----------------------------------------------------------------------------------------|
| `name`                         | str       | User's nickname.                                                                        |
| `username`                     | str       | Twitter handle/screen name (unchangeable).                                              |
| `verified`                     | bool      | Whether the user is verified or not.                                                    |
| `id`                           | str       | Unique user ID.                                                                         |
| `created_at`                   | str       | Account creation time in the format `YYYY-MM-DDTHH:mm:ss.sssZ`.                         |
| `public_metrics.followers_count` | int     | Number of followers.                                                                    |
| `public_metrics.following_count` | int     | Number of accounts the user is following.                                               |
| `public_metrics.tweet_count`   | int       | Total number of tweets by the user.                                                     |
| `public_metrics.listed_count`  | int       | Number of times the user is listed in public lists.                                      |
| `location`                     | str       | User's geo location (if provided).                                                      |

---

**`tweets` pandas DataFrame**: Same format as the pickle file.


### User Profile (TODO)

### User Network (TODO)