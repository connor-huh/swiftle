
date: 2025-07-25
categories: [projects, automation, taylor-swift]
---

# Beating Swiftle with Python and PyAutoGUI

## Introduction
If you know anything about me, you probably know I've listened to my fair share of Taylor Swift. In 2023 alone, I accumulated over 1100 hours of her music on Spotify, contributing to more than 80% of my total listening time. Over 250 of these hours were spent solely listening to _All Too Well (10 Minute Version) (Taylor's Version)_. This level of commitment landed me in the top 0.01% of Taylor Swift listeners globally. I was even able to see her perform live during the _Era's Tour_ earlier that year.

My obsession with Swift's music began through imitation. Growing up, I always looked up to my sister, whose interests were my blueprint. When she became enchanted by Taylor Swift, I followed suit. By eighth grade, many knowledge of her songs became a sort of social currency, and many of my friendships stemmed from a shared love for her music. It even led to a Taylor Swift-centric group chat that persists today, despite my switching schools for ninth grade.

Given this, you'd expect me to be good at [Swiftle Rapid](https://www.techyonic.co/swiftle/rapid), a game that requires users to correctly identify the title of a Taylor Swift song snippet in ten seconds or less. But strangely enough, I'm awful at it. Often, I recognize the melody, but my mind becomes a blank space when grasping for its title. Other times, I'm unfamiliar with the song entirely. Though I've listened to her entire catalog before, my listening habits are heavily skewed toward my favorites, leaving a vast number of tracks almost untouched. I know the lyrics to many of her songs by heart; it's just not always the right ones. While some of my friends have streaks well into the triple digits, it took me multiple tries just to break a streak of five.

Not exactly a great look for a top 0.01% Taylor Swift fan...

Ever since I learned how dreadful I was at this game, I've dreamed of writing a program that could beat the game for me. Over the past few days, I finally brought my wildest dream to life.

The pipeline was conceptually simple:
1. Capture the audio snippet from Swiftle.
2. Use a music recognition API to identify the song.
3. Click the search bar, type the answer, and hit Enter.

I hoped to automate the process entirely, meaning I could let the script run in the background without monitoring it. 

## Build
Despite the apparent simplicity of the idea, actually building it involved a decent amount of debugging and domain-specific adjustments. I knew it would rely significantly on external packages, most of which I have no knowledge of. Thus, a large portion of my preliminary work was searching for packages online that could contribute to my script.

The first step was to figure out how to record the Swiftle audio snippet automatically. I used sounddevice to capture audio from my computer. To capture internal audio on my Macbook, I installed a virtual input device called BlackHole. After selecting the correct device index in Python, I was ready to record.

```python
def record_audio(filename='snippet.wav'):
    recording = sd.rec(int(DURATION * FS), samplerate=FS, channels=2, device=DEVICE_INDEX)
    sd.wait()
    write(filename, FS, recording)
```

I opted to record the browser audio for 2.5 seconds, which proved to be enough time for the script to recognize the song. While a prolonged time would theoretically increase the chances of accurately identifying the song, it left less time for the script to process everything and enter the answer.

Once I had the audio, the next step was to identify the song. I used [Audd.io](https://audd.io/), a simple but surprisingly powerful API for music recognition. The interface is pretty clean: you post the file, include your API token, and get a JSON response with the song metadata (if recognized). I had it return the Spotify data as well, just in case, though I didn’t end up needing it.

```python
def identify_song(filename='snippet.wav'):
    with open(filename, 'rb') as f:
        files = {'file': f}
        data = {'api_token': API_TOKEN, 'return': 'spotify'}
        response = requests.post('https://api.audd.io/', data=data, files=files)
    result = response.json()
    return result['result'].get('title', '').strip()
```

Audio recognition APIs, as it turns out, are surprisingly sensitive. If your file is even slightly too short, too long, or noisy, you’re out of luck. Fortunately, Audd.io did well, recognizing snippets from any part of even the most obscure Taylor Swift vault tracks. However, I would be surprised if it maintained this level of accuracy for non-Taylor Swift songs, which often sample other tracks.

## Answer Submission

Swiftle doesn't have an API, and I didn't want to even attempt to reverse-engineer its frontend. So instead, I opted to use brute-force automation. The script needed to move the mouse to the search bar coordinates on my screen, click down, type the name of the song, and press Enter to submit. To determine the location of the search bar on my computer, I ran a separate program using PyAutoGUI. Once executed, the program waits three seconds for me to hover my cursor over Swiftle's search bar before printing the cursor's coordinates. Though I was unfamiliar with the package at the start of this project, it turned out to be relatively simple.

```python
time.sleep(3)
x, y = pyautogui.position()
print(x, y)
```

I also needed to reformat the titles for certain songs. The audio recognition API returned some song titles with superfluous information, such as "(Taylor's Version)" or "(feat. Lana Del Rey)," which are indeed useful but aren't recognized by Swiftle, as a human user would likely have trouble distinguishing these. This was a simple fix, as I just needed to remove anything from the titles contained in parentheses.

```python
def clean_title(title):
    return re.sub(r"\s*\(.*?\)", "", title).strip()
```

Finally, I proceeded with programming the keyboard commands via AppleScript and cliclick.

```python
def submit_song(song_title):
    clean = clean_title(song_title)
    x, y = SEARCH_BAR_COORDS
    apple_script = f'''
    do shell script "cliclick m:{x},{y} c:{x},{y}"
    delay 0.3
    tell application "System Events"
        keystroke "{clean}"
        delay 0.4
        key code 125 -- down arrow
        delay 0.1
        key code 36  -- return key
    end tell
    '''
```

_Note that the coordinates were hardcoded based on my screen layout. If you run this, you'll need to adjust them according to your device, which can be done using the coordinate identification method I mentioned earlier._


