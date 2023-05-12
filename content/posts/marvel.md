---
title: "Marvel API"
date: 2023-02-17T10:55:21+01:00
draft: false
---
# [Marvel API Data Collection](https://github.com/pflashgary/marvel)
The task requires the use of the Marvel API to collect data on Marvel characters and the number of comics in which they appear.

## MarvelService library

A Python module that implements a wrapper for the Marvel API, which allows for the retrieval of data on Marvel characters and comics. The module contains a class named "MarvelService" with methods that use the Marvel API to retrieve data in a paginated manner.

- "init": Initializes the MarvelService class and sets the public key, private key, API prefix, and other class variables.

- "_hash": This function generates a unique hash code based on the provided nonce, public key, and private key. The hash code is required for authentication when accessing the Marvel API. To generate the hash, the function combines the nonce, private key, and public key into a single string and passes it to the hashlib.md5 function. The resulting hash is returned as a string.

- "_get": Sends a GET request to the specified endpoint with the provided parameters and returns the response in JSON format. This function includes the required authentication parameters to access the Marvel API.

- "_get_paginated": This function retrieves paginated results from the Marvel API by iterating through the pages until all results are obtained. The function first sends a GET request to the specified endpoint using the "_get" function, with an offset value of 0 to start retrieving results from the first page. The function then reads the "total" value from the response JSON, which indicates the total number of results available for the given endpoint. The function then enters a while loop that runs until the offset value is greater than or equal to the total number of results. In each iteration of the loop, the function sends a new GET request to the same endpoint, with the offset value incremented by the number of results retrieved in the previous request. The new response JSON is then appended to a list of all results obtained so far. Once the loop finishes, the function returns the list of results. This function allows for retrieving large amounts of data from the Marvel API in a paginated manner, which is necessary because the Marvel API limits the number of results that can be retrieved in a single request.

- "get_characters": Retrieves all characters from the Marvel API using the "/v1/public/characters" endpoint and returns the paginated results in JSON format.

- "get_comics": Retrieves all comics from the Marvel API using the "/v1/public/comics" endpoint and returns the paginated results in JSON format.


```python
import hashlib
import time
import requests
import logging
import sys

class MarvelService:
    def __init__(self, public_key, private_key, api_prefix = "http://gateway.marvel.com"):

        self.public_key = public_key
        self.private_key = private_key
        self.api_prefix = api_prefix
        self.logger = logging.getLogger(__name__)
        self.limit = 100 # maximum for the marvel api
        self.session = requests.Session()

    def _hash(self, nonce: str, public_key: str, private_key: str):
        # this will be used as the nonce (each request to the marvel API key needs to have a different apikey)
        #ts = str(time.time_ns())
        hash = hashlib.md5(nonce.encode() + private_key.encode() + public_key.encode())
        return hash.hexdigest()

    def _get(self, endpoint, **kwargs):
        # make a nonce, it has to be different for every request
        ts = str(time.time_ns())
        hash = self._hash(ts, self.public_key, self.private_key)
        url = f"{self.api_prefix}{endpoint}"
        params = {
            "ts": ts,
            "apikey": self.public_key,
            "hash": hash,
            "limit": self.limit,
            **kwargs
        }
        self.logger.debug(f"making a request to {url} with {params}")
        response = self.session.get(url, params=params)
        if response.status_code != requests.codes.ok:
            raise Exception(f"unexpected response from server: {response.status_code}, {response.json()}")
        return response.json()

    def _get_paginated(self, endpoint):
        results = []
        offset = 0
        total = sys.maxsize # fake an initial max size until we get the real one from the first API response
        self.logger.debug(f"paging through {endpoint}")

        # find the .data.total, then loop over each page until we're done
        while offset < total:
            body = self._get(endpoint, offset=offset)
            total = body["data"]["total"]
            offset += body["data"]["count"]
            self.logger.debug(f'got {body["data"]["count"]} from offset {offset}')
            results += body["data"]["results"]
        self.logger.debug(f"finished paging, got {len(results)} results")
        return results

    def get_characters(self):
        endpoint = "/v1/public/characters"
        self.logger.info(f"getting characters")
        return self._get_paginated(endpoint)

    def get_comics(self):
        endpoint = "/v1/public/comics"
        self.logger.info(f"getting comics")
        return self._get_paginated(endpoint)

```

## Main.py
The main function first loads environment variables for the Marvel API public and private keys using the load_dotenv() function from the dotenv library.

Then, it creates an instance of the MarvelService class, passing in the public and private keys as arguments.

The function then calls the get_characters() method of the MarvelService object to retrieve a list of characters from the Marvel API.

Next, it creates a lambda function character_to_comics_count that takes a character as input and returns a dictionary with the character's name and the number of comics they appear in.

The map() function is used to apply the character_to_comics_count function to each character in the characters list, returning a new list of dictionaries with each character's name and the number of comics they appear in.

The sorted() function is used to sort the list of dictionaries in descending order by the comics_count key.

Finally, the tabulate() function from the tabulate library is used to print the sorted list of dictionaries in a table format to the console.

```python3
character_to_comics_count = lambda character: {"name": character["name"], "comics_count": character["comics"]["available"]}
characters_to_comic_count = map(character_to_comics_count, characters)
sorted_characters_to_comic_count = sorted(characters_to_comic_count, key = lambda x: x["comics_count"], reverse = True)
```

The logs look like:

![Logs](/logs.png)

The output looks like:

| Character name            | Number of comics |
|---------------------------|-----------------|
| Spider-Man (Peter Parker) | 4239            |
| X-Men                     | 3763            |
| Iron Man                  | 2664            |
| Wolverine                 | 2595            |
| Captain America           | 2440            |
| Avengers                  | 2133            |
| Thor                      | 1838            |
| Hulk                      | 1740            |
| Fantastic Four            | 1495            |
| Daredevil                 | 1217            |
| Human Torch               | 1176            |
| Cyclops                   | 981             |
| Thing                     | 947             |
| Doctor Strange            | 901             |
| Deadpool                  | 882             |
| Storm                     | 862             |
| Beast                     | 821             |
| Punisher                  | 742             |
| Invisible Woman           | 732             |
| Black Panther             | 717             |
| ...                       | ...             |
