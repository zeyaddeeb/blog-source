---
date: 2020-03-14
title: "Harvey, the movie!"
---

A while ago I was researching the origins of this quote from Stephen Hawking's [A Brief History of Time](https://en.wikipedia.org/wiki/A_Brief_History_of_Time):

> A well-known scientist (some say it was Bertrand Russell) once gave a public lecture on astronomy. He described how the earth orbits around the sun and how the sun, in turn, orbits around the centre of a vast collection of stars called our galaxy. At the end of the lecture, a little old lady at the back of the room got up and said: "What you have told us is rubbish. The world is really a flat plate supported on the back of a giant tortoise." The scientist gave a superior smile before replying, "What is the tortoise standing on?" "You're very clever, young man, very clever, " said the old lady."But it's turtles all the way down!" - Stephen Hawking

After some googling I learned about the book [Turtles All the Way Down](https://en.wikipedia.org/wiki/Turtles_All_the_Way_Down_(novel)), which made me a fan of [John Green](https://en.wikipedia.org/wiki/John_Green_(author)), who also produces  one of my favorite podcasts, [The Anthropocene Reviewed](https://www.wnycstudios.org/podcasts/anthropocene-reviewed).

In one of the [episodes](https://www.wnycstudios.org/podcasts/anthropocene-reviewed/episodes/anthropocene-reviewed-velociraptors-and-harvey), Green reviewed the 1950's movie _Harvey_.

![Harvey movie Poster](/images/harvey.jpg#center)

Spoiler alert ahead if you are planing on watching *Harvey*.

The movie follows the life of Elwood P. Dowd, a drunk middle-age man and his imaginary, people-sized rabbit friend, Harvey (the [Púca](https://en.wikipedia.org/wiki/P%C3%BAca)).

 Elwood has two main qualities. First, for most of the movie, he's the only person who can see this giant white rabbit. Second, he's exceptionally kind to everyone. It is his kindness that helps him successfully evade a lifetime stay in the psychiatric hospital. He explains his philosophy:

> In this world, you must be oh so smart or oh so pleasant. Well, for years I was smart. I recommend pleasant. 

After hearing this, I started to debate whether that is a good strategy from a computational game theory standpoint.

I remember using this python package `axelrod`, named after the famous political scientist [Robert Axelrod](https://en.wikipedia.org/wiki/Robert_Axelrod). The package essentially lets you model behavior displayed in the [Prisoner's Dilemma](https://en.wikipedia.org/wiki/Prisoner%27s_dilemma). In this setting, you have only two possible actions, to cooperate or defect. You choose your behavior based on the utility and the action of your opponent's strategy.

The package contains multiple strategies that players can adopt. If you follow Edward P Dowd's methodology, you'd be identified as a `Cooperator`, a player who will always cooperate, regardless of your opponent's behavior.

There are multiple player strategies in the package, the most famous ones include `TitForTat`, which describes a player who will always take the same action you take. Another is the `ForgetfulGrudger`, which is a player that starts by cooperating however will defect if at any point the opponent defects.

Given that there are currently over 200 strategies in the package, you can create tournaments and let the players compete with a stunning variety of outcomes.

We now have a theoretical way to test Dowd's claim that `Cooperator` is a winning strategy:

First we can start by defining a player called `Elwood` who is modeled after `Cooperator`:

```python
from axelrod.action import Action
from axelrod.player import Player

C, D = Action.C, Action.D

class Elwood(Player):
    """
    A player who only ever cooperates.
    """
    name = "Elwood"
    classifier = {
        "memory_depth": 0,
        "stochastic": False,
        "makes_use_of": set(),
        "long_run_time": False,
        "inspects_source": False,
        "manipulates_source": False,
        "manipulates_state": False,
    }

    @staticmethod
    def strategy(opponent: Player) -> Action:
        return C

```

Then you can import that player and set up a simple match against other strategies:

``` python
from typing import Tuple
import axelrod as axl
import pandas as pd

import Elwood from elwood_player

strategies = [s() for s in axl.strategies]

def play_match(opponent: axl.Player) -> Tuple[str, str]:
    if opponent == "Cooperator":
        pass
    players = [opponent, Elwood()]
    match = axl.Match(players, turns=10)
    match.play()
    return (opponent.name, match.winner())

results = [play_match(o) for o in strategies]

df = pd.DataFrame(results, columns=["player", "winner"])

print(
    f'Number of games where there are no winners: {round(df[df["winner"] == False].count() / df.count(), 3)[1] * 100}'
    "%"
)
# Number of games where there are no winners: 58.3% -- This number might be slightly different for everyone
```

After playing `Elwood` against all other strategies, you can see that the strategy never wins outright, but it does result in a tie 60% of the time.

Given these results, I'd say it makes good sense to be 'oh so pleasant' after all.

#### References

Game of thrones model:
[https://nikoleta-v3.github.io/2017/08/grudges-war-GoT.html](https://nikoleta-v3.github.io/2017/08/grudges-war-GoT.html)