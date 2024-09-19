# Movie Review and Related Movies Data Preparation

## Background

This project involves preparing data for a recommendation system to help people find movie reviews and related movies. Data is extracted from two sources: The New York Times API and The Movie Database (TMDb) API. The extracted text can later be utilized for natural language processing tasks.

## Table of Contents
1. [Requirements](#requirements)
2. [Setup](#setup)
3. [Instructions](#instructions)
   - [Part 1: Access the New York Times API](#part-1-access-the-new-york-times-api)
   - [Part 2: Access The Movie Database API](#part-2-access-the-movie-database-api)
   - [Part 3: Merge and Clean the Data for Export](#part-3-merge-and-clean-the-data-for-export)
4. [References](#references)

## Requirements

- Python 3.x
- Requests library
- Pandas library
- Dotenv library
- JSON library

## Setup

1. Clone the repository:
    ```bash
    git clone https://github.com/your_username/movie-review-recommendation.git
    cd movie-review-recommendation
    ```

2. Install the required packages:
    ```bash
    pip install requests pandas python-dotenv
    ```

3. Create a `.env` file in the root directory of the project and add your API keys:
    ```plaintext
    NYT_API_KEY=your_nyt_api_key
    TMDB_API_KEY=your_tmdb_api_key
    ```

## Instructions

### Part 1: Access the New York Times API

1. Set up the base URL and search string variables:
   ```python
   import requests
   import time
   from dotenv import load_dotenv
   import os
   import pandas as pd
   import json

   load_dotenv()
   NYT_API_KEY = os.getenv('NYT_API_KEY')

   # Set the base URL
   url = "https://api.nytimes.com/svc/search/v2/articlesearch.json?"
   filter_query = 'section_name:"Movies" AND type_of_material:"Review" AND headline:"love"'
   sort = "newest"
   field_list = "headline,web_url,snippet,source,keywords,pub_date,byline,word_count"
   begin_date = "20130101"
   end_date = "20230531"
   ```

2. Initialize an empty list to store the reviews:
   ```python
   reviews_list = []
   ```

3. Loop through 20 pages to retrieve reviews:
   ```python
   for page in range(20):
       query_url = f"{url}q={filter_query}&sort={sort}&api-key={NYT_API_KEY}&page={page}&begin_date={begin_date}&end_date={end_date}&fl={field_list}"
       
       try:
           response = requests.get(query_url)
           reviews = response.json()
           for review in reviews["response"]["docs"]:
               reviews_list.append(review)
           print(f"Page {page} retrieved")
           time.sleep(12)  # Respect rate limits
       except Exception as e:
           print(f"Error on page {page}: {e}")
           break
   ```

4. Preview the first five results in JSON format:
   ```python
   print(json.dumps(reviews_list[:5], indent=4))
   ```

5. Convert `reviews_list` to a Pandas DataFrame:
   ```python
   reviews_df = pd.json_normalize(reviews_list)
   ```

6. Extract the movie title:
   ```python
   reviews_df['title'] = reviews_df['headline.main'].apply(lambda st: st[st.find("\u2018")+1:st.find("\u2019 Review")])
   ```

7. Extract keywords using the supplied `extract_keywords` function:
   ```python
   reviews_df['keywords'] = reviews_df['keywords'].apply(extract_keywords)
   ```

8. Create a list of titles:
   ```python
   titles = reviews_df['title'].to_list()
   ```

### Part 2: Access The Movie Database API

1. Setup URL and API Key:
   ```python
   TMDB_API_KEY = os.getenv('TMDB_API_KEY')
   tmdb_url = "https://api.themoviedb.org/3/search/movie?query="
   ```

2. Initialize an empty list to store TMDB results:
   ```python
   tmdb_movies_list = []
   request_counter = 1
   ```

3. Loop through titles to retrieve movie details:
   ```python
   for title in titles:
       if request_counter % 50 == 0:
           print("Sleeping for 1 second to respect API limit.")
           time.sleep(1)

       request_counter += 1
       response = requests.get(f"{tmdb_url}{title}&api_key={TMDB_API_KEY}")
       
       try:
           movie_data = response.json()
           # Extract movie ID and details
           movie_id = movie_data['results'][0]['id']
           details_response = requests.get(f"https://api.themoviedb.org/3/movie/{movie_id}?api_key={TMDB_API_KEY}")
           details = details_response.json()
           
           # Store the relevant details
           genres = [genre['name'] for genre in details['genres']]
           tmdb_movies_list.append({
               'title': details['title'],
               'original_title': details['original_title'],
               'budget': details['budget'],
               'original_language': details['original_language'],
               'homepage': details['homepage'],
               'overview': details['overview'],
               'popularity': details['popularity'],
               'runtime': details['runtime'],
               'revenue': details['revenue'],
               'release_date': details['release_date'],
               'vote_average': details['vote_average'],
               'vote_count': details['vote_count'],
               'genres': genres
           })
           print(f"Found movie: {details['title']}")
       except Exception as e:
           print(f"Movie not found for title: {title}. Error: {e}")
   ```

4. Preview the results:
   ```python
   print(json.dumps(tmdb_movies_list[:5], indent=4))
   ```

5. Convert the results to a DataFrame:
   ```python
   tmdb_df = pd.DataFrame(tmdb_movies_list)
   ```

### Part 3: Merge and Clean the Data for Export

1. Merge the two DataFrames:
   ```python
   merged_df = pd.merge(reviews_df, tmdb_df, on='title', how='inner')
   ```

2. Clean string columns:
   ```python
   columns_to_fix = ['genres']
   characters_to_remove = ["[", "]", "'"]
   for column in columns_to_fix:
       merged_df[column] = merged_df[column].astype(str)
       for char in characters_to_remove:
           merged_df[column] = merged_df[column].str.replace(char, "", regex=True)
   print(merged_df.head())
   ```

3. Remove duplicates and reset index:
   ```python
   merged_df.drop_duplicates(inplace=True)
   merged_df.reset_index(drop=True, inplace=True)
   ```

4. Export to CSV:
   ```python
   merged_df.to_csv('movie_reviews.csv', index=False)
   ```

## References

- [New York Times Article Search API](https://developer.nytimes.com/docs/articlesearch-product/1/overview)
- [The Movie Database API Documentation](https://developer.themoviedb.org/docs)
- [Pandas Documentation](https://pandas.pydata.org/pandas-docs/stable/)
- [Datetime Documentation](https://docs.python.org/3/library/datetime.html)
- [JSON Documentation](https://docs.python.org/3/library/json.html)

