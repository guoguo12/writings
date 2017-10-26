---
layout: post
title: A Concise URL Shortener in Python
---

*How concise?* 14 lines.

*How?* The main ideas:
* Use Google Sheets as a database.
* Use Flask for serving and routing.

## Database setup

Create a new spreadsheet on [Google Sheets](http://sheets.google.com).
Add two columns, mapping short link names to URLs:

![]({{ site.baseurl }}/images/url-shortener-db.png)

Next, turn on link sharing. (Click the blue "Share" button and then "Get shareable link".)

Make note of the spreadsheet's unique ID, which you'll find in the URL.
Mine looks like this:

```
1vr6C7CD_RMaNXmxFybG5H6GibrT1Vsx6Jm3sRQnQ2m2
```

Now here's a neat trick: you can download your spreadsheet as a CSV file by going to

```
https://docs.google.com/spreadsheets/d/KEY/export?format=csv&id=KEY&gid=0
```

(Don't forget to replace `KEY` with your spreadsheet's ID.)

## Backend setup

The code speaks for itself:

```python
from flask import Flask, redirect
import requests

SHEET_ID = '1vr6C7CD_RMaNXmxFybG5H6GibrT1Vsx6Jm3sRQnQ2m2'
DB_URL = 'https://docs.google.com/spreadsheets/d/{0}/export?format=csv&id={0}&gid=0'.format(SHEET_ID)

app = Flask(__name__)

def url_for(short_name):
    csv = requests.get(DB_URL).text
    mappings = dict(mapping.split(',') for mapping in csv.split('\r\n'))
    return mappings[short_name]

@app.route('/<short_name>')
def handler(short_name):
    return redirect(url_for(short_name))

if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

It didn't speak for itself? No worries, here's what's happening:

Suppose I'm hosting my URL shortener at `example.com`.
When a request to `example.com/fb` comes in, Flask accepts the request and calls our `handler` function with `short_name` bound to the string `fb`.

`handler` then calls `url_for`, which downloads (using [Requests](http://docs.python-requests.org/en/master/)) our database spreadsheet in CSV format. It looks something like

```
g,http://google.com\r\nfb,http://facebook.com\r\ncs,http://cs61a.org
```

We convert this into a dict, then get the URL corresponding to `short_name`.
The result is passed to Flask's [`redirect`](http://flask.pocoo.org/docs/api/#flask.redirect), which ultimately redirects the user's browser the specified URL.

And that's it! Find the complete instructions and code on GitHub [here](https://github.com/guoguo12/gsheets-url-shortener).

## Bonus missions

Some homework for the extra eager:

* **It's dangerous to roll your own.** Find a way to make the server crash while parsing the CSV.
Then use Python's [`csv` module](https://docs.python.org/3/library/csv.html) to parse the database instead.
* **Human-centered design.** When a user requests a short URL that isn't in the database, our server responds with an ugly "Internal Server Error" page. Handle this gracefully.
* **Need for speed.** Cache database queries using [`functools.lru_cache`](https://docs.python.org/3/library/functools.html#functools.lru_cache). What are the advantages and disadvantages of this system? Write your response as a haiku.
* **Count your blessings.** Our client (yep, we have a client now) would like to track how many times each short URL has been accessed. Implement this feature. (Bonus points for writing click-count data back to the spreadsheet.)
* **Open sesame.** Our client would like to password-protect certain short URLs by specifying passwords in the spreadsheet. Implement this feature.
* **We should have used Java.** Add [type hints](https://docs.python.org/3/library/typing.html) to the code.
* **We should have used $language.** Rewrite the program in $language, unless $language is PHP.

If you complete any of these, make a pull request to the GitHub repo linked above!
