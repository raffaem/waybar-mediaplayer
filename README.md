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

# Requirements

`playerctl` must be installed.

The default configuration use Nerd Fonts, so it requires waybar to use a Nerd Font.

The default configuration use `feh` to open the album art image.

# Install

Run:

```
git clone https://github.com/raffaem/waybar-mediaplayer "$HOME/.config/waybar/waybar-mediaplayer"
```

You need to find the name of your player. To do so, run `playerctl --list-all` while your player is running.

Open `"$HOME/.config/waybar/waybar-mediaplayer/src/config.json` and change `player_name` with the name of your player.

Put the following in `$HOME/.config/waybar/config`:

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
    "exec": "$HOME/.config/waybar/waybar-mediaplayer/src/mediaplayer monitor",
    "return-type": "json",
    "format": "{}",
    "on-click": "$HOME/.config/waybar/waybar-mediaplayer/src/mediaplayer play-pause",
    "on-scroll-up": "$HOME/.config/waybar/waybar-mediaplayer/src/mediaplayer next",
    "on-scroll-down": "$HOME/.config/waybar/waybar-mediaplayer/src/mediaplayer previous",
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
@import "./waybar-mediaplayer/src/mediaplayer-progressbar.css";
```

Restart waybar.

The mediaplayer should work. Click on the progress bar to start/stop playing, and scroll on it to change song.

# Update

```
cd "$HOME/.config/waybar/waybar-mediaplayer"
git pull
```

# Personalization

To disable notifications, put `is_notification=false` in `config.json`.

To change widget's length, set `min-length` and `max-length` in `$HOME/.config/waybar/config`, and set `widget_length` in `$HOME/.config/waybar/waybar-mediaplayer/src/config.json`. These 3 variables MUST be set to the same value.

In order for the album art to automatically update on song change, it's important that the `signal` variable of the `image` module in `$HOME/.config/waybar/config` matches the `image_signal` variable in `$HOME/.config/waybar/waybar-mediaplayer/src/config.json`.

If you change the colors of the bar in `$HOME/.config/waybar/waybar-mediaplayer/src/config.json`, make sure to apply the changes by running `$HOME/.config/waybar/waybar-mediaplayer/src/mediaplayer-progressbar-gencss` and restart waybar.

# Troubleshooting

## Firefox users

Although Firefox reports MPRIS metadata, the metadata it reports is not sufficient, as it doesn't report song's length. Please install the [Plasma Integration](https://addons.mozilla.org/en-US/firefox/addon/plasma-integration) add-on and use `plasma-browser-integration` as `player_name` in `$HOME/.config/waybar/waybar-mediaplayer/src/config.json`.

To have album art, make sure to set `convert_to_jpeg` to `true` in `$HOME/.config/waybar/waybar-mediaplayer/src/config.json` (this option decreases performance, don't use it if not necessary).

## Me progress bar doesn't work

It's likely cause by the player not reporting song length or position back to us. Run `$HOME/.config/waybar/waybar-mediaplayer/src/mediaplayer -vvv` to debug.

## Player reports its name with instance number

If the player reports an instance after its name, please provide only the player name without the instance number. For example, [kew](https://github.com/ravachol/kew) may report itself as `kew123456`, where `123456` is an instance number which will change with different runs of `kew`. In this case, we report only `kew` without the instance number. This software will check whether the reported player name _starts_ with the name you provide to bind the correct player.
