---
title: "Epsilon Stealer Malware Analysis"
date: 2022-11-19T17:47:26Z
draft: false
toc: false
images:
tags:  [malware,discord,reverse-engineering,discord-malware]
---

[//]: # (This article is part of my series on Discord malware. Some terminology used in this article is defined in the )

[//]: # ([introduction]&#40;/posts/discord-malware-introduction&#41;. View the entire series [here]&#40;/tags/discord-malware/&#41;)

## Spread

Epsilon generally spreads via the game tester scam. 
The victim is prompted to try out a game from an itch.io link. The download generally contains an archive, usually RAR, which has a password.
The password will be given to the victim via Discord, this is to try and evade automated scanning/execution by anyone other than the victim.

{{< figure src="/img/epsilon/spread-example.png" title="An example DM a victim would receive" >}}

## Infection

Once the victim has downloaded the archive and extracted it, they will be faced with a Unity game. 
Other Discord malware attempts to masquerade as a Unity game by including files from unrelated legitimate games inside the archive.
Epsilon takes this a step further by wrapping the malicious code inside a Unity game shell which drops the second stage payload.
Scanning the executable with VirusTotal will also turn up clean and shares a hash with many legitimate Unity game launchers. 

{{< figure src="/img/epsilon/archive-context.png" title="The archive contents" >}}

When the victim launches the executable, a Unity game will launch and ask for a key. 
The purpose of this key is unknown, but I would guess this is designed to be a confirmation that a user has actually executed the file. 

{{< figure src="/img/epsilon/fake-out.png" title="Pretending to have run the game without asking for the key" >}}
{{< figure src="/img/epsilon/enter-the-key.png" title="An example of the key entry screen" >}}

With a correct key entry, a very basic game will launch. This game is usually ripped directly from elsewhere, 
and only serves to convince the victim that the attacker's request was legitimate.

## Payload

Upon launching the game, the Unity shell drops an application called [GameManager](https://www.virustotal.com/gui/file/b3b1578ad65142d2c12d2db14e1fcfc7d76c0cbd6bb97655a6c6b70ff9e959c7) into `%LocalAppData%\Temp\[random]\[random]` and then executes it.

GameManager is an Electron based application which executes the initial malicious payload and also displays the "game" that is shown to the user.

{{< figure src="/img/epsilon/game-manager.png" title="The GameManager program" >}}

### Information Gathering

#### IP/Location Data
The malware retrieves the users external IP address and geolocation data from [ipinfo.io](https://ipinfo.io).
 
#### Chromium Data
Epsilon looks for Chromium data in the following locations, corresponding to Discord, Chrome, Operate, Brave, Yandex, Edge and Vivaldi install locations.
Paths containing {number} are iterated over 5 times, substituting {number} with a number 1-5.
```js
const appData = process.env.APPDATA; // Default: C:\Users\[username]\AppData\Roaming
const localAppData = process.env.LOCALAPPDATA; // Default: C:\Users\[username]\AppData\Local
let chromiumPaths = [
    appData + "\\discord\\",
    appData + "\\discordcanary\\",
    appData + "\\discordptb\\",
    appData + "\\discorddevelopment\\",
    appData + "\\lightcord\\",
    localAppData + "\\Google\\Chrome\\User Data\\Default\\",
    localAppData + "\\Google\\Chrome\\User Data\\Guest Profile\\",
    localAppData + "\\Google\\Chrome\\User Data\\Default\\Network\\",
    localAppData + "\\Google\\Chrome\\User Data\\Guest Profile\\Network\\",
    appData + "\\Opera Software\\Opera Stable\\",
    appData + "\\Opera Software\\Opera GX Stable\\",
    appData + "\\Opera Software\\Opera Stable\\Network\\",
    appData + "\\Opera Software\\Opera GX Stable\\Network\\",
    localAppData + "\\BraveSoftware\\Brave-Browser\\User Data\\Default\\",
    localAppData + "\\BraveSoftware\\Brave-Browser\\User Data\\Guest Profile\\",
    localAppData + "\\BraveSoftware\\Brave-Browser\\User Data\\Default\\Network\\",
    localAppData + "\\BraveSoftware\\Brave-Browser\\User Data\\Guest Profile\\Network\\",
    localAppData + "\\Yandex\\YandexBrowser\\User Data\\Guest Profile\\",
    localAppData + "\\Yandex\\YandexBrowser\\User Data\\Guest Profile\\Network\\",
    localAppData + "\\Microsoft\\Edge\\User Data\\Default\\",
    localAppData + "\\Microsoft\\Edge\\User Data\\Guest Profile\\",
    localAppData + "\\Microsoft\\Edge\\User Data\\Default\\Network\\",
    localAppData + "\\Microsoft\\Edge\\User Data\\Guest Profile\\Network\\",
    localAppData + "\\Vivaldi\\User Data\\Default\\",
    localAppData + "\\Vivaldi\\User Data\\Guest Profile\\",
    localAppData + "\\Vivaldi\\User Data\\Default\\Network\\",
    localAppData + "\\Vivaldi\\User Data\\Guest Profile\\Network\\",
    localAppData + "\\Google\\Chrome\\User Data\\Profile {number}\\",
    localAppData + "\\BraveSoftware\\Brave-Browser\\User Data\\Profile {number}\\",
    localAppData + "\\Yandex\\YandexBrowser\\User Data\\Profile {number}\\",
    localAppData + "\\Microsoft\\Edge\\User Data\\Profile {number}\\Network\\",
    localAppData + "\\Vivaldi\\User Data\\Profile {number}\\Network\\",
    localAppData + "\\Google\\Chrome\\User Data\\Profile {number}\\Network\\",
    localAppData + "\\BraveSoftware\\Brave-Browser\\User Data\\Profile {number}\\Network\\",
    localAppData + "\\Yandex\\YandexBrowser\\User Data\\Profile {number}\\Network\\",
    localAppData + "\\Microsoft\\Edge\\User Data\\Profile {number}\\Network\\",
    localAppData + "\\Vivaldi\\User Data\\Profile {number}\\Network\\"
];
```
The malware iterates over the list multiple times to extract the following information, skipping over the paths which contain the phrase `cord`:
- Cookies
- Saved Passwords
- Autofill data
- Saved Cards

The results of these extractions are stored a temp directory created with `os.createtempdir()`.

The list is iterated a final time to retrieve Discord login data specifically.
- For paths including the phrase `discord`:
  - The `Local State` file is read to extract the login decryption key.
  - The login decryption key is decrypted using [DPAPI unprotectData](https://github.com/daguej/node-dpapi)
  - The folder `Local Storage\leveldb` is iterated over, reading each file which ends in `.log` or `.ldb`
  - Each file is opened as text, and it attempts to match the regex for an encrypted Discord token `dQw4w9WgXcQ:[^.*\['(.*)'\].*$][^\"]*`
  - For each match, the encrypted element of the token is extracted and decrypted using the key retrieved earlier
  - The decrypted token is pushed to the results array
- For all other paths:
  - The `.log` and `.ldb` files are retrieved from `Local Storage\leveldb` as before
  - The files are opened as text, and the regex for an unencrypted Discord token is used `[\w-]{24}.[\w-]{6}.[\w-]{25,110}`
  - The token is pushed to the results array

#### Firefox Data
Similar to Chromium data extraction, the following data is extracted from `%APPDATA%\Mozilla\Firefox\Profiles`:
- Saved Passwords
- Cookies
- Autofill data

The results of these extractions are stored in the same temp directory as the Chromium data, under a sub-folder called `epsilon-[username]`.
Interestingly, no attempt is made to extract Discord tokens from Firefox.

#### Discord
The tokens retrieved from Chromium data extraction are used to extract the following info:
- Account badges
- Nitro subscription (if any)
- Email address
- Payment sources
- Phone Number
- HQ Friends list

Epsilon downloads and injects a payload into the Discord client which is used for further data extraction.
The following paths are checked for a Discord installation:

```js
discordPaths = [
    localAppData + "\\discord\\",
    localAppData + "\\discordcanary\\",
    localAppData + "\\discordptb\\",
    localAppData + "\\discorddevelopment\\",
    localAppData + "\\lightcord\\"
];
```

If a Discord installation is found, it then attempts to [glob](https://en.wikipedia.org/wiki/Glob_(programming)) match a file with the path
`\app-*\modules\discord_desktop_core-*\discord_desktop_core\index.js`. This path corresponds to a plaintext JS file which is executed before the Discord client is initialised,
and is usually contains a single line. Epsilon downloads a payload from a paste service and overwrites `index.js` with the contents of the paste.
A directory is also made called `Epsilon-Stealer` as a crude way of signalling that the injection is successful. This is a common practise in stealers,
and in most the presence of this directory is checked for in order to decide whether to continue with the infection, however it doesn't appear to be used here.

The presence of BetterDiscord is also checked for and, if found, BetterDiscord's 'webhook protection' feature is 
disabled by overwriting the string `api/webhooks` with `vulsa1111` in `\BetterDiscord\data\betterdiscord.asar` 

The following paths are also checked for a file which contains the phrase `discord_backup_codes`, presumably hoping to find a file containing 2FA backup codes.

```js
let userProfile = process.env.USERPROFILE;
userPaths = [
    userProfile + "\\Desktop\\",
    userProfile + "\\Downloads\\",
    userProfile + "\\Documents\\"
];
```

#### Cryptocurrency
Each non-discord chromium path is checked for the following extensions, and the data is extracted from each:
```js
cryptoExtensions["Local Extension Settings\\nkbihfbeogaeaoehlefnkodbefgpgknn"] = "Metamask";
cryptoExtensions["Local Extension Settings\\bfnaelmomeimhlpmgjnjophhpkkoljpa"] = "Phantom";
cryptoExtensions["Local Extension Settings\\fhbohimaelbohpjbbldcngcnapndodjp"] = "Binance";
cryptoExtensions["Local Extension Settings\\hnfanknocfeofbddgcijnmhnfnkdnaad"] = "Coinbase";
cryptoExtensions["Local Extension Settings\\aeachknmefphepccionboohckonoeemg"] = "Coin98";
cryptoExtensions["Local Extension Settings\\hifafgmccdpekplomjjkcfgodnhcellj"] = "Crypto_com";
```

`%APPDATA%\Exodus\exodus.wallet` is also checked for and extracted.

The data for Crypto wallets is stored in `[temp dir]\epsilon-[username]\Wallets`.

#### Telegram

Epsilon checks for the existence of `%APPDATA%\Telegram Desktop\tdata`, lists all sub-dirs and files and:  
- If a sub-directory name is exactly 16 letters long, the contents of the directory is extracted
- If a filename ends in s and is exactly 17 letters long, the file is extracted

The malware also attempts to extract the `usertag`, `settingss` and `key_datas` files from the `tdata` directory,
due to poor programming it copies each of these files once for every file inside the `tdata` directory.

The extracted data is copied to `[temp dir]\epsilon-[username]\Messengers\Telegram`

#### MobaXterm

Interestingly, Epsilon targets [MobaXTerm](https://mobaxterm.mobatek.net/), extracting `%APPDATA%\MobaXterm\MobaXterm.ini` 
and storing it in `[temp dir]\epsilon-[username]\Ftps`. This is the only SSH client that is targeted.

#### FileZilla

[FileZilla](https://filezilla-project.org/) is also targeted, extracting `%APPDATA%\FileZilla\filezilla.xml`
and storing it in `[temp dir]\epsilon-[username]\Ftps`

## Exfiltration

Data is sent back to the attacker via a Discord webhook. This webhook is retrieved from a paste service,
although in one of the samples analysed the paste was retrieved but its result was discarded in favour of a hardcoded webhook. 

Once all the above data has been extracted, the entire `epsilon-[username]` folder is compressed into a zip file.
The zip file is then uploaded to the webhook, along with two embeds.
One which summarises the number of each type of data that has been extracted, along with the computer name, username and IP address.
The other which shows the Discord specific information. 

{{< figure src="/img/epsilon/epsilon-icons.png" title="The emojis used in the embeds" >}}

The Discord specific embed uses emojis which are prefixed with EM_, and were created between 2022-02-02 and 2022-07-05,
with one emoji being created a year earlier on 2021-02-26.

The summary embed refers to messengers as "Applications de communications", suggesting that the creator is French-speaking. 

## Evasion

Epsilon does not appear to try and evade detection or analysis in any way.
The name "Epsilon" is written in several places inside the malware, with each embed and file being marked with the name. 

## Persistence

After the initial data extraction is complete, a second version of the malware titled [WindowsBootManager.exe](https://www.virustotal.com/gui/file/3a3e3f8bb3ea348375c6afad7f6f28a90040c178ac29b378b60e6798cbf8c3ac) is downloaded from 
the Discord CDN and moved into `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup`. This file appears to be functionally 
identical to the original malware, but without the game. In the latest sample, this file was uploaded on 2022-11-11.  

Interestingly, if the package `electron-squirrel-startup` is detected, the app quits immediately before running its payload. 