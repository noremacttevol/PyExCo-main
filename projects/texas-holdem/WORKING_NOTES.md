# Texas Hold'em — Rough Draft Working Notes
*Cameron Lovett — MLS-102 Final Project*

---

## What I was building

A playable Texas Hold'em game in the terminal. Players: me (human input) + 3 computer
opponents. The game tracks chips, rotates blinds, runs full betting rounds
(Pre-Flop, Flop, Turn, River), and picks a winner at showdown.

I based the class structure on Daniel's BlackJack demo — same idea with a Manager class,
a base Player class, and a subclass that overrides one method for human input.

---

## Problems I ran into and how I fixed them

### Problem 1 — Big Blind was going first pre-flop

When I tested the first hand, BB was being asked to act before UTG and the button.
That's backwards — BB should act *last* pre-flop because they already put chips in.

I traced it back to how I was building the action queue.
I was just looping through `self.players` in list order, which meant player at index 0
(whoever that was) always went first. No order control at all.

**Fix:** I built an explicit `preflop_order` list that starts at UTG and ends at BB.
UTG is one seat left of BB, so:

```python
utg = (bb + 1) % n
preflop_order = [self.players[(utg + i) % n] for i in range(n)]
```

Then I passed that list into the betting round so the queue starts from UTG.

---

### Problem 2 — Post-flop order was wrong

After the flop I had the same problem — players were acting in seat order, not poker order.
Post-flop should be: SB first, then going clockwise, button last.
The button has position every post-flop street, meaning they always act last.

**Fix:** Built a separate `postflop_order` starting at SB:

```python
postflop_order = [self.players[(sb + i) % n] for i in range(n)]
```

Used this for Flop, Turn, and River betting rounds.

---

### Problem 3 — Same player was BB every single hand

The blinds never rotated. I had `players[0]` and `players[1]` hardcoded
as SB and BB, so it was always the same two people regardless of how many hands
we'd played.

Real poker: the button moves one seat left every hand, and SB/BB follow it.

**Fix:** Added `self.btn_idx = 0` to `__init__`. Each hand I compute positions from it:

```python
n   = len(self.players)
btn = self.btn_idx % n
sb  = (btn + 1) % n
bb  = (btn + 2) % n
```

At the end of each hand: `self.btn_idx = (self.btn_idx + 1) % len(self.players)`

The `% n` part is the key — it wraps the index back to 0 when it goes past the end
of the list. So if btn is at index 3 and there are 4 players, next hand btn moves
to index 0. No player is skipped, no one gets stuck as BB forever.

---

### Problem 4 — No "check" option when there was no bet

The computer was folding on weak hands even when there was no bet to call.
Folding for free is wrong — you always take the free check.

Also the human prompt only ever showed "fold / call / raise" even when to_call was 0.

**Root cause:** I wasn't tracking *how much* each player had already put in during the
current round. SB had posted 25 but the code thought they owed the full 50, so
`to_call` was wrong.

**Fix 1 (tracking):** Built an `amount_in` dictionary — one entry per player, seeded
with whatever the blinds put in:

```python
amount_in = {p.name: (player_amounts or {}).get(p.name, 0) for p in self.players}
```

Then each player's actual cost to call is:
```python
to_call = max(0, current_bet - amount_in[player.name])
```

**Fix 2 (AI logic):** Updated computer `decide_action` so weak hands check instead of fold
when `to_call == 0`:

```python
return "fold" if to_call > 0 else "check"
```

**Fix 3 (human prompts):** Split the prompt into three cases:
- `to_call > 0` → "fold / call / raise"
- `current_bet > 0` but `to_call == 0` → "check or raise (you're covered)"
- No bet at all → "check for free"

---

### Problem 5 & 6 — "raise" didn't accept an amount, raise was always fixed

Typing `raise` would raise by exactly one big blind regardless of what you wanted.
And you couldn't type `raise 200` on the same line — it would just error or ignore the number.

**Fix:** In `decide_action` for the human player, I split the input into a verb and the rest:

```python
words = choice.split()
verb  = words[0] if words else ""
rest  = words[1].lstrip("$") if len(words) > 1 else ""
```

If `verb` is "raise" and `rest` has a number, parse it as the raise amount.
If `rest` is "all" → all-in.
If no amount given → prompt on the next line.

I return a **tuple** instead of just a string:

```python
return ("raise", 200)
```

Then in the manager, I check if the return value is a tuple:

```python
if isinstance(action, tuple):
    action, raise_by = action
```

AI players still return plain `"raise"` — the manager defaults `raise_by` to BIG_BLIND
for those cases.

---

### Problem 7 — Call display was confusing

When SB called, it printed "calls $25" even though the current bet was $50.
That's because SB already posted 25, so they only *add* 25, but the message
wasn't reflecting that.

**Fix:** Three-way print in the call handler:
- If they couldn't afford the full call → "calls all-in for $X"
- If they had chips in already and are completing → "calls $50 (adds $25)"
- Normal full call → "calls $50"

---

### Card display

Original: `[Ace of Spades]` — too wordy, hard to read a hand at a glance.

**Fix:** Added two lookup dictionaries to `card.py`:

```python
RANK_SHORT  = {"Ace": "A", "King": "K", ..., "Two": "2"}
SUIT_SYMBOL = {"Spades": "♠", "Hearts": "♥", "Diamonds": "♦", "Clubs": "♣"}
```

Updated `__repr__` to use them:
```python
def __repr__(self):
    return f"[{self.RANK_SHORT[self.rank]}{self.SUIT_SYMBOL[self.suit]}]"
```

Output: `[A♠]  [K♥]  [10♦]` — much cleaner.

---

---

## Round 2 — Fixes found during playtesting

### Card display was too wordy

`[Ace of Spades]` is hard to read fast at a glance. Changed to `[A♠]`.

Added two lookup dictionaries to `card.py`:
- `RANK_SHORT` maps the word ("Ace") to the short version ("A")
- `SUIT_SYMBOL` maps the word ("Spades") to the Unicode character ("♠")

Updated `__repr__` to use them. Python 3 supports Unicode characters directly in strings — `♠ ♥ ♦ ♣` work without any imports.

---

### Showdown picked the wrong winner (same hand type, wrong kicker)

When two players both had One Pair, the game always picked player[0] (me).
The hand evaluator was returning a single integer like `1` for all One Pair hands.
When Python does `max()` on tied integers it just picks the first one it sees.

**Fix:** Changed the score from a plain integer to a **tuple** that includes the pair
value and kicker values in priority order:

```python
return (1, rv[0], *kickers), "One Pair"
# e.g. pair of Queens with A, K, 8: (1, 12, 14, 13, 8)
# pair of 7s with A, K, 6: (1, 7, 14, 13, 6)
```

Python compares tuples element by element — first element ties (both 1), second
element: 12 > 7, so Queens wins. No extra code needed; the `max()` call already works.

Also fixed the straight flush bug — old code called it a straight flush any time
there was a flush AND a separate straight, even if they were different suits. Fixed
it to only count a straight flush when the 5 consecutive cards are all the same suit.

---

### At 2 players, same player was asked twice — SB never completed blind

When the game got to heads-up, the pot stayed at $75 instead of going to $100.
SB was never asked to call.

Root cause: the formula `bb = (btn + 2) % n` when `n == 2` gives `bb == btn`.
I had a special `if n == 2` branch for action order that was building
`[players[btn], players[bb]]` — which became `[Cameron, Cameron]` — same player twice.

**Fix:** Deleted both heads-up special cases. The general formula already handles n=2:

```python
utg = (bb + 1) % n   # when n=2 this equals sb, so SB goes first
preflop_order = [self.players[(utg + i) % n] for i in range(n)]
```

For n=2, that produces `[SB, BTN]` — both different players, correct order.

---

### Typed wrong key accidentally and lost the game

The "play another hand?" prompt used to quit on anything that wasn't "y".
Accidentally pressing enter or any other key ended the session.

**Fix:** Wrapped it in a `while True` loop that only exits on `y`, `n`, or `s`.
Anything else just asks again.

Added `s` as a save option — writes chip counts and button position to
`holdem_save.json`. Next launch detects the file and offers to resume.

```python
while True:
    again = input("Play another hand? (y / n / s to save): ").strip().lower()
    if again in ["y", "yes"]:   break
    elif again in ["n", "no"]:  game_on = False; break
    elif again in ["s", "save"]: self.save_game(); game_on = False; break
    # else: loop re-asks, don't quit
```

Save uses `json.dump()` to write a dictionary to a file.
Load uses `json.load()` to read it back and rebuild the player list.

---

### Had to type full word — added single-letter shortcuts

Typing "fold", "call", "check", "raise" every time was slow.
Added shortcuts: `f` / `c` / `r`.

`c` is context-aware: if there's a bet it calls, if there's no bet it checks.
Both can't happen at the same time so `c` always does the right thing.

Updated prompts to show the shortcuts: `fold / call / raise (f/c/r)`.

---

## Key Python patterns I used

| Pattern | Where |
|---------|-------|
| Modular arithmetic `%` | Rotating btn/sb/bb positions |
| List comprehensions with conditions | `[p for p in order if not p.folded]` |
| Dictionaries | `amount_in`, `RANK_SHORT`, `SUIT_SYMBOL` |
| Default parameters | `def decide_action(self, ..., to_call=0)` |
| Tuple returns | `return ("raise", 200)` from human player |
| `isinstance()` | Checking if action is a tuple or string |
| `__repr__` | Custom card printing |
| Unicode characters | `♠ ♥ ♦ ♣` work in Python strings |
| Inheritance + override | `HmnPlayer` overrides `decide_action` from `CmpPlayer` |
