# Music Organization & Workflow

## The Mission

I recently began to overhaul the way I organize music for listening and DJing purposes. I've been talking about this for literally years, but got a new laptop a couple weeks ago and decided it was finally time. For some context, I currently use macOS as my workstation OS but may switch to Linux, at least in part, in the future. I also DJ using Pioneer's CDJ/XDJ players and want to be able to identify music for DJing as part of my daily listening process and have a simple method for getting music I want to play in DJ sets into Rekordbox and, preferably, be able to build playlists while listening and transfer those to Rekordbox as well. I also use Plex at home for listening to music around the house on a Sonos and sometimes while on the road.

So, with all that in mind, my primary goals/requirements are as follows:

1. Store music in my primary library in a lossless format (FLAC) for both listening and archival purposes
2. Have a simple, reasonably automated process for syncing/exporting a subset of my collection to a dedicated Rekordbox library, with FLACs converted to 16-bit/44.1kHz AIFF for broadest compatability with CDJ models
3. Be able to construct playlists for DJing purposes while listening at my desk and export those in an automated fashion to Rekordbox

Above all, I do not want to ever have to drag and drop individual tracks into Rekordbox. This is annoying and error-prone and it makes pulling music for DJing a total drag.

## The Components

I've put together a system using the following primary software components:

* [beets](http://beets.io)
* [Swinsian](https://swinsian.com)
* [Rekordbox](https://rekordbox.com/en/)

Additionally, I've written some scripts and glued things together with:

* [AppleScript](https://developer.apple.com/library/archive/documentation/AppleScript/Conceptual/AppleScriptLangGuide/introduction/ASLR_intro.html)
* [Bash](https://www.gnu.org/software/bash/)
* [Hazel](https://www.noodlesoft.com)

## The Workflow

Here is my beets config file, `~/.config/beets/config.yaml`:

```yaml
# This is where I store my primary library and the beets SQLite DB
directory: ~/Music/Library
library: ~/Music/Library/beets.db

plugins:
  # I use beets-alternatives to manage the Rekordbox library
  - alternatives

  # The convert plugin is used to convert incoming stuff to FLAC as well as
  # converting to AIFF (with ffmpeg) when exporting to Rekordbox
  - convert

  # Use discogs.com as a source for metadata
  - discogs

  # Provides the ability to manually edit metadata
  - edit

  # Fetch and embed artwork
  - embedart
  - fetchart

  - info

paths:
  default: $albumartist/$album%aunique{}/$track $title
  singleton: $artist/Miscellaneous/$title
  comp: Various Artists/$album%aunique{}/$track $title

convert:
  auto: yes
  never_convert_lossy_files: yes
  format: flac
  formats:
    rekordbox:
      # This converts to 16-bit/44.1kHz AIFF with metadata intact
      command: ffmpeg -i $source -y -acodec pcm_s16be -sample_fmt s16 -ar 44100 -write_id3v2 1 $dest
      extension: aiff

alternatives:
  rekordbox:
    directory: /Users/jd/Music/Rekordbox
    query: "rekordbox:true"
    # Convert to AIFF unless the source file is an MP3
    formats: rekordbox mp3

discogs:
  source_weight: 0.0

fetchart:
  auto: yes
  minwidth: 500
```

First, I use beets to automatically tag music an album or release at a time, convert it to FLAC (unless it's in a lossy format, like MP3), and send it to my primary library directory, `~/Music/Library`. I do this by running `beet import <path/to/album>`.

I then have Swinsian configured with `~/Music/Library` as a watched directory so that whenever beets adds or updates tracks in that directory, Swinsian will pick them up within a few seconds and I can listen and begin building playlists for DJing.

Once I'm ready to move stuff from Swinsian to Rekordbox, I can select a bunch of tracks (say, the entire playlist I've been building) in Swinsian, and run this AppleScript called `Selected Tracks To Rekordbox`.

```applescript
tell application "Swinsian"
	# Get the currently selected tracks in Swinsian
	set selected to selection of window 1

	if selected is not {} then
		set c to count of selected's items

		repeat with i from 1 to c
			set thetrack to item i of selected
			set filePath to quoted form of location of thetrack

			# Here we get the current track comments and append rb:true to them if it
			# isn't already there. This is primarily so I can create a smart playlist
			# in Swinsian if I want to see tracks that are or are not currently marked
			# for Rekordbox
			set query to "path:" & filePath
			set cmd to "/usr/local/bin/beet ls -f '$comments' " & query
			set comments to do shell script cmd

			if comments does not contain "rb:true" then
				set comments to comments & " rb:true"
			end if

			# Here is where we actually write the new comments to the beets library and
			# also set an additional rekordbox=true field, which is what I actually use
			# to query when deciding what to export to the Rekordbox library.
			set cmd to "/usr/local/bin/beet modify -y comments=\"" & comments & "\" rekordbox=true " & query

			do shell script cmd
		end repeat
	end if
end tell
```

At this point, nothing has actually been sent to the Rekordbox library, but the selected tracks have been tagged with `rekordbox:true`. Next, I run `beet alt update rekordbox`, which uses the [beets-alternatives](https://github.com/geigerzaehler/beets-alternatives) plugin, along with my `rekordbox` format configuration for the convert beets plugin, to export any files tagged with `rekordbox:true` that are new or have changed since I last ran `beet alt update rekordbox`. At this point, anything I've tagged for Rekordbox should now exist in my Rekordbox library directory, `~/Music/Rekordbox`.

I haven't quite figured out how to script adding new files to Rekordbox at this point and Rekordbox doesn't have a "Watch Directory" function, so at this point, I need to manually drag my `~/Music/Rekordbox` folder onto the Collection in Rekordbox which is, fortunately, smart enough to only add new stuff. I haven't actually tested to see if it's smart enough to update stuff, though I kind of doubt it, and I'm pretty sure it won't actually delete things from Rekordbox if the files were deleted (which would happen if I removed the `rekordbox:true` tag from something).

Once I've dragged the Rekordbox folder into the Rekordbox Collection, Rekordbox will start analyzing those tracks and they will be playable.

Then, if I want to move a playlist, I Ctrl+click the playlist in Swinsian, select "Export…", and save it to a folder I've created called `~/Music/Actions/SendPlaylistToRekordbox`. I have a Hazel rule configured that watches this directory for playlist files with a `.m3u8` extension and then runs this shell script, passing the file path of the playlist to the script as the first argument:

```bash
#!/usr/bin/env bash
set -e

playlist=$1

REKORDBOX_LIBRARY=$(echo "${HOME}/Music/Rekordbox" | sed 's/\//\\\//g')

# Rewrite filenames to match what has been exported to Rekordbox
sed -i '' 's/\.flac/\.aiff/g' "$playlist"
sed -i '' 's/^.*Library/'${REKORDBOX_LIBRARY}'/g' "$playlist"

# This is a gross hack that uses AppleScript GUI scripting to click around in
# the Rekordbox GUI and import the edited playlist.
osascript <<EOF
tell application "System Events"
	tell process "rekordbox"
		set frontmost to true
		set playlistPath to "$playlist"
		click menu item "Import Playlist" of menu of menu item "Import" of menu "File" of menu bar 1
		delay 1
		keystroke "g" using {shift down, command down}
		keystroke playlistPath
		delay 1
		keystroke return
	end tell

	tell window 1 of process "rekordbox"
		click button "Open"
	end tell
end tell
EOF

# Delete the exported playlist
rm "$playlist"
```

## The End… For Now

So, that's where I'm at currently. I just used this system when preparing a gig for the first time last weekend and it was, for me, at least, a HUGE improvement over manually dragging things into Rekordbox, even with some scripts to help with converting to AIFF that I had already. That said, there is definitely some room for improvement.

Things I'd like to continue to investigate/improve upon:

* Having to drag the Rekordbox folder into the Collection pane in Rekordbox is certainly not terrible, but it would be great if new things could pop up in Rekordbox without needing to do it
* Similarly, the GUI Scripting to import a playlist into Rekordbox works, but is error-prone.

Unfortunately, Rekordbox has no native AppleScript support, but Rekordbox 6 does have this `rekordboxAgent` helper program that [wraps an encrypted SQLite3 database](https://rekord.cloud/blog/technical-inspection-of-rekordbox-6-and-its-new-internals) with a REST HTTP API. It's intended to be used by Pioneer's cloud sync feature, but it *might* be possible to abuse it to add or update tracks and playlists.
