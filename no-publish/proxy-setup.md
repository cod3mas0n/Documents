# Setup Proxy for Services and Applications in Linux

## Docker

- [Docker-CLI Proxy Config](https://docs.docker.com/engine/cli/proxy/)
- [Docker Daemon Proxy Config](https://docs.docker.com/engine/daemon/proxy/)

- `mkdir -p /etc/systemd/system/docker.service.d`

- `vi /etc/systemd/system/docker.service.d/http-proxy.conf` and add below to `http-proxy.conf`

    **Http Proxy**

    ```conf
    [Service]
    Environment="HTTPS_PROXY=http://<Proxy-Server-IP>:<Proxy-Port>"
    Environment="HTTP_PROXY=http://<Proxy-Server-IP>:<Proxy-Port>"
    Environment="NO_PROXY=<Private Registry>"
    ```

   **Socks Proxy**

    ```conf
    [Service]
    Environment="HTTPS_PROXY=socks5://<Proxy-Server-IP>:<Proxy-Port>"
    ```

- `systemctl daemon-reload`
- `systemctl restart docker`

## [Podman](https://podman-desktop.io/docs/proxy#using-a-proxy)

Add/Edit File Which Related To Podman Engine

- `rootless`: `$HOME/.config/containers/containers.conf`
- `rootful`: `/etc/containers/containers.conf`

Set the proxy environment variables to pass into the Podman engine:

   ```conf
   [engine]
   env = ["http_proxy=<your.proxy.tld:port>", "https_proxy=<your.proxy.tld:port>"]
   ```

Restart all podman processes.

   ```bash
   pkill podman
   ```

## Git

Set proxies in git global configs or it can be local with no `--global` flag

   ```bash
   git config --global http.proxy "http://<Proxy-Server-IP>:<Proxy-Port>"
   git config --global https.proxy "https://<Proxy-Server-IP>:<Proxy-Port>"
   ```

## Package Managers

### RedHat OS Family

- `dnf` (`Fedora`/`Centos`/`Rocky`/`AlmaLinux`) edit the `/etc/dnf/dnf.conf` file and append:

  Note the `/etc/yum.conf` is a link to `/etc/dnf/dnf.conf`

   ```bash
   lrwxrwxrwx 1 root root 12 Feb  7 16:01 /etc/yum.conf -> dnf/dnf.conf
   ```

   ```conf
   proxy=socks5h://<Proxy-Server-IP>:<Proxy-Port>
   ```

### Debian OS Family

- `apt` (`Debina`/`Ubuntu`) edit/create the `/etc/apt/apt.conf.d/proxy` file and add:

  ```conf
  Acquire::http::Proxy "socks5h://<Proxy-Server-IP>:<Proxy-Port>";
  ```

</br>

## Discord

For Running [Discord](https://discord.com/) through a Proxy Server :

- Edit `/usr/share/applications/discord.desktop` file and add `--proxy-server="socks5://<Proxy-Server-IP>:<Proxy-Port>"` in it where the `Exec` binary is specified :

  ```conf
  [Desktop Entry]
   Name=Discord
   StartupWMClass=discord
   Comment=All-in-one voice and text chat for gamers that's free, secure, and works on both your desktop and phone.
   GenericName=Internet Messenger
   Exec=/usr/bin/Discord --proxy-server="socks5://<Proxy-Server-IP>:<Proxy-Port>"
   Icon=discord
   Type=Application
   Categories=Network;InstantMessaging;
   Path=/usr/bin
   X-Desktop-File-Install-Version=0.26
  ```

  Notice:
  > The Desktop config file may be different in your Linux Distro try: </br>
  > `sudo find / -iname "discord.desktop"` </br>
  > That file will be overwritten when Discord updated. </br>
  > Try to not install Discord via Snap or Flatpak , I recommend To install it via package-manager or its source.

  Its better to add proxy server in its `wrapper` which is an updater:

  ```bash
  ls -l /usr/bin/Discord
  lrwxrwxrwx. 1 root root 27 Sep 18 03:30 /usr/bin/Discord -> ../lib64/discord/wrapper.sh
  ```

  ```shell
  #!/usr/bin/bash

   # Path to discord binary
   DISCORD_BIN=$(dirname $(readlink -f $0))/Discord

   # Run python script to disable check updates
   /usr/lib64/discord/disable-breaking-updates.py

   # Launch discord
   exec "$DISCORD_BIN" "$@" --proxy-server="127.0.0.1:10808"
  ```

## Vscode

vscode desktop config file is `/usr/share/applications/code.desktop` :

```conf
[Desktop Entry]
Name=Visual Studio Code
Comment=Code Editing. Redefined.
GenericName=Text Editor
Exec=/usr/share/code/code --proxy-server=socks5://<Proxy-Server-IP>:<Proxy-Port> --unity-launch %F
Icon=vscode
Type=Application
StartupNotify=false
StartupWMClass=Code
Categories=TextEditor;Development;IDE;
MimeType=text/plain;inode/directory;application/x-code-workspace;
Actions=new-empty-window;
Keywords=vscode;

[Desktop Action new-empty-window]
Name=New Empty Window
Exec=/usr/share/code/code --new-window %F
Icon=vscode
```

Recommendation
> I recommend to set an alias for code , maybe you want `cd` to a directory and do `code .` , so that proxy set up wont be applied </br>
> add below line in `$HOME/.bashrc` or if you use `zsh` add it in `$HOME/.zshrc` </br></br>
> `alias code='code --proxy-server=socks5://<Proxy-Server-IP>:<Proxy-Port>'`
>
> with that , the proxy will apply in both way you run vscode , either `code .` in a directory or running it in your desktop via desktop icon.

## Dropbox

vscode desktop config file is `/usr/share/applications/dropbox.desktop`:

Add `proxy manual socks5 <Proxy-Server-IP> <Proxy-Port>` to `Exec`, can get help with `$(which dropbox) --help`

```conf
[Desktop Entry]
Name=Dropbox
GenericName=File Synchronizer
Comment=Sync your files across computers and to the web
Exec=dropbox start -i proxy manual socks5 127.0.0.1 10808
Terminal=false
Type=Application
Icon=dropbox
Categories=Network;FileTransfer;
Keywords=file;synchronization;sharing;collaboration;cloud;storage;backup;
StartupNotify=false

```

## Google Chrome

If you don't want use Proxy Extensions like [`FoxyProxy`][foxyproxy] or [`Switchy Omega`][switchyomega], Add below cli flag to the `Exec` in `google-chrome` Desktop Entry `/usr/share/applications/google-chrome.desktop` to use proxy server.

```bash
--proxy-server="socks5://127.0.0.1:10808"
```

Help

```bash
/usr/bin/google-chrome-stable --help
```

Notice
> Also I Use an Extension named [Vytal][vytal] which spoofs Location,TimeZone and etc , to disable debug notification in chrome set below cli flag

```bash
--silent-debugger-extension-api
```

## [PProxy](https://pypi.org/project/pproxy/)

Convert a `socks5` to `http` proxy

```bash
pip3 install pproxy
```

```bash
pproxy -l http://0.0.0.0:8080 -r socks5://<Proxy-Host-IP>:<Port> -v > $PWD/pproxy.log &
```

## Links

- [Freedom of Developers](https://github.com/freedomofdevelopers/fod)
- [Proxychains-ng](https://github.com/rofl0r/proxychains-ng)

[vytal]: https://chromewebstore.google.com/detail/vytal-spoof-timezone-geol/ncbknoohfjmcfneopnfkapmkblaenokb
[foxyproxy]: https://chromewebstore.google.com/detail/gcknhkkoolaabfmlnjonogaaifnjlfnp
[switchyomega]: https://chromewebstore.google.com/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif
