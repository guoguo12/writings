---
layout: post
title: A Concise URL Shortener in Python
modified: 2017-10-28
---

*How concise?* 14 lines.

*How?* The main ideas:
* Use Google Sheets as a database.
* Use [Flask](http://flask.pocoo.org) for serving and routing.

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

## Server setup

The server is responsible for converting short URLs into long URLs.

Say your server is hosted at `ag.io`.
When a user requests `ag.io/fb`, your server must extract the "path" part of the URL (which is just `fb`),
look up the corresponding long URL in the database,
then redirect the user to that URL.

Using Flask, this looks like:

```python
from flask import Flask, redirect

app = Flask(__name__)

@app.route('/<path>')
def handler(path):
    # We must define look_up_long_url
    return redirect(look_up_long_url(path))

if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

<!-- Note that we use Flask's [`redirect`](http://flask.pocoo.org/docs/api/#flask.redirect) to redirect the user's browser the long URL. -->

How do we implement `look_up_long_url`? The steps required are:

1. Download the database (spreadsheet) using the trick above.
2. Transform the data into a Python dict.
3. Look up the short URL in the dict.

In code (using [Requests](http://docs.python-requests.org/) to download the database):

```python
import requests

SHEET_ID = '1vr6C7CD_RMaNXmxFybG5H6GibrT1Vsx6Jm3sRQnQ2m2'
DB_URL = 'https://docs.google.com/spreadsheets/d/{0}/export?format=csv&id={0}&gid=0'.format(SHEET_ID)

def look_up_long_url(path):
    csv = requests.get(DB_URL).text
    mappings = dict(mapping.split(',') for mapping in csv.split('\r\n'))
    return mappings[path]
```

And that's it! See [GitHub](https://github.com/guoguo12/gsheets-url-shortener) for the full code and instructions on setting up the server.
