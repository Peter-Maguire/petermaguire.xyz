---
title: "BbyStealer Analysis"
date: 2022-12-07T22:10:14Z
draft: false
toc: false
images:
tags:
  - malware
---


BbyStealer is one of the most prevalent information stealing malware that exists on Discord today.
It has been around since approximately December 2021 and has evolved several times.
This article focuses on the version from December 2022, but may make references to older versions where appropriate.  
  
  
## Spread  
BbyStealer is sold as Malware as a Service, so the spread varies depending on the individual malicious actor (called "operators" for the rest of this article).
Generally, this malware comes through as the "game tester" scam, but has been seen before being advertised as a "Nitro generator" or "Discord Beta".  

{{< figure src="/img/bby/spread-beta.png" title="The 'Beta' and 'Game Tester' scams, commonly used by BbyStealer." >}}  
  
The malware is often distributed as a RAR file with a password which is usually supplied to the user as a "beta key".
This is to evade automatic detection or analysis.  
  
Once the RAR file is opened, a single file often called `Setup.exe` is inside. This file usually has the same icon each time.  

{{< figure src="/img/bby/icons.png" title="Some of the commonly used icons and names." >}}

## Infection

Once launched, BbyStealer wastes no time with displaying anything to the user and simply immediately kills any open Browser or Discord windows.  
Browser cookies, autofill and saved passwords are then stolen and a payload is injected into the Discord client.
This payload is obfuscated, and contains two additional obfuscated payloads within it.

There does not appear to be any persistence other than through the payload inserted into the Discord program.  
  
## Information Gathering  
  
### Discord
The malware extracts information from Discord by hooking into the electron [onCompleted](https://www.electronjs.org/docs/latest/api/web-request#webrequestoncompletedfilter-listener) event.
This event allows a script to execute on completion of a request to a specified array of possible URLs. The URLs supplied are as follows:  
```json
[
    "https://discord.com/api/v*/users/@me",
    "https://discordapp.com/api/v*/users/@me",
    "https://*.discord.com/api/v*/users/@me",
    "https://discord.com/api/v*/users/@me/mfa/totp/enable",
    "https://discordapp.com/api/v*/users/@me/mfa/totp/enable",
    "https://*.discord.com/api/v*/users/@me/mfa/totp/enable",
    "https://discordapp.com/api/v*/auth/login",
    "https://discord.com/api/v*/auth/login",
    "https://*.discord.com/api/v*/auth/login",
    "https://api.stripe.com/v*/tokens"
]
```  
Note that this method of extracting information completely bypasses Discord's token encryption, because the token must be in its insecure form in order to be sent as part of the API request.  
The data either sent to or returned from the above URLs are transformed into data which is exfiltrated via the API.  
  
In addition to the token, each event sends the following information:
  
### login  
This event is sent for all URLs ending with `login`, and extracts the `password` field from the request.  
  
### enabled2FA  
Sent when the `totp/enable` request is made, includes the TOTP secret and account password.  
  
### changedEmail  
Sent when a PATCH request is sent to `users/@me` which includes an `email` and `password` field. Sends the `email` and `password` fields from the request.  
  
### changedPassword  
Sent when a PATCH request is sent to `users/@me` which includes a `new_password` and `password` field. Sends the `new_password` and `password` fields from the request.  
  
### cardAdded  
Sent for the lone stripe URL in the array, includes all card info sent to stripe, including card number, cvc, name, etc.  
  
Any accounts logged in via the account switcher also have their token stolen.   
  
### QR Login  
The QR login is tampered with, overriding the normal QR code display and replacing it with a QR code supplied by a websocket connection to `wearenotbbystealer.nl`.  
Once connected, the websocket sends a 'welcome' event to the client:  
```json
{
    "action": "welcome"
}
```  
In response, the websocket is expecting a 'key' event, containing the operators key:  
```json
{
    "action": "key",
    "key": "[the key]"
}
```
If all goes well, the websocket will periodically update the QR code, as if it were legitimate:  
```json
{
    "action": "qrcode",
    "qrcode": "[a QR code URI]"
}
```
This technique is used to gain access to the users account even if they choose to login via QR, which with other malware may be a more secure option.  
Thankfully, they added a `title` element to the QR code when hovered over to tell you what's going on:  
```html
    <div class="qrCode-2R7t9S" title="Oh nice! bby steal your account"/>
```
  
## API  
Unlike many simpler information stealers, BbyStealer does not use a webhook directly. Instead, there are multiple C2 (Command and Control)
servers which serve as a relay for information from the malware creator to the individual customers Discord server.
This also allows the malware creator to take a copy of information that is stolen for themselves, a technique known as "dualhooking".

As of December 2022, the current C2 uses the following domains: 
- `t4ck0wsvvpbmktxzluyee11uce27kbct.nl`  
- `kqnfkpoccicxiudstqonfotuwsrhuxkwhqjjfsbjhonoubrccy.nl`  
- `wearenotbbystealer.nl`  

The following domains have been used in the past, but are no longer active or are dormant:
- `mdvksublbpczqluqvvbytfprxdwakuke.nl`
- `indianboatparty.com`
- `blackboat.party`
- `superfuniestindianparty.rip`  
- `superfurrycdn.nl`
- `bbystealer.wtf`
- `bbystealer.in` 
- `bbystealer.rip`
- `bbynetwork.nl`
- `bby.gg` / `socket.bby.gg`
- `bby.sex`
- `bby.rip`
- `bby.solutions`
- `weloveponysuwu.org`
  
BbyStealer's authentication revolves around "keys". These are randomly generated strings which are assigned to an operator and map in the backend to a Discord webhook.
Previously, operators have been seen to have 'vanity' keys, or keys prefixed with `FREE-` denoting a limited feature version of the malware.
  
### Endpoints  
In previous versions of the malware distinct endpoints have been used, each prefixed with the operators key.
In the current version, a single POST request to `/:key` is used with the `type` field (See [the Discord section](#information-gathering) for type names) dictating what type of data is being sent.  
The endpoint expects data in the following format:  
```json5
{
    "data": {
      // Depends on the `type` field
    },
    "billing": {},
    "friends": [],
    "token": "xyz",
    "type": "login"
}
```  
  
## Evasion
As is common with these Discord-focussed malware strains, there is very little in the way of evasion.
The malware code is obfuscated using 2 different obfuscators, although the obfuscation is easily reversed.
The name of the malware is plastered all over the code, and even added into the Discord client.