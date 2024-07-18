#!/usr/bin/env python3
# Copyright 2024 Raffaele Mancuso
# SPDX-License-Identifier: MIT
# Inspired by: https://github.com/Alexays/Waybar/blob/
#              master/resources/custom_modules/mediaplayer.py
import argparse
import io
import json
import logging
import math
import re
import shutil
import subprocess
import sys
import time
import urllib
from collections import namedtuple
from functools import partial
from pathlib import Path

import gi
import requests
import syncedlyrics
from PIL import Image

gi.require_version("Playerctl", "2.0")
from gi.repository import GLib, Playerctl  # noqa: E402

# Internal global variables
config = None
logger = logging.getLogger(__name__)
artfp = Path("/tmp/waybar-mediaplayer-art")
last_metadata = None
last_art_url = None
last_rot = 0
last_rot_time = 0
last_notification_time = 0
# Whether refresh_interval callback is registered
is_refint = False
format_class = None
icons = {"play": " ", "stop": " "}
is_text_rotating = None
lyrics = dict()


def on_player_appeared(manager, player, requested_player):
    logger.debug("I was called")
    if player and (player.name == requested_player):
        init_player(manager, player)
    else:
        logger.debug(
            "New player appeared, but it's not the selected player, skipping"
        )

    logger.debug("Returning True")
    return True


def delete_album_art():
    p = Path(__file__)
    if config["album_art_placeholder"] == "dark":
        phfp = p.parents[1] / "assets" / "music_black.png"
    elif config["album_art_placeholder"] == "light":
        phfp = p.parents[1] / "assets" / "music_white.png"
    elif config["album_art_placeholder"] == "no":
        phfp = None
    else:
        raise Exception(
            f"ERROR: Invalid value {config['album_art_placeholder']} "
            "for `album_art_placeholder` option"
        )
    if phfp:
        shutil.copy(phfp, artfp)
    else:
        if artfp.is_file():
            artfp.unlink()
    signal_album_art_change()


def on_player_vanished(manager, player):
    logger.info("I was called")
    write_output()
    delete_album_art()
    logger.info("Returning True")
    return True


def on_metadata(player, metadata, manager):
    logger.debug("I was called")
    update_metadata(player, metadata, manager)
    logger.debug("Returning True")
    return True


def on_playback_status(player, status, manager):
    logger.info("I was called")
    logger.debug(f"status={status}")
    # Update the icon in the progressbar (play/pause)
    logger.debug("Calling register_refresh_interval_callback")
    register_refresh_interval_callback(player, manager)
    logger.debug("Calling update_progressbar")
    update_progressbar(manager, player)
    logger.debug("Returning True")
    return True


def on_refresh_interval(manager):
    global is_refint
    logger.debug("I was called")
    # If there are no players, stop calling this handler
    if len(manager.props.players) == 0:
        logger.debug("Returning False")
        is_refint = False
        return False
    player = manager.props.players[0]
    # If we are not playing, stop calling this handler
    is_playing = player.props.status == "Playing"
    if not is_playing:
        logger.debug("Returning False")
        is_refint = False
        return False
    # Update progressbar and return
    update_progressbar(manager, player)
    logger.debug("Returning True")
    return True


def write_output(text=None, class_=None, tooltip=""):
    global config
    # Default values if not explicitly set
    if not text:
        text = "" + (" " * (config["widget_length"] - 1))
    if not class_:
        class_ = gen_class(0.0)
    if not tooltip:
        tooltip = ""
    # HTML entities need to be escaped
    text = text.replace("&", "&amp;")
    tooltip = tooltip.replace("&", "&amp;")
    # Write to stdout
    output = {
        "text": text,
        "class": class_,
        "tooltip": "<span>" + tooltip + "</span>",
    }
    sys.stdout.write(json.dumps(output) + "\n")
    sys.stdout.flush()


def register_refresh_interval_callback(player, manager):
    global is_refint
    logger.info("I was called")
    is_playing = player.props.status == "Playing"
    if is_playing and (not is_refint):
        logger.debug("Registering `refresh_interval` callback")
        GLib.timeout_add(
            config["refresh_interval"], on_refresh_interval, manager
        )
        is_refint = True


def rotate_str_left(s, rot):
    return s[rot:] + s[:rot]


def gen_widget_text(player):
    global last_rot, last_rot_time, is_text_rotating

    metadata = player.props.metadata
    # artist = player.get_artist()
    # artist = metadata["xesam:artist"][0]
    title = player.get_title() or ""
    # title = metadata["xesam:title"]

    icon = icons["play"] if player.props.status != "Playing" else icons["stop"]

    if (
        player.props.player_name == "spotify"
        and "mpris:trackid" in metadata.keys()
        and ":ad:" in metadata["mpris:trackid"]
    ):
        song_info = "AD PLAYING"
    else:
        song_info = title.strip()

    if not is_text_rotating:
        text = left_text(icon + song_info)
    else:
        song_info += " " + config["sepchar"] + " "
        elapsed = (time.time() - last_rot_time) * 1000
        rot_needs_update = elapsed > config["text_rot_int"]
        if rot_needs_update:
            to_add = math.floor(elapsed / config["text_rot_int"])
            l_rot = (last_rot + to_add) % len(song_info)
            last_rot = l_rot
            last_rot_time = time.time()
        else:
            l_rot = last_rot
        song_info = rotate_str_left(song_info, l_rot)
        text = icon + song_info

    return text


def signal_album_art_change():
    cmd = ["pkill", "-RTMIN+" + str(config["image_signal"]), "waybar"]
    res = subprocess.run(cmd)
    logger.debug(f"pkill returned: {res}")


def download_art(art_url):
    logger.debug("Updating album art")
    if art_url.startswith("http"):
        resp = requests.get(art_url)
        with open(artfp, "wb") as fh:
            fh.write(resp.content)
        logger.debug(
            f"Album art '{art_url}' downloaded and saved into '{artfp}'"
        )
    elif art_url.startswith("file://"):
        art_url = art_url[7:]
        art_url = urllib.parse.unquote(art_url)
        # symlinking the file doesn't work
        # apparently symlinks are not followed
        # artfp.symlink_to(art_url)
        shutil.copy(art_url, artfp)
        logger.debug(f"Album art '{art_url}' copied to '{artfp}'")
    elif Path(art_url).is_file():
        shutil.copy(art_url, artfp)
        logger.debug(f"Album art '{art_url}' copied to '{artfp}'")
    else:
        logger.debug(
            f"Invalid schema for {art_url}. Please report this as a bug."
        )
        return
    # Convert image to JPEG
    if config["convert_to_jpeg"]:
        artfp2 = artfp.parent / (artfp.stem + ".jpeg")
        with Image.open(artfp) as im:
            im.save(artfp2)
        shutil.move(artfp2, artfp)
    # Update album art on bar
    signal_album_art_change()


def send_notification(summary, body):
    global last_notification_time
    cmd = [
        "notify-send",
        "--app-name=waybar-mediaplayer",
        "--icon=" + artfp,
        summary,
        body,
    ]
    logger.error(f"Running {cmd}")
    subprocess.run(cmd)
    last_notification_time = time.time()


def update_metadata(player, metadata, manager):
    global last_notification_time, last_art_url, is_text_rotating
    logger.info("I was called")
    metadata = dict(metadata)
    # Update progressbar
    updated = update_progressbar(manager, player)
    if not updated:
        logger.debug("Progress bar not updated. Returning")
        return False
    # Download album art
    art_url = metadata.get("mpris:artUrl", None)
    if art_url and art_url != last_art_url:
        download_art(art_url)
        last_art_url = art_url
    # Notification
    notexp = time.time() - last_notification_time
    if config["is_notification"] and (
        notexp >= config["notification_min_interval"]
    ):
        send_notification(player.get_title(), player.get_artist())
    # Does text need rotation?
    title = player.get_title() or ""
    is_text_rotating = (
        len(title) + len(icons["stop"]) + 1 > config["widget_length"]
    )
    # Lyrics
    artist = player.get_artist() or ""
    get_new_lyrics(artist, title)
    logger.debug("Returning")


def download_lyrics(artist, title):
    stream = io.StringIO()
    lyrlogger = syncedlyrics.logger
    lyrlogger.propagate = False
    lyrlogger.handlers.clear()
    lyrlogger.addHandler(logging.StreamHandler(stream))
    lyrlogger.setLevel(logging.INFO)
    try:
        lt = syncedlyrics.search(
            f"{artist} {title}",
            synced_only=True,
            providers=config["lyrics_providers"],
        )
    except Exception:
        logger.debug(
            "ERROR: syncedlyrics.search threw an exception. Returning None.")
        return None
    if lt is None:
        logger.debug(
            "ERROR: syncedlyrics.search returned None. Returning None.")
        return None
    lt = lt.replace("\n", "")
    lt = lt.split("[")
    lt = ["["+x for x in lt if x]
    provider = stream.getvalue().split("on ")[1].strip()
    stream.close()
    SyncedLyrics = namedtuple("SyncedLyrics", ["lt", "provider"])
    return SyncedLyrics(lt, provider)


def get_new_lyrics(artist, title):
    global lyrics
    lyrics = {
        "secs": [0],
        "text": ["[INTRO]"],
        "last_ix": -1,
        "max_len": 0,
        "provider": None,
    }
    lyr = download_lyrics(artist, title)
    if lyr is None:
        logger.debug("ERROR: download_lyrics returned None. Returning.")
        lyrics = None
        return
    lyrics["provider"] = lyr.provider
    for i, line in enumerate(lyr.lt):
        m = re.match(r"\[(.*)\](.*)", line)
        if not m:
            continue
        # convert time from min:sec into sec
        timing = m.group(1)
        timing2 = timing.split(":")
        if len(timing2) != 2:
            logger.debug(f"ERROR: timing2={timing2} is not of length 2")
            continue
        try:
            secs = int(timing2[0]) * 60.0 + float(timing2[1])
        except ValueError:
            logger.debug(f"ERROR: ValueError while converting timing2={
                         timing2} to seconds")
            continue
        # get lyrics text
        text = m.group(2).strip()
        if text == "":
            text = "[INTERMISSION]"
        lyrics["secs"].append(secs)
        lyrics["text"].append(text)
        lyrics["max_len"] = max(lyrics["max_len"], len(text))
        # DEBUG
        logger.debug(str(i) + str(line) + "->" + "[" + str(secs) + "] " + text)
    lyrics["text"].append("[OUTRO]")
    lyrics["secs"].append(lyrics["secs"][-1] + 1)


def left_text(s):
    le = len(s)
    if le > config["widget_length"]:
        return s
    missing = config["widget_length"] - le
    s = s + " " * (missing)
    return s


def center_text(s, ml=None):
    if not ml:
        ml = max([len(x) for x in s])
    for i in range(0, len(s)):
        delta = max(math.floor((ml - len(s[i])) / 2), 0)
        s[i] = (" " * delta) + s[i] + (" " * delta)
    return s


def gen_class(perc):
    if not isinstance(perc, float):
        raise Exception(f"{perc} ({type(perc)}) is not a float")
    return "perc" + format_class(perc).replace(".", "-")


# Called
# 1. By `on_refresh_interval`
# 2. By `on_metadata`
# 3. By `on_playback_status`
def update_progressbar(manager, player):
    global last_metadata, lyrics
    logger.debug("I was called")
    # Get percentage in the track
    pos = player.get_position()
    logger.debug(f"pos={pos}")
    # If it doesn't work, use CLI utility
    if pos == 0:
        cmd = ["playerctl", "--player=" + player.props.player_name, "position"]
        pos = subprocess.run(cmd, capture_output=True, text=True).stdout
        pos = float(pos) * (10**6)
        logger.debug(f"pos from CLI={pos}")
    pmetadata = player.props.metadata
    logger.debug(f"pmetadata={pmetadata}")
    try:
        # cast to int is important as some softwares,
        # like Amberol, are reporting a string
        # If the length of the track is known,
        # it should be provided in the metadata property
        # with the "mpris:length" key.
        # The length must be given in microseconds,
        # and be represented as a signed 64-bit integer
        # see: https://specifications.freedesktop.org/mpris-spec/2.2/Track_List_Interface.html # noqa: E501
        length = int(pmetadata["mpris:length"])
    except KeyError:
        logger.debug(
            "ERROR: The player is not reporting the length of the song to us. "
            "Progress bar won't work."
        )
        length = 100
        pos = 0
    else:
        logger.debug(f"length={length}")
        if length == 0:
            logger.debug(
                "ERROR: Length of song is 0. Progress bar won't work."
            )
            length = 100
            pos = 0
    # Compute song percentage
    perc = (pos / (length / config["length_factor"])) * 100
    perc = min(perc, 100.0)
    metadata = {
        "title": player.get_title() or "",
        "artist": player.get_artist() or "",
        "album": player.get_album() or "",
        "pos": pos,
        "length": length,
        "perc": perc,
    }
    # Check if a rotating text needs to be updated
    elapsed = (time.time() - last_rot_time) * 1000
    rot_needs_update = elapsed > config["text_rot_int"]
    # Check if progress bar needs to be updated
    diff = metadata["perc"] - last_metadata["perc"] if last_metadata else 99999
    pbar_needs_update = diff > config["interval"]
    # Check if lyrics needs an update
    if lyrics:
        # pos is in microseconds
        lyr_pos = float(pos / (10**6))
        lyrics_ix = sum(lyr_pos >= x for x in lyrics["secs"]) - 1
        lyrics_same = lyrics_ix == lyrics["last_ix"]
    else:
        lyrics_same = True
    # Check if song is the same (otherwise text needs to be updated)
    same_song = (
        last_metadata
        and metadata["title"] == last_metadata["title"]
        and metadata["artist"] == last_metadata["artist"]
        and metadata["album"] == last_metadata["album"]
        and metadata["length"] == last_metadata["length"]
        and lyrics_same
    )
    # Check if we need to update
    if same_song and \
            not pbar_needs_update and \
            not rot_needs_update:
        logger.debug(
            "Song is the same and neither the text "
            "nor the progressbar nor the lyrics needs an update. Returning"
        )
        return False
    # Write output
    # - Widget text
    widget_text = gen_widget_text(player)
    # - Tooltip text
    s = [metadata["title"], metadata["artist"], metadata["album"]]
    # -- Lyrics
    curr_line = None
    text_len = None
    if lyrics:
        cand = [len(x) for x in s]
        cand.append(lyrics.get("max_len", 3))
        text_len = max(cand)
        s.append("-" * text_len)
        provider = lyrics.get("provider", "")
        if provider is None:
            provider = "NO PROVIDER"
        s.append("["+provider+"]")

        # DEBUG
        # print(f"lyrics['secs']={lyrics['secs']}")
        # print(f"pos={pos}; length={length}")
        # print(f"lyr_pos={lyr_pos}; lyrics_ix={lyrics_ix}")

        lyrics["last_ix"] = lyrics_ix
        span_before = config["lyrics_span_before"]
        span_after = config["lyrics_span_after"]
        # Make sure we always have the same number of lines
        # also at start and end of the song
        span_before += max(lyrics_ix + span_after - len(lyrics["text"]) + 1, 0)
        span_after += max(span_before - lyrics_ix, 0)
        # Loop over lyrics window
        to_put = range(
            max(lyrics_ix - span_before, 0),
            min(lyrics_ix + span_after, len(lyrics["text"]) - 1) + 1,
        )
        for t in to_put:
            s.append(lyrics["text"][t])
            if t == lyrics_ix:
                curr_line = len(s) - 1
    # -- Tooltip
    center_text(s, text_len)
    if curr_line:
        s[curr_line] = "<b>" + s[curr_line] + "</b>"
    tooltip = f"<span variant='title-caps' font_weight='bold'>{s[0]}</span>"
    tooltip += f"\n{s[1]}"
    # Skip album in tooltip if we don't have it
    s[2] = s[2].strip()
    if s[2]:
        tooltip += f"\n<i>{s[2]}</i>"
    if len(s) >= 4 and s[3].strip():
        tooltip += "\n"
        tooltip += "\n".join(s[3:])
    write_output(widget_text, gen_class(perc), tooltip)
    # It's important to return True to keep the handler going
    last_metadata = metadata
    logger.debug("Returning True")
    return True


def init_player(manager, name):
    logger.debug("I was called")
    logger.debug(f"name.name={name.name}")
    # Register handlers
    player = Playerctl.Player.new_from_name(name)
    player.connect("playback-status", on_playback_status, manager)
    player.connect("metadata", on_metadata, manager)
    manager.manage_player(player)
    # Call update_metadata to update progress bar and send notification
    logger.debug("Calling update_metadata")
    update_metadata(player, player.props.metadata, manager)
    # If we are playing, register refresh interval callback
    register_refresh_interval_callback(player, manager)
    logger.debug("Returning None")


def signal_handler(sig, frame):
    logger.debug("I was called")
    sys.stdout.write("\n")
    sys.stdout.flush()
    # loop.quit()
    sys.exit(0)


def parse_arguments():
    parser = argparse.ArgumentParser()
    # Increase verbosity with every occurrence of -v
    parser.add_argument("command", action="store", default="error")
    parser.add_argument("-v", "--verbose", action="count", default=0)
    return parser.parse_args()


def main():
    arguments = parse_arguments()

    # Initialize logging
    logging.basicConfig(
        stream=sys.stderr,
        level=logging.DEBUG,
        format="[%(name)s] [%(funcName)s] [%(levelname)s] %(message)s",
    )

    # Logging is set by default to WARN and higher.
    # With every occurrence of -v it's lowered by one
    # until it reaches 0
    loglevel = max((3 - arguments.verbose) * 10, 0)
    logger.setLevel(loglevel)
    logging.getLogger("urllib3").setLevel(loglevel)
    logging.getLogger("syncedlyrics").setLevel(loglevel)

    logging.getLogger("syncedlyrics").setLevel(loglevel)

    # Log the sent command line arguments
    logger.info("I was called")
    logger.debug("Arguments received {}".format(vars(arguments)))

    # Read configuration from file
    fp = Path(sys.argv[0]).parent.resolve() / "config.json"
    if not fp.is_file():
        logging.critical(f"ERROR: Configuration file {fp} not found")
        sys.exit(1)
    with open(fp, "r") as fh:
        global config
        config = json.load(fh)
    logger.debug(f"config={config}")
    assert isinstance(config, dict)

    # Remove album art if present
    delete_album_art()

    # Function to generate CSS classes
    global format_class
    interval = config["interval"]
    digits = math.floor(math.log10(interval))
    digits = max(0, math.fabs(digits))
    digits = int(digits)
    format_class = partial(lambda x, y: str(round(x, y)), y=digits)

    manager = Playerctl.PlayerManager()

    requested_player = config["player_name"].split(".")[0]
    logger.debug(
        f"Splitting player_name on `.`. requested_player={requested_player}"
    )

    ini = False
    for player in manager.props.player_names:
        logger.debug(f"Found player '{player.name}'")
        if not player.name.startswith(requested_player):
            logger.debug("This is not the filtered player, skipping it")
            continue
        if arguments.command == "play-pause":
            Playerctl.Player.new(player.name).play_pause()
            sys.exit(0)
        elif arguments.command == "next":
            Playerctl.Player.new(player.name).next()
            sys.exit(0)
        elif arguments.command == "previous":
            Playerctl.Player.new(player.name).previous()
            sys.exit(0)
        elif arguments.command == "monitor":
            logger.debug("Initializing player")
            init_player(manager, player)
            ini = True
        else:
            logger.critical(f"Invalid argument {sys.argv[1]}")
            sys.exit(1)

    if not ini:
        logger.debug("No player found. Printing empty")
        write_output()

    loop = GLib.MainLoop()

    on_player_appeared_inst = partial(
        on_player_appeared, requested_player=requested_player
    )
    # manager.connect("player-appeared", on_player_appeared)
    manager.connect("name-appeared", on_player_appeared_inst)
    manager.connect("player-vanished", on_player_vanished)

    # signal.signal(signal.SIGINT, signal_handler)
    # signal.signal(signal.SIGTERM, signal_handler)
    # signal.signal(signal.SIGPIPE, signal.SIG_DFL)

    loop.run()


if __name__ == "__main__":
    main()
