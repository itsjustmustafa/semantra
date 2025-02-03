# Implementing CLI Functionality for Semantra

## Python Setup Adjustments

To get the CLI functionality working properly, I had to tweak some Python setup configurations. First, `pyproject.toml` needed to run in editable mode, which required upgrading `setuptools` to version 64 or higher.

Another issue I encountered was the heavy use of relative imports, like:

```python
from .pdf import get_pdf_content
```

I changed these to absolute imports:

```python
from pdf import get_pdf_content
```

Not entirely sure why relative imports were used, but everything seems to work fine after switching.

## Fixing Frontend File Retrieval Issue

Semantra serves frontend files via symlink, using:

```python
pkg_resources.resource_filename("semantra.semantra", "client_public")
```

Python wasn't able to locate the `semantra` module regardless of where I ran the script. Changing it to:

```python
pkg_resources.resource_filename("semantra", "client_public")
```

fixed the issue.

## Refactoring the Backend for CLI Support

To run the program without starting the server, I had to separate the Flask endpoints from their core logic. Previously, `query()` handled both receiving the POST request and executing the model search. Now, I've extracted that logic into a function called `query_by_queries_and_preferences()`, not the nicest name, but for quick work I prefer to be as explicit as possible to avoid confusion. The endpoint now simply acts as a wrapper:

```python
    @app.route("/api/query", methods=["POST"])
    def query():
        queries = request.json["queries"]
        preferences = request.json["preferences"]
        return jsonify(query_by_queries_and_preferences(queries, preferences))
```

and `query_by_queries_and_preferences()` as follows:

```python
def query_by_queries_and_preferences(queries, preferences):
    #...<some stuff with queries>...
    #...<some stuff with preferences>...
    #...<a lot more stuff>...
    results = #...<put it all together>...
    return results
```

With this, `query_by_queries_and_preferences()` acts as the main processor of the semantic search, while `query()` is the wrapper that just injects the client data.

The `query()` method also called two other models, `annoy` via `queryann`, and `svm` via `querysvm`, I made similar adjustments to both these too, however it needs a lot of refactoring in my opinion.

Next, the interface for the user.

### Handling Query Data

The two relevant pieces of client data are:

-   **`queries`**: A list of objects containing a `search term` and `weight`.
-   **`preferences`**: I think these are prior search results for relevance tuning (using Semantra, these are left empty for standalone searches when the user initially searches something).

For the CLI, constructing these is simple:

```python
    def query_by_search_term(search_term: str):
        queries = [{
            'query': search_term,
            'weight': 1,
        }]
        preferences = [] # Since this is a fresh search
        return query_by_queries_and_preferences(queries, preferences)
```

## Adding CLI Arguments

To make the tool work from the command line, I introduced two new arguments:

-   `--search <SEARCH-TERM>`
-   `--save-search-to <DEST-FILE>`

Now, running the following command saves results to `christmas.json`:

```sh
python3 semantra/src/semantra/semantra.py semantra/The\ Rust\ Programming\ Language\ Chapter\ 3.pdf --no-server --search christmas --save-search-to christmas.json
```

## Final Output

Here's an excerpt from the generated `christmas.json` file:

```json
{
    "results": [
        [
            "semantra/The Rust Programming Language Chapter 3.pdf",
            [
                {
                    "text": "...Print the lyrics to the Christmas carol ‘The Twelve Days of Christmas’...",
                    "distance": 0.2202,
                    "filename": "semantra/The Rust Programming Language Chapter 3.pdf",
                    "queries": [{ "query": "christmas", "weight": 1 }],
                    "preferences": []
                }
            ]
        ]
    ],
    "sort": "desc"
}
```

Everything is working now, but if I had more time, I'd refactor my changes to be more scalable for future backend models. Not too happy with how I implemented the ann and svm parts of this, but it works well enough.
