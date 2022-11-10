---
title: Dream Machine - Turn On/Off LED Light Switch  
tags: [Dream Machine]
author: "Yunier"
date: "2020-11-21"
description: "Guide on how to turn off the LED light switch"
---

I want to step away from software for this post to talk about some hardware. Over the years I've owned a few routers, some have been really really [good](https://www.amazon.com/R7000-100PAS-Nighthawk-Parental-Controls-Compatible/dp/B00F0DD0I6/) and some have been really [bad](https://en.wikipedia.org/wiki/Linksys_WRT54G_series). So far the best router I have owned is my current router, the [UniFi Dream Machine](https://www.amazon.com/Ubiquiti-UniFi-Dream-Machine-UDM-US/dp/B081QNJFPV/). While I was mostly happy with my last router, the NighHawk AC1900, it did dropped the WiFi signal a lot, I think that at one point it was dropping the WiFi signal once a week. That prompted me to start looking for a new router. After doing some research, and seeing [Troy Hunt presentation on bad IOT devices](https://youtu.be/FRsRoaubPiY?t=1800) and NetworkChuck's [review](https://www.youtube.com/watch?v=BezoNUflqXo) on the Dream Machine, I was sold on the Dream Machine.

I've owned the Dream Machine for over nine months, and while I love my Dream Machine, there is one tiny little thing that I hate about it. The Blue LED light, the one that appears on top of the cylinder, the blue light means the Dream Machine is on and that there is internet connectivity. It may not look like it from the photos or videos, but that light is super bright, and if you have Dream Machine in a bedroom then it becomes almost impossible to sleep at night. I've been searching for a way to turn this light off from the very first day that I received the Dream Machine. There is no option to turn off the light on the UniFi Dashboard or UniFi app. Luckily, [someone else](https://www.reddit.com/r/Ubiquiti/comments/iklhf3/tool_for_automatic_turning_onoff_led_light_of_udm/) also hate how bright the LED light was and came up with a solution.

That someone else was [**ameyuuno**](https://github.com/ameyuuno), he create a [tool](https://github.com/ameyuuno/docker-unifi-led-light-switch) that allows you to turn the LED light on and off base on a cron job. Which is just the perfect solution for anyone that wants to turn off the LED light, but for me, I was so tired of the LED light that I want it off permanently. I took a look at the project ameyunno created and I notice that all I needed to do was modify [this](https://github.com/ameyuuno/docker-unifi-led-light-switch/blob/master/scripts/unifi-led-switch.sh) shell script and then to run the script from a linux based machine like a Raspberry Pie or from Windows using [WSL](https://docs.microsoft.com/en-us/windows/wsl/install-win10).

```shell
#!/bin/sh

set -e

echo "Run LED light switch script [$(date)]."

led_state=off
username='REPLACE_WITH_UNIFI_USERNAME'
password='REPLACE_WITH_UNIFI_PASSWORD'
url='REPLACE_WITH_UNIFI_URL:8443'


echo "State is: '${led_state}'"
echo "Username is: '${username}'"
echo "Password is: '${password}'"
echo "Url is: '${url}'"

case "${led_state}" in
    on)
        state=true
        ;;
    off)
        state=false
        ;;
    *)
        echo "Unknown state argument value: '${led_state}'"
        exit 1
        ;;
esac

username=${username}
password=${password}
baseUrl=${url}
cookie=/tmp/unifi-cookie

make_request() {
    payload=$1
    url=$2

    curl --tlsv1 --silent --cookie ${cookie} --cookie-jar ${cookie} --insecure --data "${payload}" "${url}"
}

extract_status() {
    cat | jq -r '.meta | if .rc == "ok" then .rc else .rc + "(" + .msg + ")" end | "Status: " + .'
}

print_status_and_set_return_code() {
    status=$(cat)
    echo ${status}

    if [ "${status}" != "Status: ok" ]; then
        echo "Return non-zero code because an error was occurred."
        return 1
    fi

    return 0
}

echo "Login on UniFi."
make_request "{\"username\": \"${username}\", \"password\": \"${password}\"}" "${baseUrl}/api/login" | extract_status | print_status_and_set_return_code

echo "Change state of LED to ${led_state}."
make_request "{\"led_enabled\": \"${state}\"}" "${baseUrl}/api/s/default/set/setting/mgmt/" | extract_status | print_status_and_set_return_code

echo "Logout."
make_request "" "${baseUrl}/api/logout" | extract_status | print_status_and_set_return_code

rm -f ${cookie}

echo "Done."
```

Before running the script, make sure to update the variables **username, password and url** with your username, your password, and the URL you use to connect to the UniFi Dream Machine. Do not drop the port, 8443, you need it. Also, if you have 2FA authentication enabled, you should, you will need to disable it before running the script. You can turn 2FA on again after running the script.

**Credits:** All credits go to [**ameyuuno**](https://github.com/ameyuuno). I recommend checking out his script if you are interested on having the LED light be on and off base on a schedule.
