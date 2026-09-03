# ha-list-bridge — reference

Information for looking up; nothing here tells you what to do. Setup and tasks are in
[`README.md`](README.md).


### Blueprint inputs

**To-do list bridge** — `blueprints/automation/gwilliams00/todo_bridge.yaml`

| Input | Type | Default | Meaning |
|---|---|---|---|
| Source to-do lists | one or more `todo` entities | — | Lists the trigger watches |
| Target to-do list | `todo` entity | — | Where items are added |
| Grace period | seconds, 0–300 | 20 | Wait before forwarding, so an undo on the source still works |

Mode queued. Trigger: `todo.item_added` on the sources.

**To-do list bridge — sweep** — `blueprints/automation/gwilliams00/todo_bridge_sweep.yaml`

| Input | Type | Default | Meaning |
|---|---|---|---|
| Source to-do list | `todo` entity | — | Its config entry is reloaded, then it is read |
| Target to-do list | `todo` entity | — | Where items are added |
| Run every | 15 / 30 / 60 minutes | 30 | `time_pattern` minutes `/15`, `/30`, or `0` |
| Settle time after the reload | seconds, 10–300 | 60 | Delay after the reload before reading |

Mode single. Calls `homeassistant.reload_config_entry` on the source entity.

**To-do list bridge — stuck-item alert** — `blueprints/automation/gwilliams00/todo_stuck_alert.yaml`

| Input | Type | Default | Meaning |
|---|---|---|---|
| Watched source lists | `todo` entities | none | Ignored when a pattern is given |
| Entity-id pattern | regular expression | empty | Matched against `todo.*` entity ids; overrides the list |
| Exclude pattern | regular expression | empty | Matching ids are never watched |
| Minutes before alerting | 1–240 | 10 | How long items may sit |
| Notification actions | actions | — | Run once per episode; `stuck` and `count` are available |

With neither a pattern nor watched lists the trigger has nothing to sum and never fires; the
blueprint does not validate this.

Mode single. Trigger: template over the watched entities' states, `for` the given minutes.
State counts of `unavailable` and `unknown` entities are ignored.

### A complete configuration

[`examples/alexa_to_anylist.yaml`](examples/alexa_to_anylist.yaml): the three automations as
HA stores them, with placeholders for your account prefix, your target list and your phone;
its header says how to use it.

### Entity id shapes

| Integration | Entity id | Example |
|---|---|---|
| Alexa Devices, built-in lists | `todo.<email with . and @ as _>_shopping_list`, `…_to_do_list` | `todo.jane_doe_gmail_com_shopping_list` |
| Alexa Devices, custom lists | same prefix + list name | `todo.jane_doe_gmail_com_grocery` |
| AnyList (moryoav fork) | `todo.anylist_<list name>` | `todo.anylist_grocery` |

The folder name `gwilliams00` in blueprint paths is the author's GitHub handle: HA files
imported blueprints under the author's name, and the repo mirrors that layout. Not a setting.

### Measured behaviour (HA 2026.7.4, aioamazondevices 14.1.9, AnyList fork 0.4.7)

| Quantity | Value |
|---|---|
| Alexa → HA push, adds and deletes, own account | 1–2 s |
| Spoken item → in AnyList | ≈ 25 s (20 s grace + writes) |
| Alexa Devices entry reload | ≈ 45 s, entities unavailable meanwhile |
| Household mirroring, other account → yours | under 5 min; the event does not reach HA live |
| Adding a name whose checked-off twin exists in AnyList | that item is reactivated; no duplicate |
| Amazon coordinator after ~25 calls in a few minutes | fails for ≈ 5 min; every `todo` action returns 500 |

### What changed, with sources

Checked 2026-09-03. Both APIs the bridge rides are unofficial; expect these to rot.

| Fact | Consequence | Source |
|---|---|---|
| **2024-07-01: Amazon retired List Skills and the List Management REST API.** Third-party apps can no longer be the target of Alexa's built-in list phrases, and cannot read Alexa's own lists | "Add milk to the shopping list" lands in Amazon's list and Alexa confirms it; the item never reaches your app and nothing tells you | [Amazon deprecated features](https://developer.amazon.com/en-US/docs/alexa/ask-overviews/deprecated-features.html) · [AnyList notice](https://help.anylist.com/articles/alexa-skill-update-july-2024/) |
| AnyList's replacement is an ordinary invocation-name skill, "Alexa, tell AnyList to add milk", with a configurable default list | Works only when Alexa routes the utterance to the skill; the developer has no say in that step | [AnyList skill commands](https://help.anylist.com/articles/alexa-skill-commands/) |
| **From March 2026 that routing fails widely**, especially under Alexa+. AnyList reported it to Amazon 2026-03-20; no fix. AnyList's own workarounds: "Alexa, open AnyList", then "add milk"; or "Alexa, disable Alexa+". AnyList asked for the Alexa+ APIs and was declined | This is the inconvenience the bridge removes; do not wait for either vendor to fix it | [AnyList, early 2026](https://help.anylist.com/articles/alexa-issue-early-2026/) |
| Google Assistant dropped third-party lists 2023-06-20 | Google speakers are not an escape route either | [9to5Google](https://9to5google.com/2023/05/31/google-assistant-notes-lists-integration/) |
| **AnyList has no public API.** Two reverse-engineered clients exist: `codetheweb/anylist` (JavaScript, v0.8.6 2026-05-03) and `anylist_rs` → `pyanylist` (Rust with Python bindings, v0.0.6 2026-03-17) | Any automated write into AnyList rides an unofficial API that AnyList could change or block; both are small hobby projects | [codetheweb/anylist](https://github.com/codetheweb/anylist/commits/master) · [pyanylist](https://pypi.org/project/pyanylist/) |
| One unresolved login failure: AnyList's token endpoint returned 401 for a user of the kevdliu add-on, filed 2026-06-18 | If the integration's setup dialog rejects a login the app accepts, this is why | [hacs-anylist #29](https://github.com/kevdliu/hacs-anylist/issues/29) |
| **HA 2026.7 added a to-do platform to core Alexa Devices** — shopping, to-do and custom lists as `todo.*` entities with create / update / delete, updated from Amazon's push feed. Contributed by the author of the archived `ha-alexa-todo-lists` custom component | The unlock: the native phrase keeps working and HA can read the result | [2026.7 release notes](https://www.home-assistant.io/blog/2026/07/01/release-20267/) · [Alexa Devices docs](https://www.home-assistant.io/integrations/alexa_devices/) · [archived component](https://github.com/lonlazer/ha-alexa-todo-lists) |
| Alexa Devices login needs the Amazon account on app-based two-step verification (authenticator, not SMS). Known limits: rate limiting makes entities briefly unavailable; must not run alongside `alexa_media_player` | Two account settings decide whether setup takes ten minutes or is blocked | [Alexa Devices docs](https://www.home-assistant.io/integrations/alexa_devices/) |
| HA 2026.9 re-reads a list after HA's own write (core PR #180598); 2026.7 does not | A missed push leaves a stale entity until a reload — the sweep exists for this | [core issue #179990](https://github.com/home-assistant/core/issues/179990) |
| With two Alexa Devices entries, only one receives pushed to-do events | Run one entry | [core issue #181229](https://github.com/home-assistant/core/issues/181229) |

### Traps

Symptom, cause, and what to do. Found on HA 2026.7.4.

| Symptom | Cause | What to do |
|---|---|---|
| A second Alexa Devices entry's list entity never changes after its first load | With two entries only one applies pushed to-do events; the other freezes at its initial sync | Run one entry — README, *How to handle a second Amazon account* |
| An item added on the other Household account shows in the Alexa app, on your list too, but not in HA | Mirroring is real but its push does not reach the entity | Wait for the sweep, or reload the entry |
| The alert never fired although an item sat for an hour | The alert reads HA's copy; HA never saw the item | Keep the sweep enabled; it refreshes that copy |
| A `todo.add_item` or `todo.remove_item` through HA "succeeded" but the entity still shows the old state | 2026.7 never re-reads after its own write; a missed push leaves the cache stale | Reload the entry, or upgrade to 2026.9+ |
| `todo.remove_item` by name says the item cannot be found | Amazon re-capitalises names ("bridge test" → "Bridge test"); HA matches summaries exactly | Remove by uid — the blueprints already do |
| Every `todo` action returns 500 for a few minutes after a burst of calls | The Alexa coordinator failed; entities are unavailable until the next successful poll | Wait five minutes; do not read a retry inside the window as evidence |
| Lists you never made appear in HA | Phrases like "add X to my grocery list" make Alexa create a custom list of that name | Alexa app → More → Lists & Notes → archive → View Archive → delete; watch by pattern so new ones are caught |

