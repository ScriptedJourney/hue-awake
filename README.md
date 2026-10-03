# Hue Awake

Keep macOS awake so Hue Sync and music light effects can continue while your Mac would otherwise become idle.

Hue Awake runs macOS's built-in `/usr/bin/caffeinate -dimsu` as a per-user LaunchAgent. It starts when you log in, runs in the background, and restarts if the process exits. No additional software or administrator privileges are required.

## Requirements

- macOS with a graphical user login.
- Hue Sync or your preferred lighting application configured separately. Hue Awake does not launch or configure that application.

## Install

Run this in Terminal while logged into the macOS account that should run Hue Awake:

```sh
mkdir -p "$HOME/Library/LaunchAgents"
cat > "$HOME/Library/LaunchAgents/com.unit76.hueawake.plist" <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.unit76.hueawake</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/caffeinate</string>
        <string>-dimsu</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>ProcessType</key>
    <string>Background</string>
    <key>StandardOutPath</key>
    <string>/tmp/hue-awake.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/hue-awake-error.log</string>
</dict>
</plist>
EOF

chmod 644 "$HOME/Library/LaunchAgents/com.unit76.hueawake.plist"
plutil -lint "$HOME/Library/LaunchAgents/com.unit76.hueawake.plist"
```

If validation reports `OK`, load the service:

```sh
launchctl bootstrap "gui/$(id -u)" "$HOME/Library/LaunchAgents/com.unit76.hueawake.plist"
```

It runs immediately and starts again at future logins. If it is already loaded, use the status command below instead of loading it again.

## Manage the service

### Check status

```sh
launchctl print "gui/$(id -u)/com.unit76.hueawake"
pmset -g assertions
```

Look for `state = running` in the LaunchAgent output and assertions owned by its `caffeinate` process in the power-management output. Other applications may also hold sleep-prevention assertions.

### Stop until the next login

```sh
launchctl bootout "gui/$(id -u)/com.unit76.hueawake"
```

The installed LaunchAgent remains enabled for future logins. Stopping it releases Hue Awake's sleep assertions.

### Start after stopping

```sh
launchctl bootstrap "gui/$(id -u)" "$HOME/Library/LaunchAgents/com.unit76.hueawake.plist"
```

### Restart a loaded service

```sh
launchctl kickstart -k "gui/$(id -u)/com.unit76.hueawake"
```

### Uninstall

```sh
launchctl bootout "gui/$(id -u)/com.unit76.hueawake"
rm -f "$HOME/Library/LaunchAgents/com.unit76.hueawake.plist"
```

If the service was already stopped, `bootout` may report that it could not find the service; remove the plist anyway. Other applications and your macOS settings may still prevent sleep.

## What the flags do

| Flag | Effect |
| --- | --- |
| `-d` | Prevent display sleep. |
| `-i` | Prevent idle system sleep. |
| `-m` | Prevent disk idle sleep. |
| `-s` | Prevent system sleep while connected to AC power. |
| `-u` | Declare user activity; without a timeout, this particular assertion lasts five seconds. |

The other requested assertions remain active while `caffeinate` runs. The display stays awake with this configuration, which increases power use. This service is intended for idle sleep prevention; it does not guarantee continued operation when a laptop lid is closed or the Mac is explicitly put to sleep.

For the documentation installed on your Mac, run `man caffeinate`, `man launchctl`, or `man pmset`.

## Troubleshooting

- Run commands as your regular logged-in user, without `sudo`.
- If loading fails, check whether the service is already loaded with `launchctl print`, and validate the plist with `plutil -lint`.
- If lights stop while the service is running, check Hue Sync, audio playback, and the lighting connection separately.
- Service output is directed to `/tmp/hue-awake.log` and `/tmp/hue-awake-error.log`. These files may be empty because `caffeinate` normally runs quietly.

## Original shell-script interface

If you saved the companion script as `hue-awake.sh`, it provides these shortcuts:

```sh
chmod +x hue-awake.sh
./hue-awake.sh install
./hue-awake.sh status
./hue-awake.sh stop
./hue-awake.sh start
./hue-awake.sh uninstall
```

The standalone Terminal instructions above work without the script. The original script's status check matches processes by command line, and its uninstall command can also terminate other processes running `/usr/bin/caffeinate -dimsu`. The LaunchAgent commands above target the Hue Awake service directly.
