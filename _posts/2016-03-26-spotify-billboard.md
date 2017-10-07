---
layout: post
title: Spotify and billboard.py
---

_(Note: As of mid-2017, Billboard no longer publishes Spotify links with their charts, so the code in this article won't work.)_

_(Disclaimer: I am the maintainer and original author of billboard.py.)_

[billboard.py](https://github.com/guoguo12/billboard-charts) is a Python library for exploring _Billboard_ music ranking charts. You can download a chart...

```python
>>> import billboard
>>> chart = billboard.ChartData('hot-100', date='2016-03-26')
```

...and print it...

```python
>>> print chart
# hot-100 chart from 2016-03-26
# -----------------------------
# 1. 'Work' by Rihanna Featuring Drake (0)
# 2. 'Love Yourself' by Justin Bieber (0)
# 3. 'Stressed Out' by twenty one pilots (0)
# 4. 'My House' by Flo Rida (0)
# 5. '7 Years' by Lukas Graham (+4)
# ...
```

...and access detailed information about its entries.

```python
>>> song = chart[0]  # Get no. 1 song on chart
>>> song.title + ' by ' + song.artist
u'Work by Rihanna Featuring Drake'
>>> song.weeks  # Get no. of weeks on chart
7
>>> song.spotifyLink
u'https://embed.spotify.com/?uri=spotify:track:50R3hdsZQeg3k8zJdVphSZ'
```

That last feature is pretty important: since we can connect songs on the _Billboard_ charts to their corresponding Spotify tracks, we can use billboard.py to create something we can actually listen to.

In this demo, we'll do just that: we'll create a Spotify playlist containing the past 100 songs that have reached the top spot on the [Hot 100](http://www.billboard.com/charts/hot-100#/charts/hot-100).

### Walking back in time

The Hot 100 chart, which measures the popularity of singles in the US, has been published by _Billboard_ since the late 1950s. They release an updated chart once per week.

One of the nice features of billboard.py is the ability to jump from one week's chart to the previous week's chart:

```python
>>> chart = billboard.ChartData('hot-100', date='2016-03-26')
>>> prev_chart = billboard.ChartData('hot-100', date=chart.previousDate)
>>> prev_chart.date
u'2016-03-19'
```

To collect the past 100 top singles, we'll simply walk backwards in time, adding the Spotify URL of the top song from each chart to a set. For ease of use, we can formulate this process as a generator:

```python
def unique_top_tracks_generator():
    seen_tracks = set()
    chart = billboard.ChartData('hot-100')  # Start at most recent

    while len(seen_tracks) < 100:
        top_track = chart[0]
        duplicate = top_track.spotifyLink in seen_tracks

        if not duplicate:
            seen_tracks.add(top_track.spotifyLink)
            yield top_track

        chart = billboard.ChartData('hot-100', date=chart.previousDate)

    raise StopIteration
```

To make sure it works, let's look at the titles of the first few tracks the generator returns:

```python
>>> g = unique_top_tracks_generator()
>>> next(g).title
u'Work'
>>> next(g).title
u'Love Yourself'
>>> next(g).title
u'Pillowtalk'
>>> next(g).title
u'Sorry'
```

Seems right to me! We can check it against Wikipedia's [list of number-one singles](https://en.wikipedia.org/wiki/List_of_Billboard_Hot_100_number-one_singles_of_2016), just to make sure.

### Creating a playlist

To create the actual playlist, we'll use [Spotipy](https://spotipy.readthedocs.org), a Python wrapper around the [Spotify Web API](https://developer.spotify.com/web-api/). We'll also need to make a [Spotify Developer](https://developer.spotify.com/) account and [register an app](https://developer.spotify.com/my-applications/#!/applications/create) with the API. This will give us a pair of API keys to use during authorization.

At the top of our Python script, we set the following constants:

```python
SPOTIFY_USERNAME      = 'your-spotify-username'
SPOTIFY_CLIENT_ID     = 'your-client-id'
SPOTIFY_CLIENT_SECRET = 'your-client-secret'
SPOTIFY_REDIRECT_URI  = 'https://github.com/guoguo12/billboard-charts'
SPOTIFY_SCOPE         = 'playlist-modify-public'
```

Let's break these down:

*   `SPOTIFY_USERNAME` is the account the playlist will be created under, as well as the Spotify Developer account.
*   `SPOTIFY_CLIENT_ID` and `SPOTIFY_CLIENT_SECRET` are the two app keys mentioned above.
*   `SPOTIFY_REDIRECT_URI` is the URL the user will be directed to after the authorization process. Here it's set to the GitHub URL of the billboard.py project, but it can really be any URL, since we'll be the only "users" seeing it.
*   `SPOTIFY_SCOPE` contains the single permission we'll need to create and modify a public playlist.

We can now go through the standard Spotipy authorization process (described in more detail [here](https://spotipy.readthedocs.org/en/latest/#authorized-requests)):

```python
token = spotipy.util.prompt_for_user_token(
    SPOTIFY_USERNAME,
    scope=SPOTIFY_SCOPE,
    client_id=SPOTIFY_CLIENT_ID,
    client_secret=SPOTIFY_CLIENT_SECRET,
    redirect_uri=SPOTIFY_REDIRECT_URI)
```

When we run this chunk of code, Spotipy will walk us through an interactive authorization process. A message like this will appear in terminal:

> User authentication requires interaction with your web browser. Once you enter your credentials and give authorization, you will be redirected to a url. Paste that url you were directed to to complete the authorization.

After authorization finishes, we'll be able to use the `token` returned by `prompt_for_user_token()` to create a `Spotify` object, which we can then use to create our playlist:

```python
sp = spotipy.Spotify(auth=token)
playlist = sp.user_playlist_create(SPOTIFY_USERNAME, 'Best of the Hot 100')
```

Now we just have to loop over the generator we created earlier and add the tracks to the playlist!

```python
for track in unique_top_tracks_generator():
    url = track.spotifyLink
    sp.user_playlist_add_tracks(SPOTIFY_USERNAME, playlist_id, [url])
```

### Final result

How did our playlist turn out? Let's take a look:

<iframe src="https://embed.spotify.com/?uri=spotify:user:lightningatdawn:playlist:5vN7CQx3GbYN2cyNWR97X1" width="500" height="580" frameborder="0" allowtransparency="true" style="margin: 0px auto; display: block;"></iframe>

Not bad at all! Luckily, billboard.py and Spotipy did a lot of the heavy lifting for us.

If mainstream music isn't your thing, don't fret: the code that generated the playlist above can easily be adapted to work with other _Billboard_ charts as well. Check it out at [GitHub](https://github.com/guoguo12/billboard-charts/tree/master/examples/top-singles-playlist).
