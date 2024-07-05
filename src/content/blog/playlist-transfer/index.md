---
title: "Automating Music Transfers Between YouTube and Spotify"
description: "Simple code workbook to migrate your playlists."
date: "Jul 03 2024"
---

Recently, my partner and I decided to rationalize our music subscriptions. She had been using Spotify, and I was on YouTube Music. Spotify's family program looked like it would work perfectly for us, but it meant I would lose all my playlists and favorite songs on YouTube.

Naturally, I wanted to transfer my playlists over from YouTube to Spotify.

A quick Google search showed there were a number of tools that could do this, but they all required me to grant quite permissive access to my accounts to a third-party tool that I wasn't completely sure of the provenance of. So, I decided to investigate how hard it would be to do it myself.

Wonderfully, both Spotify and YouTube Music have excellent API access available, and it's pretty straightforward to set up developer-style access to your account. Below are the steps you can take to try this yourself!

The guide below assumes some experience setting up a python coding environment.

## Setting Up the Environment

First things first, I needed to set up my development environment. I created a virtual environment to keep things organized and installed the necessary Python packages.

You should run the following in the shell you use for your development environment.

Here’s how I set it up:

Create a Virtual Environment:

```
python -m venv myenv
```

This command creates a virtual environment named myenv.

Activate the Virtual Environment:

On macOS/Linux:

```
source myenv/bin/activate
```

Install Required Packages:

```
pip install spotipy python-dotenv google-api-python-client google-auth google-auth-oauthlib google-auth-httplib2
```

This command installs all the necessary libraries you'll need for interacting with the Spotify and YouTube APIs.

## Authentication with APIs

To interact with both YouTube and Spotify, I needed to authenticate my application with their APIs. Both services have developer consoles to generate credentials to authorise your application via OAuth flows.

### Spotify Authentication

I used Spotipy for Spotify API authentication. Spotipy makes it easy to interact with the Spotify API in Python.

#### Setting Up Spotify API Credentials

**Sign Up for a Spotify Developer Account**:

- Go to the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard/login).
- Log in with your Spotify account. If you don’t have one, you’ll need to create one.

**Create a New Application**:

- Once logged in, click on the "Create an App" button.
- Fill in the required details, such as App name and App description. These can be anything you like.
- After filling in the details, click "Create".

**Get Your Client ID and Client Secret**:

- After creating the app, you will be redirected to your app's dashboard.
- Here, you’ll see your Client ID and Client Secret. Click on "Show Client Secret" to view it.

**Set Up Redirect URI**:

- In your app's dashboard, scroll down to the "Redirect URIs" section.
- Click on "Edit Settings".
- Add `http://localhost:8888/callback` as a Redirect URI.
- Click "Add" and then "Save".

**Create a `.env` File**:
Create a file named `.env` in your project directory and add your Spotify API credentials:

```
SPOTIFY_CLIENT_ID=your_spotify_client_id
SPOTIFY_CLIENT_SECRET=your_spotify_client_secret
SPOTIFY_REDIRECT_URI=http://localhost:8888/callback
```

**Load Environment Variables and Authenticate**:

The code below loads your Spotify credentials from the `.env` file and authenticates your application with the Spotify API.

```python
import spotipy
from spotipy.oauth2 import SpotifyOAuth
from dotenv import load_dotenv
import os

# Load environment variables from .env file

load_dotenv()

SPOTIFY_CLIENT_ID = os.getenv('SPOTIFY_CLIENT_ID')
SPOTIFY_CLIENT_SECRET = os.getenv('SPOTIFY_CLIENT_SECRET')
SPOTIFY_REDIRECT_URI = os.getenv('SPOTIFY_REDIRECT_URI')

sp = spotipy.Spotify(auth_manager=SpotifyOAuth(
  client_id=SPOTIFY_CLIENT_ID,
  client_secret=SPOTIFY_CLIENT_SECRET,
  redirect_uri=SPOTIFY_REDIRECT_URI,
  scope='playlist-read-private playlist-modify-public playlist-modify-private'
))
```

### YouTube Authentication

For YouTube, I used the Google API Client library. This library helps you interact with Google APIs using Python.

#### Download Credentials:

- Go to the Google Cloud Console.
- Create a new project and enable the YouTube Data API v3.
- Create OAuth 2.0 credentials and download the client_secret.json file.

#### Authenticate and Build the Service:

This function handles the authentication process, allowing you to interact with the YouTube API. When you run this cell, you'll be asked to click a link to start the OAuth flow and allow access to your youtube account.

```python
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build
from google.auth.transport.requests import Request
import pickle
import os

# Define the scope for YouTube - we are only reading existing info, so need the readonly scope

YOUTUBE_SCOPES = ['https://www.googleapis.com/auth/youtube.readonly']

def get_authenticated_service():
  creds = None
  token_path = 'token.pickle'

  # Load existing credentials from file if available
  if os.path.exists(token_path):
      with open(token_path, 'rb') as token:
          creds = pickle.load(token)

  # If there are no valid credentials available, request new ones
  if not creds or not creds.valid:
      if creds and creds.expired and creds.refresh_token:
          creds.refresh(Request())
      else:
          flow = InstalledAppFlow.from_client_secrets_file(
              'client_secret.json', YOUTUBE_SCOPES, redirect_uri='http://localhost:8888/'
          )
          creds = flow.run_local_server(port=8888)

      # Save the credentials for future use
      with open(token_path, 'wb') as token:
          pickle.dump(creds, token)

  return build('youtube', 'v3', credentials=creds)

youtube = get_authenticated_service()
```

## Fetching Liked Songs from YouTube

YouTube has a special playlist for your liked songs, which is not listed as a standard playlist. This playlist is identified by the ID `LL`. Here’s how I fetched my liked songs:

**Define a Function to Fetch Liked Songs**:

```python
def fetch_liked_songs(youtube):
    songs = []
    request = youtube.playlistItems().list(
        part="snippet,contentDetails",
        playlistId='LL',
        maxResults=50
    )
    while request is not None:
        response = request.execute()
        songs.extend(response.get('items', []))
        request = youtube.playlistItems().list_next(request, response)
    return songs

liked_songs = fetch_liked_songs(youtube)
```

**Print the Liked Songs**:

This code retrieves and prints the details of your liked songs from YouTube.

```python
for song in liked_songs:
  print(f"Title: {song['snippet']['title']}, Video ID: {song['snippet']['resourceId']['videoId']}")
```

## Fetching Standard Playlists from YouTube

In addition to your liked songs, you might have created several playlists on YouTube. Here’s how to fetch these playlists:

**Define a Function to Fetch Playlists**:

```python
def fetch_playlists(youtube):
    playlists = []
    request = youtube.playlists().list(
        part="snippet,contentDetails",
        mine=True,
        maxResults=50
    )
    while request is not None:
        response = request.execute()
        playlists.extend(response.get('items', []))
        request = youtube.playlists().list_next(request, response)
    return playlists

playlists = fetch_playlists(youtube)
```

**Print the Playlists**:

This code retrieves and prints the details of your playlists from YouTube.
for playlist in playlists:

```python
print(f"Playlist: {playlist['snippet']['title']}, Playlist ID: {playlist['id']}")
```

**Fetch Songs from a Specific Playlist**:

```python
def fetch_songs_from_playlist(youtube, playlist_id):
    songs = []
    request = youtube.playlistItems().list(
        part="snippet,contentDetails",
        playlistId=playlist_id,
        maxResults=50
    )
    while request is not None:
        response = request.execute()
        songs.extend(response.get('items', []))
        request = youtube.playlistItems().list_next(request, response)
    return songs
```

## Creating Playlists on Spotify

With the liked songs and playlists fetched, the next step was to create a playlist on Spotify and add these tracks.

### Create a Playlist

**Define a Function to Create a New Playlist**:

```python
def create_spotify_playlist(sp, playlist_name, description=""):
  user_id = sp.current_user()['id']
  playlist = sp.user_playlist_create(user=user_id, name=playlist_name, public=True, description=description)
  return playlist
```

```python
# Example usage

playlist_name = "My New Playlist"
description = "A new playlist created via Spotipy"
new_playlist = create_spotify_playlist(sp, playlist_name, description)

print(f"Created playlist: {new_playlist['name']} with ID: {new_playlist['id']}")
```

### Add Tracks to Playlist

To add tracks to the playlist, I wrote a function to search for the track on Spotify and handle errors gracefully:

**Define a Function to Extract Artist and Title from YouTube Song Data**:

Due to the strcuture of the song data returned from the YouTube endpoint, we need to extract the song name and artist name.

The Song name is easily available in the `['snippet']['title']` dictionary entry, but the artist name is a bit more hidden.

The pattern I've observed is that it is stored in the `['snippet']['videoOwnerChannelTitle']` dictionary entry, but with a trailing " - Topic" string part. We create a helper function to extract this below.

```python
def extract_artist_name(channel_title):
    if " - Topic" in channel_title:
        return channel_title.split(" - Topic")[0]
    return channel_title
```

**Define a Function to Search for a Track**:

```python
def search_track(track_name, artist_name):
    query = f'track:{track_name} artist:{artist_name}'
    result = sp.search(q=query, type='track', limit=1)

    if result['tracks']['items']:
        track = result['tracks']['items'][0]
        track_info = {
            'name': track['name'],
            'artist': track['artists'][0]['name'],
            'album': track['album']['name'],
            'release_date': track['album']['release_date'],
            'uri': track['uri']
        }
        return track_info
    else:
        return None
```

### Batch Adding Tracks with Error Handling

To avoid hitting API rate limits, I batched the requests and handled any errors that occurred:

**Define a Function to Add Tracks in Batches**:

```python
import time

def add_tracks_to_playlist(sp, playlist_id, track_uris):
    for i in range(0, len(track_uris), 20):  # Batch size of 20
        batch = track_uris[i:i + 20]
        try:
            sp.playlist_add_items(playlist_id, batch)
        except Exception as e:
            print(f"Error adding batch: {str(e)}")
        time.sleep(1)  # Wait for 1 second between batches
```

## Wiring Up YouTube and Spotify

Now, let's wire up the playlists and songs we found on YouTube and use that to create playlists with songs on Spotify.

### Fetch Liked Songs and Create a Playlist on Spotify

Do the Liked Song playlist first.

**Fetch Liked Songs**:

```python
liked_songs = fetch_liked_songs(youtube)
```

**Create a New Playlist on Spotify**:

```python
playlist_name = "My YouTube Favorites"
description = "Favorite songs from YouTube, now on Spotify"
new_playlist = create_spotify_playlist(sp, playlist_name, description)
```

**Search and Add Songs to Spotify Playlist**:

```python
track_uris = []
for song in liked_songs:
    title = song['snippet']['title']
    artist = extract_artist_name(song['snippet']['videoOwnerChannelTitle'])
    print(f"Processing: {title} by {artist}")

    track_info = search_track(title, artist)
    if track_info:
        track_uris.append(track_info['uri'])
    else:
        print(f"Error: Track not found - {title} by {artist}")


# Add tracks to the new playlist in batches
if track_uris:
    try:
        add_tracks_to_playlist(sp, new_playlist['id'], track_uris)
        print(f"Added {len(track_uris)} tracks to playlist: {new_playlist['name']}")
    except Exception as e:
        print(f"Error adding tracks to playlist: {str(e)}")
else:
    print("No valid tracks to add to the playlist.")
```

### Fetch Standard Playlists and Create Corresponding Playlists on Spotify

Now transfer the standard playlists.

**Fetch Standard Playlists**:

```python
playlists = fetch_playlists(youtube)
```

**Create Spotify Playlists from YouTube Playlists**:

```python
for playlist in playlists:
    playlist_name = playlist['snippet']['title']
    playlist_id = playlist['id']
    print(f"Processing playlist: {playlist_name}")

    # Create a new playlist on Spotify
    new_spotify_playlist = create_spotify_playlist(sp, playlist_name, f"Playlist from YouTube: {playlist_name}")

    # Fetch songs from the YouTube playlist
    songs = fetch_songs_from_playlist(youtube, playlist_id)

    # Search and add songs to the Spotify playlist
    track_uris = []
    for song in songs:
        title = song['snippet']['title']
        artist = extract_artist_name(song['snippet']['videoOwnerChannelTitle'])
        print(f"Processing: {title} by {artist}")

        track_info = search_track(title, artist)
        if track_info:
            track_uris.append(track_info['uri'])
        else:
            print(f"Error: Track not found - {title} by {artist}")

    # Add tracks to the new Spotify playlist in batches
    if track_uris:
        try:
            add_tracks_to_playlist(sp, new_spotify_playlist['id'], track_uris)
            print(f"Added {len(track_uris)} tracks to playlist: {new_spotify_playlist['name']}")
        except Exception as e:
            print(f"Error adding tracks to playlist: {str(e)}")
    else:
        print("No valid tracks to add to the playlist.")
```

## Summary

This project streamlined the process of transferring liked songs and playlists from YouTube to Spotify playlists. By automating the authentication, fetching, and playlist creation steps, I saved a lot of manual effort.

The `search_track` function is relatively naive and looks for exact matches only.

I've found that it will occasionally fail to find a match especially if there are unusual characters in the artist name or there are multiple versions of the song which the platforms capture slightly differently. But you can see these ones in the console output labelled: `Error: Track not found...` so can just add them manually.

Hope this helps others in a similar situation!
