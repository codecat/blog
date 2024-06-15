+++
title = "What I learned from The Serious Ranker"
date = 2015-10-25
path = "2015/10/what-i-learned-from-the-serious-ranker"

[extra]
og_description = "Designing a third-party leaderboard for an FPS game, and its progression towards its end-of-life."
+++

This is a post I've wanted to do for a while. It's about a leaderboard design and its progression towards its end-of-life. I'm posting this in the hope that it will be useful to people, you could learn something from this!

<!-- more -->

Take note that all of the below info is just **my experience**, it's about what happened, not what **could happen**. I realize a lot of things here might not be applicable to different games.As you may or may not know, I've worked on a project called "The Serious Ranker" for a while, which was a website that ranked player's kills and skills in Serious Sam HD: The Second Encounter versus servers, as a way to get an "official" leaderboard for versus. It was ranking Croteam's official servers (which has since been taken offline because of a severe lack of activity in the game) and a couple of servers that were online at the time for SeriousZone.

There's a couple of things that this website kept track of:

* Kills
* Deaths
* Dueling ELO
* Weapon usage
* Map popularity
* Clans and clan membership
* Country statistics
* Player reputation

By default, the site would order the top 25 players by kills. This means that a player with 100 kills in total would be in a better position than a player with 50 kills in total for example. This worked for a while, until it became clear that players were cheating the system by killing their friends over and over again.

People complained.

Hardcore players complained that the rankings were unfair and that it did not represent true skill. Okay, fair enough. We introduced default ordering by KDR. This brought up an issue where people with only a total of 5 kills and a total of 0 deaths would be #1 by default, since their KDR would be 5.00, so we put in a minimum amount of kills for players to show up on the top 25.

People complained.

Why? Surely, a KDR value must be a fair representation of their skill, right? Well yes, in a way. But people complained that a KDR value would be "permanent", meaning that if their skill changes over the course of their ranker presence, their KDR value wouldn't be a fair representation of skill anymore. Needless to say, people also didn't like their amount of deaths showing up next to their amount of kills for everyone to see. So, we eventually came to the conclusion that we had to implement ELO scores.

ELO score is a type of score that originated from chess. I won't go into how it's calculated in this post, but the general idea is that when a player wins a fight with a player that has a higher score, they will get a higher increase in score than when winning a fight against someone with a lower score. This score is then added to the winner's score and deducted from the loser's score. For example, if a player with score of 1000 wins against a player with a score of 1250, the winning player might get an addition in score of 100, while if the player of 1000 wins against a player of 750, the winning player might only get 10.

Since ELO scores only work with matches that involve 2 players fighting each other, we had it implemented for duel matches only. (It's possible to [do it with more players](https://xonotic.org/posts/2012/much-ado-about-elo/) (see "team matches"), but we didn't implement that at the time.) If there were 2 players fighting each other in a server without any other players playing, there would be an ELO increase/decrease to both those players.

...people complained.

We had ELO implemented, but a few people still didn't like it. At that point, a small group of people seemed to be more in a phase where they just wanted to have fun, but were getting annoyed by the players boasting about their ranker top 25 presence. Nothing you can really do about that. The other players liked the ELO score, and some KDR-boasting sunglasses-wearing people were quieted down since they sucked at duels, making the rankings a lot more accurate.

As of later, we implemented clans and clan membership. Clans would be detected from the player's names using a regex. Ofcourse, it didn't catch anything, but it detected the most commonly used formats in Serious Sam HD: \[Clan\], \[~Clan~\], &gt;&gt;Clan&lt;&lt;, etc. Later on we changed this system to include a clan-admin system, where clan admins would enter their clantag and could even populate their clan page with a clan description. People really enjoyed this and never complained about the sorting of clans being based on KDR. I'm not sure if this was because nobody looked at the clan page, or they just didn't care. 🙂

Country statistics was added, showing which country had the most players, played the most, etc. Again, this was sorted by amount of Kills or KDR value (yep, countries had a kill/death ratio) instead of ELO score, so it wasn't accurate. People never complained about this either.

With the increase of douchebags and ego-boasting players that the ranker had created, we introduced a couple new features on player profiles, namely player reputation and player descriptions. Players could set their own description and anonymously give other players some reputation. Some of these reputation buttons included Pro, Noob, Lagger, Gold Star and some other negative ones which would give players a general overview of what a player's mentality was like. This seemed to create some sort of "trust" between players, since they would all agree on something ("this player is a total dick!") and other people would immediately see what kind of reputation a player has.

Since somewhere at the beginning of this year (2013), we've taken the ranker offline because of inactivity in Serious Sam HD versus. It was fun while it lasted, and I've learned a whole lot from it. I hope you did too, from reading this.
