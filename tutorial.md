# Implement Semantic Search using Datasette
### Spinning up 'vibe-search' on a SQLite Database

[Datasette](https://datasette.io/) is an open-source tool for exploring and publishing data. It allows you to create web interfaces for exploring databases with minimal effort. Developed by Simon Willison, a co-creator of the Django web framework, Datasette is similarly designed to be simple, lightweight, and easy to use. One of the things it greatly simplifies is the ability to quickly spin up "semantic search" against a SQLite database. 

Semantic search is a method for returning highly relevant results based on *meaning* rather than just keyword filtering. This allows users to find documents that may relate to their search term, even when the matching keywords are not present in the search query. This is ideal for those instances where we are trying to search in a general direction or where we don't yet know the right keywords to use.

I discussed this with a lawyer friend of mine and he immediately understood it as "vibe search." Fittingly, this is the same term that Willison used in his [excellent post](https://simonwillison.net/2023/Oct/23/embeddings/) that describes working with semantic search and embeddings in more detail. 

I'm going to walk through how I used Datasette to quickly create and deploy a functional semantic search engine on a database of 5,000 recipes. With this functionality, we will be able to search our database for "a small levantine appetizer" and receive highly useful results. That is an odd way to look up a recipe but semantic search "just gets" what we mean.

The key point here is that you do not *need* a vector database or lots of tooling to quickly spin up powerful semantic search on a decent-sized data set.

*Note: this tutorial is largely modeled after Willison's posts on his blog, starting with [this post](https://simonwillison.net/2023/Jan/13/semantic-search-answers/). I simplified things a bit and focused more narrowly on implementing semantic search and working with Datasette for the first time.* 

## Tutorial

### Download dataset into project directory
We're going to work with a simple dataset that includes the title, ingredients, and instructions for 5,000 recipes. Grab the 5k-recipes.db file from this [recipe dataset repository](https://github.com/josephrmartinez/recipe-dataset).

Make a new project directory folder and move the 5k-recipes.db database file inside of this directory.


### Create and activate virtual environment
Navigate to your project directory and create and activate a virtual environment running python 3.11. Datasette requires Python 3.7 or higher.

```
python3.11 -m venv venv
source venv/bin/activate  # On Windows, use `venv\Scripts\activate`
```
### Install datasette
It's generally a good practice to install Python packages within the virtual environment to keep dependencies isolated. This ensures that your project's dependencies won't interfere with other projects or the global Python environment. Use pip to install Datasette in your current virtual environment:

```
pip install datasette
```

Or follow [these instructions](https://docs.datasette.io/en/stable/installation.html) to install Datasette globally on your machine.

### Run Datasette
To launch the Datasette server with the '5k-recipes' database, ensure you are in the project directory with the terminal open and your virtual environment activated. Run the following command:

```
datasette 5k-recipes.db
```

Open a browser and navigate to [http://127.0.0.1:8001](http://127.0.0.1:8001)

You should now be able to easily explore and apply SQL queries to the recipes table within the 5k-recipes database. 

### Enable full text search
Before we spin up semantic search, we are going to enable full text search on our recipes table. Eventually, we will be able to run comparisons between the two search methods and evaluate response time as well as query flexibility.

[sqlite-utils](https://datasette.io/tools/sqlite-utils), also by Simon Willison, is a CLI tool and Python library for manipulating SQLite databases. We're going to use sqlite-utils to [enable full-text search](https://sqlite-utils.datasette.io/en/stable/cli.html#configuring-full-text-search) across the Title, Ingredients, and Instructions columns in our recipes table:

```
pip install sqlite-utils
sqlite-utils enable-fts 5k-recipes.db recipes Title Ingredients Instructions
```

Now navigate back to our recipes table running on Datasette in the browser: [http://localhost:8001/5k-recipes/recipes](http://localhost:8001/5k-recipes/recipes)

Datasette automatically recognizes that full text search has been enabled on the table and provides a search box UI in the web interface. 

Let's run a search for "comfort food."

We get two results because the phrase "comfort food" literally shows up in the instructions for two recipes. But surely there are more than just two recipes that would qualify as "comfort food" in this collection of 5,000 recipes...

There are! And to find those, we are going to start by calculating embeddings for all of the recipes in our database.

### Calculate embeddings
An embedding serves as a condensed representation of text, capturing its nuanced meaning through a list of floating-point numbers. The OpenAI embedding model allows us to transform a given text into a 1,536-dimensional vector that encapsulates the intricate semantic features of the text. 

We will use the [openai-to-sqlite](https://datasette.io/tools/openai-to-sqlite) Datasette tool to efficiently calculate and store an embedding on each recipe in our SQLite database. 

**Note:** this operation requires an OpenAI API key. Running this operation will take about two minutes and cost 1,909,410 tokens. That amounts to about $0.19 based on the current pricing for the ada v2 embedding model. Since I have already calculated these embeddings, you can skip this step and download the full-text search enabled database that also includes the embeddings table here: [https://www.kaggle.com/datasets/josephmdev/recipes/](https://www.kaggle.com/datasets/josephmdev/recipes/)

If you choose to download this resource, be sure to delete the old 5k-recipes.db file and replace it with the new resource. 

Enter the commands below if you would like to install the openai-to-sqlite tool and calculate the embeddings yourself. Provide your API key in the token parameter. 

```
pip install openai-to-sqlite
openai-to-sqlite embeddings 5k-recipes.db \
--sql 'select id, Title, Ingredients, Instructions from recipes' \
--table recipe_embeddings \
--token=sk-...
```

This concatenates together the Title, Ingredients, and Instructions columns from the recipes table, runs them through the OpenAI embeddings API and stores the results in a new table called recipe_embeddings with the following schema:

```
CREATE TABLE [recipe_embeddings] (
   [id] INTEGER PRIMARY KEY,
   [embedding] BLOB
)
```


### Create metadata file to enable advanced functionality

When we enabled full text search on the recipes table, Datasette automatically provided a search box for us to run queries against the table. In order to run semantic search queries in the Datasette web UI using the embeddings we just created, we are going to create a [metadata](https://docs.datasette.io/en/stable/metadata.html) file and develop a ["canned query"](https://docs.datasette.io/en/stable/sql_queries.html#canned-queries) that will enable similar functionality for the user to conduct semantic search.

Create a metadata.json file in your project folder with the following content:

```
{
    "title": "Recipe Search",
    "description": "Full text and semantic search on 5,000 recipes.",
    "databases": {
        "5k-recipes": {
            "source": "Recipe Dataset from Epicurious",
            "source_url": "https://github.com/josephrmartinez/recipe-dataset",
            "license": "CC BY-SA 3.0",
            "license_url": "https://creativecommons.org/licenses/by-sa/3.0/",
            "queries": {
            }
        }
    },
    "plugins": {
    }
}
```
As we develop our custom query, we will populate the 'queries' and 'plugins' sections within the metadata file.

Close the terminal running your Datasette server, then restart it with the following command to apply the metadata and relaunch the server:

```
datasette 5k-recipes.db --metadata metadata.json
```

Navigate back to the browser and notice that the database name, data source attribution, and other metadata is now displayed our Datasette web display.

Now we are going to build out the custom SQL query that will let us perform semantic search. We will drop this query into our metadata file so that it is easy for users to invoke this from the Datasette web interface.

To perform the semantic search operation, our query needs access to two helper functions: 
First, we need to calculate an embedding on the user's query.
Then, we need to perform a vector similarity calculation between the embedding of the user's query and all of the embeddings in our recipe_embeddings table.

This is one of the most exciting parts of working with Datasette. We can create or install plugins that allow us to utilize powerful Python libraries in custom SQL queries that go far beyond what many people think of as possible with SQL. 


### Function to calculate embedding on user's query

The [datasette-openai](https://datasette.io/plugins/datasette-openai) plugin allows us to use SQL functions within Datasette for calling the OpenAI APIs. Let's install this plugin so that we can use its openai_embeddings function to calculate an embedding on our user query:

```
datasette install datasette-openai
```

Having this plugin installed will allow us to use the openai_embeddings function in custom SQL queries:

openai_embedding(text, api_key)

This calls the OpenAI embedding endpoint and returns a binary object representing the floating point embedding for the provided text, in this case the user's query.

### Function to perform a vector similarity calculation

The function above gives us an embedding for the user's query. We can perform a cosine similarity calculation between this query embedding and all of the values in our recipe_embeddings table to find the "approximate nearest neighbors" -- the documents with the most similar meaning. 

This is really exciting! Embeddings allow us to perform calculations on *meaning.* I'll never get over this.

A "brute-force" cosine similarity calculation in python written by Willison looks like this:

```
def cosine_similarity(a, b):
    dot_product = sum(x * y for x, y in zip(a, b))
    magnitude_a = sum(x * x for x in a) ** 0.5
    magnitude_b = sum(x * x for x in b) ** 0.5
    return dot_product / (magnitude_a * magnitude_b)
```

This works, but it can be slow on larger datasets. To more efficiently perform this similarity search calculation, we are going to use the [Faiss](https://github.com/facebookresearch/faiss) library developed by Facebook AI Research. 

The [datasette-faiss](https://datasette.io/plugins/datasette-faiss) plugin enables us to use the Faiss library within Datasette. This plugin creates in-memory Faiss indexes for specified tables on startup, using an IndexFlatL2 Faiss index type. The tables to be indexed must have id and embedding columns. Our recipe_embeddings table is already formatted properly, so we are ready to install the plugin:

```
datasette install datasette-faiss
```

In order for the datasette-faiss plugin to function properly, you must specify which tables should have indexes created for them by updating the "plugins" section of our metadata.json file:

```
{
    "plugins": {
        "datasette-faiss": {
            "tables": [
                ["5k-recipes", "recipe_embeddings"]
            ]
        }
    }
}
```

We can now use the following function in our custom query:

```
faiss_search_with_scores(database, table, embedding, k)
```

This function returns a JSON array of the k nearest neighbors and similarity score of each to the embedding found in the specified database and table.


### Develop custom SQL query

Now that we have installed the appropriate plugins and identified our two helper functions, we are ready to develop our cutom SQL query to perform semantic search. Let's start here:

```
select
  value
from
  json_each(
    faiss_search_with_scores(
      '5k-recipes',
      'recipe_embeddings',
      (
        select
          openai_embedding(:query, :openai_api_key)
      ),
      10
    )
  )
```

This innermost subquery calculates the OpenAI embedding for the user's query (:query) using the API key specified (:openai_api_key). Both of these values are provided by the user through the Datasette web UI. The result is a binary object representing the floating-point embedding for the provided text.

This result is then used as the embedding parameter in the faiss_search_with_scores function. This function performs a semantic search using Faiss on the specified database ('5k-recipes') and table ('recipe_embeddings'). It returns a JSON array of the 10 nearest neighbors and their similarity scores to the provided embedding.

The outermost part of the query uses json_each to extract values from the JSON array returned by the faiss_search_with_scores function. This allows the query to retrieve the actual values (recipe ids in this case) associated with the nearest neighbors.

Now we are going to add "where length(coalesce(:query, '')) > 0" to prevent the query from running if the user hasn't added anything into the search box yet. If you do not add this, you will get this error message "user-defined function raised exception." 

Finally, since we are going to put this query into our metadata.json file, the whole query should just be formatted as one long string.

Update your metadata.json file to the following:

```
{
    "title": "Recipe Search",
    "description": "Full text and semantic search on 5,000 recipes.",
    "databases": {
      "5k-recipes": {
        "source": "Recipe Dataset from Epicurious",
        "source_url": "https://github.com/josephrmartinez/recipe-dataset",
        "license": "CC BY-SA 3.0",
        "license_url": "https://creativecommons.org/licenses/by-sa/3.0/",
        "queries": {
            "semantic-search": {
              "hide_sql": false,
              "sql": "select value from json_each(faiss_search_with_scores('5k-recipes', 'recipe_embeddings', (select openai_embedding(:query, :openai_api_key)), 10)) where length(coalesce(:query, '')) > 0"
            }
          }
        }
    },
    "plugins": {
        "datasette-faiss": {
            "tables": [
                ["5k-recipes", "recipe_embeddings"]
            ]
        }
    }
  }
```

Close the terminal running your Datasette server, then restart it with the following command to apply the updated metadata and relaunch the server:

```
datasette 5k-recipes.db --metadata metadata.json
```

...

Head over to [http://localhost:8001/5k-recipes](http://localhost:8001/5k-recipes) and notice how we now have "semantic search" listed under "Queries" on the index page. Click on this link. 

Let's try "comfort food" again as the query. You're going to need to enter your API key in the openai_api_key field in order for this to work. It is going to cost a fraction of a fraction of a penny to run this operation.

And look! Something happened. We get what looks like a table of IDs and then some floating point numbers. Those must be the "similarity scores" calculated by the Faiss function.

Let's develop our custom SQL function a little bit more to perform a JOIN with our original recipes table:

```
SELECT
  ROUND(CAST(json_extract(value, '$[1]') AS REAL), 3) AS score,
  r.Title,
  r.Ingredients,
  r.Instructions
FROM
  json_each(
    faiss_search_with_scores(
      '5k-recipes',
      'recipe_embeddings',
      (
        SELECT
          openai_embedding(:query, :openai_api_key)
      ),
      10
    )
  ) AS json_data
  JOIN recipes AS r ON r.id = CAST(json_extract(json_data.value, '$[0]') AS INTEGER)
WHERE
  length(coalesce(:query, '')) > 0;
```

This should give us a nicely formatted table with the rounded similarity scores displayed next to the recipe titles, ingredients, and instructions.

Since this function is going to live in our metadata.json file, it will need to be formatted as a long string:

```
"sql": "SELECT ROUND(CAST(json_extract(value, '$[1]') AS REAL), 3) AS score, r.Title, r.Ingredients, r.Instructions FROM json_each(faiss_search_with_scores('5k-recipes', 'recipe_embeddings', (SELECT openai_embedding(:query, :openai_api_key)), 10)) AS json_data JOIN recipes AS r ON r.id = CAST(json_extract(json_data.value, '$[0]') AS INTEGER) WHERE length(coalesce(:query, ''))>0;"
```

Make the update to the custom query and save your metadata file. For reference, your metadata.json file should now look like the demo [metadata.json](metadata.json) file in this repo. Close the terminal running your Datasette server, then restart it with the following command to apply the updated metadata and relaunch the server:

```
datasette 5k-recipes.db --metadata metadata.json
```

### Run semantic search

Head on over to [http://localhost:8001/5k-recipes/semantic-search](http://localhost:8001/5k-recipes/semantic-search) and try out a search for "comfort food."

And how about that! Feel-Good Chicken Soup, Italian Sundaes with Nutella, Chicken and Squash Cacciatore... I'm already feeling better.

We can see at the bottom of the Datasette page that it took about 260ms to run this query. This includes the time it takes to hit the OpenAI API to create an embedding of the user query! Not bad. Sure, it only took 10ms to run the same full-text search on the recipes table. But that query only returned two results. Semantic search gave us better results in number, significance, and deliciousness.

With this "vibes-based search" I can also grasp for something much more conceptual, such as "a small levantine appetizer." If I try this query using full text search, I get the following response: "0 rows where search matches "a small levantine appetizer" sorted by rowid"

But with my custom query that first creates an embedding on the query phrase and then uses Faiss to find the approximate nearest neighbors, I get Sam's Spring Fattoush Salad, spiced labneh, and eight other appetizer options. The word "levantine" does not show up at all in these recipes. Instead, the embedding gets at the underlying meaning in the word "levantine" and the similarity search algorithm helps us find the most relevant documents. 

## Key takeaways:

Datasette is highly extensible: It allows you to install plugins that enhance its functionality, such as the datasette-openai and datasette-faiss plugins that enable the integration of OpenAI's API and the Faiss library into Datasette's SQL queries.

Python Integration: Datasette allows Python code to be integrated into SQL queries. This is achieved by using special functions that can execute Python code, such as openai_embedding and faiss_search_with_scores. The Python code is executed in the context of the Datasette environment.

Metadata File: The metadata file (metadata.json) is used to configure Datasette's behavior, including defining custom queries, plugins, and other settings. It helps organize and describe the data and functionality provided by Datasette.

In traditional SQL databases, you typically don't have this level of integration with external services or custom Python functions. However, Datasette's approach makes it easy to extend the capabilities of SQLite databases, providing a powerful and flexible platform for exploring and interacting with data.


## Further development

**Deploy your project** by [publishing your Datasette instance](https://docs.datasette.io/en/stable/publish.html#publishing). Note that due to the plugins we installed, you will need to provision additional memory and you will likely exceed any free hosting tiers. 

**Build out a custom display** for your data by configuring a [custom template](https://docs.datasette.io/en/stable/custom_templates.html). You can also take this a step further and develop a completely custom front end that hits your deployed [Datasette page and API endpoints](https://docs.datasette.io/en/stable/pages.html). 

**Use another data set.** Try re-building this project but with a much larger dataset. Do you start to encounter performance issues? Maybe you need to build multiple embedding indexes. Experiment with structured and unstructured data.

**Implement RAG.** This tutorial covered the first parts of what is known as [Retrieval Augmented Generation](https://github.com/openai/openai-cookbook/blob/main/examples/Question_answering_using_embeddings.ipynb). We ended at the stage where the most relevant documents are returned. Now complete the implementation of RAG by enabling full question answering. 

**Improve this tutorial** [Fork the repository](https://github.com/josephrmartinez/recipe-dataset) and improve this tutorial. Or, write your own tutorial to help others get up and running with Datasette!
