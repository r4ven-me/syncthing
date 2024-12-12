# Syncthing - file synchronization server in docker

- [Syncthing - file synchronization server in docker](#syncthing---file-synchronization-server-in-docker)
  - [Preparation](#preparation)
  - [Running Syncthing Server with Docker Compose](#running-syncthing-server-with-docker-compose)
    - [Setting up autostart with systemd](#setting-up-autostart-with-systemd)
  - [Access to Syncthing server web interface](#access-to-syncthing-server-web-interface)
  - [Installing clients](#installing-clients)
    - [Linux Mint](#linux-mint)
    - [Windows](#windows)
    - [Android](#android)
  - [Sources used](#sources-used)

## Preparation

To set up our Syncthing service, we will need a Linux server with **docker engine** installed and running . If you don't have your own server yet, you might find the following articles useful:

- [Installing Debian 12 Server](https://r4ven.me/it-razdel/instrukcii/ustanovka-servera-debian-12-v-virtualbox/)
- [Initial setup of Linux server using Debian as an example](https://r4ven.me/it-razdel/instrukcii/nachalnaya-nastrojka-linux-servera-na-primere-debian/)
- [Installing Docker engine on a Linux server running Debian](https://r4ven.me/it-razdel/instrukcii/ustanovka-docker-engine-na-linux-server-pod-upravleniem-debian/)

If the Linux server is ready, we connect to it, for example, via [SSH](https://r4ven.me/it-razdel/zametki/ssh-bezopasnoe-podklyuchenie-k-udalyonnym-hostam-vvedenie/) 🔐 and first of all we install the version control system  `git`:

```bash
sudo apt update && sudo apt install -y git
```

[![](https://r4ven.me/wp-content/uploads/2024/12/image-8.png)](https://r4ven.me/wp-content/uploads/2024/12/image-8.png)

Next, download the project files from my [GitHub](https://github.com/r4ven-me/syncthing) to the directory `/opt`:

```bash
sudo git clone https://github.com/r4ven-me/syncthing /opt/syncthing
```

[![](https://r4ven.me/wp-content/uploads/2024/12/image-10.png)](https://r4ven.me/wp-content/uploads/2024/12/image-10.png)

> Please note that the service `syncthing`described in the file `docker-compose.yml`is intentionally limited in hardware resources for use `cpus: '0.70'`and `memory: 512M`, i.e. the maximum allowed CPU usage is 70% of one core and 512 MB of RAM. If necessary, adjust these parameters according to your needs. Read more about service resource limits when using docker-compose [here](https://docs.docker.com/compose/compose-file/deploy/) .

Now let's create a service group and user `syncthing`to run the service:

```bash
sudo addgroup --system --gid 3113 syncthing

sudo adduser --system --gecos "Syncthing file synchronization system" \
    --disabled-password --uid 3113 --ingroup syncthing  \
    --shell /sbin/nologin --home /opt/syncthing/data syncthing
```

[![](https://r4ven.me/wp-content/uploads/2024/12/image-11.png)](https://r4ven.me/wp-content/uploads/2024/12/image-11.png)

Don't forget to set limited rights for the directory☝️ where our files will be stored:

```bash
sudo chmod 700 /opt/syncthing/data
```

> For more information about file rights, see [a separate article](https://r4ven.me/it-razdel/zametki/komandnaya-stroka-linux-prava-na-fajly-komandy-id-chmod-chown/) .

We check the presence of all necessary files and folders:

```bash
sudo ls -l /opt/syncthing
```

[![](https://r4ven.me/wp-content/uploads/2024/12/image-12.png)](https://r4ven.me/wp-content/uploads/2024/12/image-12.png)

Great, let's move on 🚶.

## Running Syncthing Server with Docker Compose

We check the syntax of the compose file and launch our new service:

```bash
sudo docker compose -f /opt/syncthing/docker-compose.yml config --quiet

sudo docker compose -f /opt/syncthing/docker-compose.yml up -d

sudo docker ps
```

[![](https://r4ven.me/wp-content/uploads/2024/12/image-14.png)](https://r4ven.me/wp-content/uploads/2024/12/image-14.png)

Yeah, great. Let's look at the container output:

```bash
sudo docker logs -f syncthing
```

It will be something like this:

[![](https://r4ven.me/wp-content/uploads/2024/12/image-13.png)](https://r4ven.me/wp-content/uploads/2024/12/image-13.png)

Everything is fine👌.

### Setting up autostart with systemd

At the beginning of the article, we created a service user `syncthing`. We use it to safely launch our new service using `sudo`💪.

We copy the file with the description of limited **sudoers** permissions for the user `syncthing`, having previously checked it for syntax errors:

```bash
sudo visudo --check --file=/opt/syncthing/syncthing_sudoers

sudo cp /opt/syncthing/syncthing_sudoers /etc/sudoers.d/
```

[![](https://r4ven.me/wp-content/uploads/2024/12/image-15.png)](https://r4ven.me/wp-content/uploads/2024/12/image-15.png)

The file `syncthing_sudoers`describes the privileges📝 that allow the user `syncthing`to start and stop the services described in our `docker-compose.yml`, on behalf of the privileged **docker** group using `sudo`. These are the commands that will be used in the **systemd** service unit .

> Recently I have been examining the intricacies of the privilege escalation mechanism in Linux. For a better understanding I recommend reading the article: [Linux command line, privilege escalation: su, sudo commands](https://r4ven.me/it-razdel/zametki/komandnaya-stroka-linux-povyshenie-privilegij-komandy-su-sudo/) 😌.

Now we also check, then copy the systemd service file and reread the configuration:

```bash
sudo systemd-analyze verify /opt/syncthing/syncthing.service

sudo cp /opt/syncthing/syncthing.service /etc/systemd/system/

sudo systemctl daemon-reload
```

[![](https://r4ven.me/wp-content/uploads/2024/12/image-16.png)](https://r4ven.me/wp-content/uploads/2024/12/image-16.png)

The file `syncthing.service`describes the conditions for launching the container `syncthing`, such as: checking the running docker daemon, restarting the service when it crashes (with an interval of 5 seconds), defining the user/group on behalf of which the service should be launched, and the actual commands for starting/stopping this service.

We stop the previously launched container manually and start it using systemd + activate the service's autostart at system startup:

```bash
sudo docker compose -f /opt/syncthing/docker-compose.yml down

sudo systemctl enable --now syncthing
```

[![](https://r4ven.me/wp-content/uploads/2024/12/image-17.png)](https://r4ven.me/wp-content/uploads/2024/12/image-17.png)

Checking the status:

```bash
sudo systemctl status syncthing
```

It should be like this:

[![](https://r4ven.me/wp-content/uploads/2024/12/image-18.png)](https://r4ven.me/wp-content/uploads/2024/12/image-18.png)

The server is up and running and ready to go😎!

## Access to Syncthing server web interface

In my example, Syncthing, for security purposes, was launched on a local interface with the address `127.0.0.1`. If you are launching the service on a remote machine, then to access the Syncthing web interface, forward the required port (8384) on the client machine using the following SSH command:

```bash
ssh -N -f -L 7777:127.0.0.1:8384 user@example.com
```

Where (in order):

- `-N`– use only port forwarding;
- `-f`– go to background mode immediately before executing the command (forwarding);
- `-L` – local port forwarding key;
- `7777` – the port that the client machine will listen to and redirect it to `8384`the server port;
- `127.0.0.1` – the address of the server on which it listens to the Syncthing web interface port;
- `8384` – accordingly the port to which we are redirecting;
- `user@example.com` – user and SSH server address.

In my example the command is:

[![](https://r4ven.me/wp-content/uploads/2024/12/image-50.png)](https://r4ven.me/wp-content/uploads/2024/12/image-50.png)

Now open a web browser and enter:

```bash
http://127.0.0.1:7777
```

## Installing clients

Syncthing clients exist for all popular platforms💻📱. You can find the full list [here](https://syncthing.net/downloads/) and [here](https://docs.syncthing.net/users/contrib.html) 😇.

Under the hood, clients start the Syncthing backend (similar to the server) and the graphical frontend + system tray. By default, the service also listens for web connections🌐, so you can always connect to the service via a browser (if desired, this function can be disabled). The listening port depends on the client. Most often, it is **8080** or **8384** 🤷‍♂️.

Next, I will show the process of connecting clients using **Linux Mint 22** , **Windows 10** and **Android 13** as examples 🫡.

**Recommendation** : During the initial setup of clients, use directories without important files to minimize the risks in case of incorrect actions.

### Linux Mint

Open the terminal, update the package cache and install the Syncthing graphical client with system tray support from the standard [repositories](https://r4ven.me/it-razdel/slovarik/repozitorij-programmnogo-obespecheniya/) :

```bash
sudo apt update && sudo apt install -y syncthing-gtk
```

[![](https://r4ven.me/wp-content/uploads/2024/12/image-7.png)](https://r4ven.me/wp-content/uploads/2024/12/image-7.png)

> There is also a **qt** version.

After installation, launch the program from the main menu:

[![](https://r4ven.me/wp-content/uploads/2024/12/image-28.png)](https://r4ven.me/wp-content/uploads/2024/12/image-28.png)


### Windows

There is an official Syncthing build for Windows, but it does not include system tray support, which is inconvenient🥲. Therefore, the developers recommend using the third-party **SyncTrayzor** application on their website . You can download it from the GitHub [releases page⬇️](https://github.com/canton7/SyncTrayzor/releases) .

The installation is typical for Win systems “next-next-next”:

[![](https://r4ven.me/wp-content/uploads/2024/12/image-40.png)](https://r4ven.me/wp-content/uploads/2024/12/image-40.png)

After installation, open the application (you will see an icon in the system tray🧐). We see an interface similar to the server.

### Android

For Android OS, there is an application of the same name Syncthing, but its development stopped this year😔 and the developers recommend using its extended fork: **Syncthing-Fork** . These applications can be easily found in Google play and F-droid. Or on the GitHub [releases page](https://github.com/Catfriend1/syncthing-android/releases).

## Sources used

- [Official website of the Syncthing project](https://syncthing.net/)
- [Syncthing source code on GitHub](https://github.com/syncthing/syncthing)
- [Syncthing repository on hub.docker.com](https://hub.docker.com/r/syncthing/syncthing)
- [Official documentation (EN)](https://docs.syncthing.net/users/index.html)