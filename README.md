### Update MOTD

##### Install prerequisites and set up files

`apt install lsb-release figlet jq update-motd -y`

`rm -r /etc/update-motd.d/`

`mkdir /etc/update-motd.d/`

`touch /etc/update-motd.d/00-header ; touch /etc/update-motd.d/10-sysinfo ; touch /etc/update-motd.d/90-footer`

`chmod +x /etc/update-motd.d/*`

`cd  /etc/update-motd.d`

##### Populate the files

**Header (00-header)**

```
# HEADER FOR MOTD
# THIS CONFIG COMES WITH NO WARRANTY. USER ASSUMES ALL RISKS
# TAILORED FOR DEBIAN BASED LINUX RELEASES

[ -r /etc/lsb-release ] && . /etc/lsb-release
 
if [ -z "$DISTRIB_DESCRIPTION" ] && [ -x /usr/bin/lsb_release ]; then
        # Fall back to using the very slow lsb_release utility
        DISTRIB_DESCRIPTION=$(lsb_release -s -d)
fi

docker=`docker ps --format '{{.Names}}'`
RED='\033[0;31m'
NC='\033[0m' # No Color

figlet $(hostname) #MAKE SURE FIGLET IS INSTALLED (apt install figlet). MORE STYLES AVAILABLE (http://www.figlet.org/)
echo "Docker containers:"
printf "${RED}$docker${NC}\n"

printf "\n"
 
printf "Welcome to %s (%s).\n" "$DISTRIB_DESCRIPTION" "$(uname -r)"
printf "\n"
```

** System Information (10-sysinfo) **

```
# SYSTEM INFORMATION FOR MOTD
# THIS CONFIG COMES WITH NO WARRANTY. USER ASSUMES ALL RISKS
# TAILORED FOR DEBIAN BASED LINUX RELEASES

date=`date`
load=`cat /proc/loadavg | awk '{print $1}'`
root_usage=`df -h / | awk '/\// {print $(NF-1)}'`
memory_usage=`free -m | awk '/Mem:/ { printf("%3.1f%%", $3/$2*100) }'`
swap_usage=`free -m | awk '/Swap:/ { printf("%3.1f%%", $3/$2*100) }'`
users=`users | wc -w`
time=`uptime | grep -ohe 'up .*' | sed 's/,/\ hours/g' | awk '{ printf $2" "$3 }'`
processes=`ps aux | wc -l`
#CHANGE ens18 WITH THE NAME OF YOUR INTERFACE (eth0, enp18s, etc...)
ip=`ifconfig ens18 | grep "inet " | awk '{print $2}'`
#MAKE SURE JQ IS INSTALLD (apt install jq)
myip=`dig +short myip.opendns.com @resolver1.opendns.com`
city=`curl --silent ipinfo.io | jq -r '.city'`
state=`curl --silent ipinfo.io | jq -r '.region'`


RED='\033[0;31m'
NC='\033[0m' # No Color

echo "System information as of: $date"
echo
printf "This system's public IP is: ${RED}$myip${NC}\n"
printf "Located in: ${RED}$city, $state${NC}\n"
echo
printf "System Load:\t%s\tIP Address:\t%s\n" $load $ip
printf "Memory Usage:\t%s\tSystem Uptime:\t%s\n" $memory_usage "$time"
printf "Usage On /:\t%s\tSwap Usage:\t%s\n" $root_usage $swap_usage
printf "Local Users:\t%s\tProcesses:\t%s\n" $users $processes
echo
/usr/lib/update-notifier/update-motd-reboot-required
/usr/lib/update-notifier/apt-check --human-readable
echo
```

** Footer (90-footer) ** *optional*

##### Plex streaming information

Create the following support files in /etc/update-motd/

** requirements.txt **

```
python-dotenv~=0.19.2
```

** .env **

```
# if your configuration for server/port is different than the defaults, edit them accordingly
plex_server=localhost
plex_port=32400
# Replace 'YOUR_TOKEN_HERE' with your X-Plex-Token string obtained from your server.
plex_token=ADDYOURPLEXTOKENHERE
```

** Plex (95-plex) **

```
# PLEX STREAMING INFORMATION
# MUST BE INSTALLED ON THE SAME SERVER THAT IS RUNNING PLEX MEDIA SERVER

#!/usr/bin/env python3
from os import getenv
from re import search
from urllib import request, error
from xml.etree.ElementTree import XML

from dotenv import load_dotenv

# loads the environment variables from the .env file
load_dotenv()

# if your configuration for server/port is different from the defaults, edit them accordingly in the .env file
server = str(getenv('plex_server'))
port = str(getenv('plex_port'))
plex_token = str(getenv('plex_token'))

# ----------------COLORS-----------------
white = '\033[1;37m'
yellow = '\u001b[38;5;214m'
green = '\u001b[32m'
red = '\u001b[31m'
reset = '\u001b[0m'
# ---------------------------------------

# session array
sessions = []


# open the url
def request_open():
    # get session status
    try:
        _fp = request.urlopen('http://' + server + ':' + port + '/status/sessions?X-Plex-Token=' + plex_token)
        _mybytes = _fp.read()
        _fp.close()
        return _fp, _mybytes
    except error.HTTPError as e:
        if e.code == 401:
            exit(str(e) + '\n Please make sure your Plex token is correct and try again')
    except error.URLError as e:
        exit(str(e.reason) + '\n Please make sure the server address is correct and try again')


fp, mybytes = request_open()

# parse text into XML document
xmlstr = mybytes.decode('utf8')
root = XML(xmlstr)


# calculate the bandwidth
def bandwidth():
    session_arr = []
    for item in root.findall('./Video/Session'):
        session_arr.append(item.attrib['bandwidth'])
    res = [float(i) for i in session_arr]
    calc = str(round(sum(res) * 0.001, 1)) + ' Mb/s' \
        if round(sum(res) * 0.001, 1) > 1 else str(round(sum(res), 1)) + ' Kb/s'
    return calc if float(calc[:1]) > 0 else None


# traverse the XML looking for session data
for item in root.findall('./Video'):
    session = {}

    session['Title'] = item.attrib['title']

    # figure out if it's an episode, if so, get more data
    session['Type'] = item.attrib['type']
    if session['Type'] == 'episode':
        session['Series'] = item.attrib['grandparentTitle']
        session['Season'] = item.attrib['parentIndex'].zfill(2)
        session['Episode'] = item.attrib['index'].zfill(2)

    for child in item:
        if child.tag == 'User':
            session[child.tag] = child.attrib['title']
        if child.tag == 'Player':
            session[child.tag] = child.attrib['title']
        if child.tag == 'Media':
            session[child.tag] = child.attrib['videoResolution']

    sessions.append(session)
print(white + 'CURRENT ' + yellow + 'PLEX ' + white + 'SESSIONS' + reset)
sessionNum = len(sessions)
if bandwidth():
    print(white + 'Estimated bandwidth usage: ' + bandwidth() + reset)
if sessionNum == 1:
    print(white + 'Currently, ' + green + str(sessionNum) + white + ' user is in a session:' + reset)
else:
    print(white + 'Currently, ' + red + '0' + white + ' users are in a session.' + reset
          if not sessions else
          white + 'Currently, ' + green + str(sessionNum) + white + ' users are in a session.' + reset)

for session in sessions:
    if session['Type'] == 'episode':
        print(
            white + f'- {session["User"]} is watching "{session["Series"]} - '
                    f'S{session["Season"]}E{session["Episode"]} '
                    f'- {session["Title"]}" on {session["Player"]} in {session["Media"]}'
            + reset if search('[a-zA-Z]', session["Media"]) else white + f'- {session["User"]} is watching '
                                                                         f'"{session["Series"]} - S{session["Season"]} '
                                                                         f'E{session["Episode"]} - '
                                                                         f'{session["Title"]}" on {session["Player"]} '
                                                                         f'in {session["Media"]}p' + reset)
    else:
        print(
            white + f'- {session["User"]} is watching "{session["Title"]}" on {session["Player"]} in {session["Media"]}'
            + reset if search('[a-zA-Z]', session["Media"]) else white + f'- {session["User"]} is watching '
                                                                         f'"{session["Title"]}" on {session["Player"]} '
                                                                         f'in {session["Media"]}p' + reset)
print()
```
