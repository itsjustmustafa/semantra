# Semantra Tool Coding Challenge - Codebase Exploration Findings

The goal is to modify the Semantra so that instead of highlighting relevant passages in the UI, it extracts all relevant passages to a file, along with metadata about the document they came from.

There are two ways to go about this:

1. Modify the CLI to handle this extraction - adding in extra command line arguments as the new user interface.
2. Add functionality to the frontend that saves a JSON of the search results - maybe a button on the page.

The first approach will require backend modifications that the second one also depends on, so it might be a good idea to start there. The frontend addition is relatively minor in comparison — possibly just a button and file-saving logic.

## Codebase

Semantra has a Python backend with a Svelte frontend written in TypeScript. The backend runs on Flask, and interestingly, it also starts the frontend server. One key endpoint stands out, I've pseudo-coded it up but the ideas are the same:

```python
@app.route("/api/query", methods=["POST"])
def query():
    # This is the data the client posted to the server (hence a POST request)
    # It will include data like what term are they searching up in Semantra
    query_data, other_client_data = client.get_data()

    #...<some stuff with query_data>...
    #...<some stuff with other_client_data>...
    #...<a lot more stuff>...
    results = #...<put it all together>...
    # Send as a network response object back to the client (jsonify converts it to such an object)
    return jsonify(results)
```

This is the REST API's main search endpoint. The plan is to refactor it so it can run independently as a function, rather than requiring an HTTP request. The idea is if it can be run as a non-endpoint function, the flask server does not need to start. There is a command line argument `--no-server` to run Semantra without starting a server.

### Refactor

The idea is as follows

```python
def query_logic(query_data, other_client_data):
    #...<some stuff with query_data>...
    #...<some stuff with other_client_data>...
    #...<a lot more stuff>...
    results = #...<put it all together>...
    return results

@app.route("/api/query", methods=["POST"])
def query():
    query_data, other_client_data = client.get_data()
    return jsonify(query_logic(query_data, other_client_data))
```

With this change, `query_logic()` can be called from both the Flask route and the CLI without needing a running server. All that needs to be done is specify `query_data` and `other_client_data`, or its equivalent in the codebase.

## Recommended Changes

To accomplish the extraction functionality, my reccomended plan is:

1. **Refactor `query()`**: Separate out the core logic so it can be called independently, outside the Flask route.
2. **Use Click for CLI arguments**: Allow specifying a search term and an output file destination.
3. **Frontend Integration**: Really just make the same network request as you would for a search term, and save the network response as json. Could even prompt the user where to save it.

For the frontend addition, I’m not super familiar with Svelte, but from my React experience, it doesn’t seem too difficult. All that’s needed is a button that triggers a function to save the search results to a file. Svelte might handle reactivity a bit differently than React, but the core idea remains the same - Add a button (somewhere that looks nice), upon clicking it, get the json object of the results that the client just got after they searched stuff up, prompt the user to for a save-file, and save the json to that file.

## Next Steps

-   Refactor `query()` to decouple it from Flask.
-   Implement and test the CLI modifications.
-   Add frontend functionality to allow users to save results with a click - This one I'll need to research a bit about Svelte.
