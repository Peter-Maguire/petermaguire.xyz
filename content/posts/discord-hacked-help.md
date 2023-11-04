---
title: "What to do if you fall for the Game Tester Scam"
date: 2023-11-04T16:05:57Z
draft: false
toc: true
images:
tags:
  - malware
  - discord
---

# Introduction  
  
This article should help you if you've fallen from the "test my game" scam on Discord. 
If you don't know what that is, it's outlined in [these](epsilon-stealer-analysis) [posts](bbystealer-analysis).  

# Instructions  

1. **Do not log back into Discord on your PC** as this will give them your password.  Right now, they only have temporary access to your account.  
If you have a second device that is still logged in such as a phone, change your password from that device as soon as you can.  
This will kick them out of your account for now but there is more to do.  
2. **Change any passwords stored in Chrome, Firefox, etc**. Again, do this from a separate device. 
The most important one to change is the password for the email associated with your Discord account as they may use that to change the email address of your account. 
The rest you can do whilst waiting for Discord to respond.
3. **Contact Discord Support**. Discord's [support page](https://dis.gd/support) has a dropdown for if you've been hacked.
In the first dropdown select "Hacked Account", then in the dropdown for "Suspicion of how your account was compromised" select "I downloaded a file".
Be patient in waiting for their response, as repeated messages or new tickets will put you to the back of the queue and the response will take longer.  
4. **Check your PC for suspicious startup files.** Use [Autoruns](https://learn.microsoft.com/en-us/sysinternals/downloads/autoruns) to check for things you don't recognise and disable/remove them.
Many of these malwares use generic sounding names like "Game Launcher" so keep an eye out for things like that.
5. **Uninstall Discord from your PC**. Most malware lives directly in your Discord installation, so before you can safely log back in you need to uninstall and reinstall Discord.
6. **Report the domain that you were sent**. You can report to [phish.report](https://phish.report), [Netcraft](https://report.netcraft.com) or [Vaccinator](https://discord.gg/gmMdSSW8EQ). These will help stop others from getting hacked in the same way.  
7. (OPTIONAL, BUT RECOMMENDED) **Reinstall Windows**. I can't cover all bases here and malware changes all the time, if you want to guarantee that your PC is no longer infected, reinstall Windows. 
  
# FAQ

You may be quite confused about what happened at this point, so this section may clear up some of those questions.   
  
### Why Me?  
Normally they are not targeting specific people for any personal reason, it's usually about the number of badges you have or how "rare" your username is.
Accounts like those are considered "High Quality" and sell for larger amounts. But it's not about making money for everyone, some people just want recognition or attention.  
  
### Why do people do this?  
The people doing this are generally bored kids who think it's funny or clever or gets them some sort of 'clout'. Normally they get bored after they realise they aren't Mr. Robot.  
  
### Why won't Discord do something about this?  
In fairness to Discord, they have started taking this issue more seriously in recent times but they've got a long way to go and there's some seemingly simple changes they could make to make this scam a lot more difficult.
That being said, it's likely that it's just not a large enough problem to really register to them as a priority. They've likely got much larger fires to put out first.  
  
### Why fake games specifically?
Discord is primarily a gaming platform, so many people on the platform are willing to try out new games for people.
A much bigger reason though is that the people who run these scams are not very inventive and just copy off the last person they saw doing it.
This is why the "games" are often named very similarly and use almost identical websites.  
That being said, there are some people who have tried different things - I've seen everything from "Free Nitro Generators" to "Roblox Cheats".

### How can I stop this from happening to me again?  
Once you've seen this scam once, new people trying it are pretty easy to spot as they almost always follow the same pattern.
I've received hundreds of different iterations of this scam, and I have never once had someone ask me to try their game who is actually legit.
Unless you are specifically in those circles, that kind of thing just doesn't really happen.   
  
However hard you try though, it is impossible to completely rule out the risk of falling for it or something more advanced. To reduce the impact, you can follow these steps:  
1. If you are able to, use [Windows Sandbox](https://learn.microsoft.com/en-us/windows/security/application-security/application-isolation/windows-sandbox/windows-sandbox-overview) or a VM to
run new programs or games that you don't trust. It will protect you from most malware and once you're done everything is wiped as soon as you close the window.  
2. Don't use the built-in Firefox or Chrome password managers as they aren't secure, use something like [1Password](https://1password.com) or [BitWarden](https://bitwarden.com) instead.
Not only are these more secure, they are less often targeted by malware developers.
3. Enable 2FA, and keep your backup codes stored somewhere safe. 2FA will make it much harder for hackers to gain persistent access to your account.
Some malware searches for downloaded backup codes and steals them, so make sure you store them somewhere safe and secure, like a USB thumb drive.
4. (ADVANCED) Change your Chrome data directory. This is only really required if you think you're at a high risk of falling for one of these, but it's possible to change where
Chrome stores your browser history and cookies etc.
