# waybar-mediaplayer

This is a mediaplayer for [waybar](https://github.com/Alexays/Waybar).

Widget with album art and progress bar:

![progress_bar](./showcase/progress_bar.gif)

Notification with album art, track ticle and track artist

![notification](./showcase/notification.png)

Tooltip with track title, track artist and track album:

![tooltip](./showcase/tooltip.png)

It features:
1. Progress bar
1. Tooltip that displays title, author and album
1. Album cover art (click to zoom)
1. Optional notification on song change
1. Click to play/pause, scroll up/down to scroll up/down on the playlist

# Installation

It requires `playerctl` to be installed. The default configuration use Nerd Fonts, so the default configuration requires waybar to use a Nerd Font.

To install:

```
mkdir -p $HOME/.config/waybar/mediaplayer
cp ./src/* $HOME/.config/waybar/mediaplayer/
```

Open `$HOME/.config/waybar/mediaplayer/config.json` with a text editor and personalize your configuration.

Run `$HOME/.config/waybar/mediaplayer/mediaplayer-progressbar-gencss` to generate the necessary CSS files for the progress bar.

Put the following in `$HOME/.config/waybar/config`, substituting `ncspot` with the name of your player (you can find out the name of your player with `playerctl --list-all`):
```
"modules-left": ["image", "custom/mediaplayer"],
```

```
"image": {
  "path": "/tmp/waybar-mediaplayer-art",
  "size": 32,
  "signal": 2,
  "on-click": "feh --auto-zoom --borderless --title 'feh-float' /tmp/waybar-mediaplayer-art"
},

"custom/mediaplayer": {
  "exec": "$HOME/.config/waybar/mediaplayer/mediaplayer --player ncspot",
  "return-type": "json",
  "format": "{}",
  "on-click": "playerctl --player=ncspot play-pause",
  "on-scroll-up": "playerctl --player=ncspot next",
  "on-scroll-down": "playerctl --player=ncspot previous",
  "min-length": 20,
  "max-length": 20
},
```

Put the following in `$HOME/.config/waybar/style.css`:

```
#custom-mediaplayer
{
  font-size: 16px;
  border-radius: 2%;
}
@import "./mediaplayer/mediaplayer-progressbar.css";
```

To disable notifications, put `is_notification=false` in `config.json`.

To change widget's length, set `min-length` and `max-length` in `$HOME/.config/waybar/config`, and set `widget_length` in `config.json`. These 3 values should all be set to the same value.

