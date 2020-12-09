---
title: "Simple Discord Pronouns Bot"
date: 2020-12-09T15:00:00-00:00
tags: [discord]
---

## Introduction

[Discord](https://discord.com) is a popular and quite ubiquitous communications platform, with a powerful role management system.

I've coded a simple bot to allow users to assign roles to themselves for their [preferred personal pronouns](https://www.adl.org/education/resources/tools-and-strategies/lets-get-it-right-using-correct-pronouns-and-names). Pronouns are incredibly important, and it's important to get them right, and give your community members the autonomy to assign their own pronouns.

## Goals 
My bot was design to self-assign the three most common pronoun sets:

- He/Him
- She/Her
- They/Them

There are far more pronoun sets and combinations in use today, but to attempt to keep up with every new pronoun set would be a sisyphean task. Also, introducing the concept of pronouns to those "out of the loop" often has much better reception when using more commonly used and recognized pronouns (such as "he/him/she/her/they/them") rather than less frequently recognized (so-called "[neo-pronouns](https://lgbta.wikia.org/wiki/Neopronouns)").

However, I recognize that not every server is the same and not every member will agree with this line of thinking, so I've open-sourced the bot and made it as simple as I can to add additional pronouns to the list, should your community have a need for that.

> Note: If you do want to have a different set of pronouns included in your community bot, feel free to reach out to me (check my [About]({{< ref "about" >}} "About Me") page if you'd like to communicate with me). I'll be happy to help you customize your own bot and set up hosting/etc if you need assistance.

## Instructions

If you'd like to use my SPB bot as-is, here are the steps:

1. Add roles to your discord server for: "He/Him", "She/Her", "They/Them"
2. Invite the bot using [this link](https://discord.com/oauth2/authorize?client_id=783782475146723339&scope=bot&permissions=268774464).
3. If necessary, grant the following permissions for the bot:
- Read message history (so when it gets restarted, it can find the previous message)
- Add Reactions (it puts 1 emoji per role on the first message)
- Manage Roles (allows the bot to add/remove roles from users)
- Read Messages and Send Messages (obviously :slight_smile: )
> Note: The bot will add a dedicated role (called Simple Pronouns Bot), you can assign the permissions there if the @everyone role doesn't already have those permissions
4. Enter `=setup` in a channel, it will post the react message.
5. After it posts the message, delete your own `=setup` message.

## Code

The source code for this bot is available on my GitHub: [https://github.com/hackerman1337-xo/simple-pronouns-discord-bot](https://github.com/hackerman1337-xo/simple-pronouns-discord-bot). Feel free to submit a PR or poke around the code if you'd like to make some changes!