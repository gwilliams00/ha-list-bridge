# ha-list-bridge

Home Assistant blueprints that forward items from one to-do list into another. Built so that
**"Alexa, add milk to the shopping list" reaches AnyList again**; generic enough for any pair of
`todo` entities (Alexa or Google lists → Todoist, Bring, Mealie, Grocy, AnyList, or the reverse).

## Features

- **Inbox, not mirror.** An item spoken to Alexa is copied to your shopping app and then removed
  from the Alexa list, so nothing is counted twice and anything left on the Alexa list means
  something is wrong.
- **Undo still works.** A short grace period before forwarding, so "Alexa, remove that" removes
  the item before it is copied.
- **Survives a lost push event.** A scheduled reload re-reads the Alexa list from Amazon and
  forwards whatever the live feed missed — including items mirrored from a second Alexa
  Household account.
- **Tells you when it is broken.** One notification when an item has sat on the Alexa list too
  long, with the list named.
- **Generic.** Any `todo` entity to any `todo` entity; nothing in the blueprints knows about
  Alexa or AnyList. Three blueprints, twelve inputs, no custom code.
- **Documented failure modes.** Every trap found while building it, with cause and fix. The one
  that is Home Assistant's own — a second Alexa Devices entry never receives pushed list events —
  is filed as [core #181229](https://github.com/home-assistant/core/issues/181229).

Three parts follow: **Why this exists** (the problem and the design), **Set it up** (a first
install, end to end — start here if you just want it working), and **How to** (changes you make
later). Every input, source, measurement and trap is in [`REFERENCE.md`](REFERENCE.md).

---

## Why this exists

### The problem

On 2024-07-01 Amazon retired the Alexa List Skills API, so third-party apps can no longer be the
target of Alexa's built-in list phrases ([AnyList's notice](https://help.anylist.com/articles/alexa-skill-update-july-2024/)).
"Add milk to the shopping list" lands in Amazon's own list, and Alexa confirms it cheerfully.
AnyList's replacement, "Alexa, tell AnyList to add milk", depends on Alexa routing the phrase to
the skill, which has failed widely since March 2026, especially under Alexa+, and AnyList has
said only Amazon can fix it ([AnyList, early 2026](https://help.anylist.com/articles/alexa-issue-early-2026/)).

### The idea

Home Assistant 2026.7 added a to-do platform to the core **Alexa Devices** integration: Alexa's
own lists become `todo` entities, updated live from Amazon's push feed. So keep the native
phrase, let Alexa's list take it, and bridge it inside HA to the list you actually shop from.

```
Echo ──"add milk"──▶ Alexa shopping list ──alexa_devices──▶ todo.<account>_shopping_list
                                                                    │ todo.item_added
                                                                    ▼
                 target list ◀── todo.add_item ── bridge ── todo.remove_item (source)
```

### Why three automations, not one

- **The bridge** is the instant path: an item spoken to Alexa is in the target in about 25
  seconds. It waits 20 seconds first so Alexa's own "remove that" still works — an undo arrives
  as a deletion, and the re-check after the delay simply no longer sees the item. It forwards
  only the items its trigger announced, so an item that cannot be removed is not copied again on
  every later add. It skips names already active on the target, which keeps the target's own
  merge behaviour intact, and it removes by uid because Amazon re-capitalises names. A refused
  removal is tolerated; a failed add stops the run, so the item stays on the Alexa list where
  the alert can name it.
- **The sweep** is the floor under the bridge. Amazon's push feed is reliable for items you add
  yourself and unreliable for everything else — items mirrored from another Alexa Household
  account arrive silently, and a second Alexa Devices entry never receives its events at all.
  Nothing in HA 2026.7 re-reads a list on its own; a config-entry reload is the only full
  re-read. So every half hour the sweep reloads the source entry and forwards whatever the feed
  missed. The worst case for any item, from any cause, becomes one interval, not never.
- **The alert** exists because Alexa says "added" whether or not anything downstream is alive.
  It watches HA's copy of the lists, which is exactly why it needs the sweep: an item HA never
  saw could never trip it.

### What already existed

Three things solve neighbouring problems:

- **[Dr. Mor's Alexa ↔ AnyList sync](https://smarthome.yoavmor.com/home-assistant/sync-alexa-shopping-list-anylist-home-assistant-no-nodered/)**
  (July 2026), by the author of the AnyList integration this recipe uses. The same two
  integrations, wired as a *mirror*: whichever list changed becomes the source and the other is
  updated to match, so items live on both lists. A pasted automation rather than a blueprint.
  If you want the Alexa list to stay a full copy of your shopping list, use that.
- **[To-do List Sync — one-way items, two-way completion](https://github.com/ausdi/cookidoo_bring_sync.yaml)**
  (August 2026), a blueprint built for Cookidoo → Bring!: items flow one way, completion flows
  both ways, items stay on the source, with a five-minute fallback pass.
- **[Synchronize 2 Home Assistant todo lists](https://community.home-assistant.io/t/synchronize-2-home-assistant-todo-list/684390)**
  (2024), a two-way sync originally for Google Keep and Bring!.

This repo differs in the first four items under *Features*: inbox rather than mirror, undo
preserved, missed pushes recovered, and an alert. If those are not the properties you want, one
of the three above is closer.

### What it costs

Both halves ride private APIs with hobbyist maintainers: Amazon's mobile-app API and AnyList's
reverse-engineered client. The Amazon account must use authenticator-app two-step verification.
The Amazon and target-app passwords end up in HA's storage. The source's Echo entities go unavailable for
about 45 seconds at every sweep. And HA becomes load-bearing for groceries: a restart pauses the
bridge, though nothing is lost, because the item waits on the Alexa list.

---

## Set it up

Alexa and AnyList as the example; about 30 minutes; each step ends with something you can see.

### 1. Prepare the Amazon account

In the Amazon account the Echos are registered to: Your Account → Login & Security → 2-Step
Verification → add an **Authenticator App** and make it preferred. The integration will not
accept SMS codes. If your Echos can switch between two Amazon accounts (an "Alexa Household"),
pick one account for HA and stay with it — a second entry does not help; see *How to handle a
second Amazon account*.

### 2. Add Alexa Devices

Settings → Devices & services → Add integration → **Alexa Devices** → email, password, and the
current code from the authenticator. When it finishes, open **To-do lists** in the sidebar: you
should see one entry per Alexa list, named after the account. Say "Alexa, add corn to the
shopping list" and watch the shopping list entry gain an item within a couple of seconds. Then
"Alexa, remove that" and watch it go. If either does not happen, stop here; nothing downstream
will work.

### 3. Add the target list

For AnyList: HACS → search **AnyList** → the one by *moryoav* (in the default catalogue) →
Download → restart HA → Add integration → AnyList → your AnyList login → expose only the list
you shop from. Open To-do lists again: the AnyList list appears with its real items. Type an
item into it from HA and check it shows in the AnyList app on a phone.

For another app, add its integration the same way and note its `todo` entity id.

### 4. Clear the Alexa list

In the Alexa app, delete everything on the shopping list. The bridge fires only for items added
after it exists, and anything old would sit there tripping the alert immediately.

### 5. Import the three blueprints

Settings → Automations & scenes → **Blueprints** → **Import blueprint**, three times, with these
URLs:

```
https://raw.githubusercontent.com/gwilliams00/ha-list-bridge/main/blueprints/automation/gwilliams00/todo_bridge.yaml
https://raw.githubusercontent.com/gwilliams00/ha-list-bridge/main/blueprints/automation/gwilliams00/todo_bridge_sweep.yaml
https://raw.githubusercontent.com/gwilliams00/ha-list-bridge/main/blueprints/automation/gwilliams00/todo_stuck_alert.yaml
```

If you cloned the repo instead, copy the three files into
`<config>/blueprints/automation/gwilliams00/`.

### 6. Create the three automations

First find your two entity ids: Settings → Devices & services → **Entities**, filter on
`todo.`. The Alexa shopping list looks like `todo.jane_doe_gmail_com_shopping_list`; the part
before `_shopping_list` is your **account prefix**. Note your app's list entity too.

Then Settings → Automations & scenes → **Create automation** → pick the blueprint:

| Blueprint | Fill in |
|---|---|
| To-do list bridge | Source to-do lists: the Alexa shopping-list entity. Target to-do list: your app's list entity. Grace period: 20. |
| To-do list bridge — sweep | Source to-do list: the same Alexa entity. Target to-do list: the same. Run every: 30 minutes. Settle time after the reload: 60. |
| To-do list bridge — stuck-item alert | Entity-id pattern: `todo\.<account prefix>_`. Exclude pattern: `_to_do_list$` (the Alexa To-do list holds real to-dos, so it must never alert). Minutes before alerting: 10. Notification actions: one notify action, for example `notify.mobile_app_<your phone>` with the message `Items have sat for 10 minutes on: {{ stuck }} ({{ count }} open).` |

On the alert, fill either *Entity-id pattern* or *Watched source lists*; with both empty it can
never fire. [`examples/alexa_to_anylist.yaml`](examples/alexa_to_anylist.yaml) is all three,
filled in.

### 7. Prove it

Say "Alexa, add corn to the shopping list". Within about 25 seconds corn is in your app and gone
from the Alexa list. Say "Alexa, add tofu to the shopping list" and then "Alexa, remove that"
within 20 seconds: tofu never arrives.

Then prove the two safety nets once, because they only matter on the day the bridge fails:
disable the bridge automation, say "Alexa, add rice to the shopping list", and wait. Within ten
minutes your phone gets the alert naming the shopping list; within thirty, the sweep has moved
rice into your app anyway. Re-enable the bridge. You are done.

---

## How to

Each guide assumes the setup above is working.

### How to point it at a different app

Add that app's to-do integration, then edit the bridge and sweep automations and change the
target to its entity. Nothing else changes. The dedupe compares item names case-insensitively
against the target's open items, so the target's own behaviour for repeated names is preserved.

### How to change how often the sweep runs

Edit the sweep automation → Run every. Thirty minutes means an item that missed the push feed
waits at most thirty minutes. Hourly halves the number of reloads. Each reload takes about 45
seconds, during which the source entry's entities are unavailable. The sweep is still needed on
HA 2026.9 and later: that release re-reads a list after HA's own writes, not after Amazon's.

### How to tell whether AnyList still accepts logins

A 401 from AnyList's token endpoint was reported in mid-2026, and every unofficial client goes
through that endpoint. The integration's own setup dialog is the test: if adding AnyList fails
with an authentication error while the same login works in the AnyList app, the API has changed
under the client, and no version of the bridge will help until the integration is updated.

### How to handle a second Amazon account

If your Echos switch between two Amazon accounts, do not add a second Alexa Devices entry: with
two entries only one receives pushed to-do events ([core #181229](https://github.com/home-assistant/core/issues/181229)).
Instead rely on the Alexa Household, which mirrors items added on the other account onto yours
within minutes, and on the sweep, which is what makes those mirrored items reach the target.
Your login can delete them from both lists.

### How to tell whether the bridge or the feed is broken

Open To-do lists in HA next to the Alexa app. If an item is on the Alexa list in the app but not
in HA's copy, the push feed missed it; the next sweep will pick it up, and a manual reload of
the Alexa Devices entry picks it up now. If it is in HA's copy and still there ten minutes
later, the bridge or the target failed: Settings → Automations → the bridge → Traces shows the
run and its error.

## Status

In use since 2026-09-03. MIT licence. Issues and pull requests welcome; the two
APIs this rides are unofficial and will change, so a report of "it stopped working on version X"
is useful on its own.
