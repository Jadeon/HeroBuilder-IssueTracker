# Hero Siege Build Interchange, v1

**Status:** draft, v1. **Scheme name:** `hsbi/1` (Hero Siege Build Interchange, version 1).

Author: HeroSiege Build Engine project. Audience: any third-party tool that wants
to import or export a Hero Siege build to or from another tool. Written so an
implementer can build against it with no access to this project's source.

This document is prose. Rules are stated with the reason behind them, because
a rule without a reason gets implemented wrong at the first ambiguity that the
spec did not literally spell out.

## 1. What this is, and what it is not

**What it is.** A file format for one Hero Siege build. JSON. A build is the
character sheet a planner works with: class and level, spent attributes and
skill points, equipped items and their affix rolls, the tree the player has
walked, and every other player-authored input that determines how the character
performs.

**What it is not.** A save file. Not a wire protocol. Not a URL-compressed
share code. It is the shape of the payload. How that payload gets from tool A
to tool B, whether by file download, clipboard paste, or HTTP POST, is
outside this document.

Specifically, HSPlanner
(github.com/HeroSiegePlanner/HSPlanner) already ships a share feature that
"export[s] the entire build to a compressed URL (lz-string)"
(HSPlanner README, fetched 2026-08-19). That compressed URL is HSPlanner's
own internal format. This spec does not describe it and does not compete with
it. A tool is free to keep its own internal format and to translate to and
from `hsbi/1` at the export and import boundary. Interop only requires that
both sides agree on the shape of what crosses the boundary.

## 2. Scope for v1

**In scope for v1:** one complete build in one JSON document. Class, level,
attributes, allocated skill and talent tree state, both the class tree and
the Ether board, gear per slot with sockets and affixes, charms, relics,
and jewels.

**Out of scope for v1, on purpose:**

- **Multiple builds and profiles per document.** Both tools organise saved
  builds inside a library or a builds menu of their own design. That is a
  container concern. Standardising it in v1 would force every tool to accept
  another tool's library layout, which is not the interop question anyone is
  asking. Multiple-build packages, if they are ever needed, are a v2 topic and
  will be a thin wrapper around v1 documents, not a redesign.
- **Encoded or compressed transport.** Base64, `lz-string`, or any other
  wrapping is a decision for the calling tool, not the format.
- **Character progression history, kill counts, playtime, screenshots, chat
  bio.** These are not build inputs. A build snapshot is what determines
  computed stats; nothing else belongs.
- **Season and patch pinning.** A build may declare the patch it was authored
  under (see the `game` block), but v1 does not describe migration rules
  between patches. Season 10 lands 2026-08-21 and will rework classes; the
  version marker in the envelope is what lets a future spec migrate cleanly.

**Why the scope is this tight.** Every field this document standardises is a
field two tools have to agree on forever. A field that stays out of v1 can
be added in v2 without breaking anyone. A field that ships wrong in v1
cannot.

## 3. The envelope

Every document is a JSON object with these top-level keys, in this shape:

```json
{
  "spec": "hsbi/1",
  "game": {
    "id": "herosiege",
    "patch": "2.7.0.0",
    "patchDate": "2026-05-22"
  },
  "producedBy": {
    "tool": "HeroSiege Build Engine",
    "toolVersion": "0.1.0",
    "generatedAt": "2026-08-19T00:00:00Z"
  },
  "build": { ... }
}
```

### 3.1 `spec` (required)

The literal string `"hsbi/1"`. Version-first, from day one, because a format
that ships without a version marker cannot be evolved without breaking every
existing importer. A future v2 will use `"hsbi/2"`, and importers must
refuse a document whose scheme they do not recognise (see §7).

### 3.2 `game` (required)

Identifies what game the build is for. `id` is a stable string. For Hero
Siege the value is the literal `"herosiege"`. This field exists because the
underlying build engine is game-agnostic by design (multi-game packs), so a
future document for a different game will be distinguishable at the envelope
level without having to inspect the payload.

`patch` is the game version string the build was authored under. `patchDate`
is the release date of that patch, ISO-8601. Both are optional, but strongly
recommended: numeric values in Hero Siege are season-versioned, and a
consumer that receives a build without knowing which patch produced it has
to guess.

### 3.3 `producedBy` (optional)

Free-form provenance. Useful for debugging and for the receiver to tell one
tool's quirks from another's. Never load-bearing: an importer must not
change its behaviour based on which tool produced the document, because
that turns the format into a per-vendor negotiation.

### 3.4 `build` (required)

The build itself. See §4.

## 4. The build object

```json
{
  "build": {
    "identity": {
      "class": "Viking",
      "level": 100,
      "difficulty": "inferno",
      "title": "Frost Shieldbearer, Inferno viable"
    },
    "attributes": {
      "strength": 200, "dexterity": 20,
      "vitality": 495, "energy": 20
    },
    "skills": {
      "unspentPoints": 0,
      "allocated": [
        { "id": "weaponMaster", "rank": 20 }
      ]
    },
    "trees": {
      "class": { ... },
      "ether": { ... },
      "incarnation": { ... }
    },
    "gear": {
      "helmet": { ... },
      "amulet": { ... },
      "bodyArmor": { ... },
      "weapon1": { ... },
      "weapon2": { ... },
      "gloves": { ... },
      "belt": { ... },
      "boots": { ... },
      "ring1": { ... },
      "ring2": { ... },
      "flask1": { ... },
      "flask2": { ... }
    },
    "charms": [ ... ],
    "relics": [ ... ],
    "notes": "Optional plain-text notes from the author."
  }
}
```

Notes on the block:

- **`identity.class`** is the English display name of the class as it appears
  in the client (for example `"Viking"`). Class names are stable across
  locales in the client's internal naming: the client's own class list is
  keyed by these English strings (see [data/extracted/class-trees.json](../data/extracted/class-trees.json)
  keys under `classes.*`). A numeric class code (`class_value`) exists inside
  the client but is not the interop key, because a numeric code is opaque to
  a human reading the JSON and to anyone maintaining a per-tool mapping table.
- **`identity.level`** is the player level, integer, 1 to the current cap.
  The v1 spec does not enforce a maximum, because the cap changes with the
  season.
- **`identity.difficulty`** is one of the string values matching the game's
  own difficulty names, lower-case. This is optional; when absent the
  receiver may render the build without a difficulty-dependent verdict.
- **`skills.allocated[].id`** is the client's own ability identifier, in the
  form the client uses internally (camelCase strings such as `"weaponMaster"`,
  visible in [data/extracted/class-trees.json](../data/extracted/class-trees.json)
  under `classes.Viking.specs[].skills[].ability_id`). See §5.2 on why
  identifiers travel by client id rather than by display name.
- **`trees.class`**, **`trees.ether`**, and **`trees.incarnation`** are
  sparse allocation maps: each entry names a node the player has invested in,
  by the client's node identifier, with the number of points invested. The
  shape is described in §5.3.
- **`gear.*`** slot names are the singular, camelCased slot nouns
  (`"helmet"`, `"amulet"`, `"bodyArmor"`, `"weapon1"`, `"weapon2"`,
  `"gloves"`, `"belt"`, `"boots"`, `"ring1"`, `"ring2"`, `"flask1"`,
  `"flask2"`). A slot with no item is either absent or `null`; both are
  legal and mean the same thing.
- **`charms`** and **`relics`** are ordered arrays of item entries because
  their position in the player's inventory or belt is authored, and swapping
  positions is a real edit.

## 5. Item entries

The most important shape in the spec after the envelope.

### 5.1 The shape

```json
{
  "id": "boots_aetherflow_treads",
  "rarity": "satanic",
  "name": "Aetherflow Treads",
  "level": 85,
  "sockets": [
    { "type": "rune", "id": "ol" },
    { "type": "empty" }
  ],
  "affixes": [
    { "id": "life", "value": 62 },
    { "id": "lightning_resistance", "value": 20 },
    { "id": "movement_speed", "value": 18 }
  ]
}
```

### 5.2 Why the id, the rarity, and the name are three separate fields

**`id` is the join key.** It must be the client-derived identifier for the
item, in the naming form the producing tool uses natively. The receiver is
responsible for translating from its counterpart's naming form to its own,
using an id mapping table maintained out of band. See §5.4.

**`rarity` is a separate string field**, not baked into the id, with the
values the game itself uses (`"normal"`, `"heroic"`, `"angelic"`, `"satanic"`,
`"unholy"`, and any that later seasons add). It is separate because
different tools infix rarity into the item id differently, and forcing one
convention on the id would force every tool to rewrite its item table.
Concretely: HSPlanner's identifier for the item pictured above is
`boots_satanic_aetherflow_treads` (rarity infixed), and this project's own
extraction key for the same item is `boots_aetherflow_treads` (no rarity in
the key). Both forms are found together in
[data/extracted/crosscheck_disagreements.json](../data/extracted/crosscheck_disagreements.json)
lines 38 to 43. Neither is wrong. Neither is going to change. The spec
sidesteps the argument by carrying rarity out of band.

**`name` travels with the entry, but is not the join.** It is a human check.
Names are subject to being renamed by the game between patches, are
localised, and are not unique across items in any given locale (see
[data/extracted/class-trees.json](../data/extracted/class-trees.json)
`_provenance.functions.spec_names` for how the client itself keys spec names
by translation table row rather than by display string). If the receiver
finds that the name it resolves the incoming `id` to disagrees with the
`name` in the entry, the correct behaviour is to warn the user with both
strings and continue, not to abort. Names change; identifiers must not.

### 5.3 Sockets, affixes, and nested nodes

- **`level`** is the item level, integer, optional. Absent means unknown.
- **`sockets`** is an ordered array. Each socket is either occupied by a
  socketable (rune, jewel, or tree jewel), or empty. Empty sockets are
  represented explicitly, because "no socket in slot 2" and "an item with
  only one socket" are different builds. `type` distinguishes them.
- **`affixes`** is an unordered array of `{ id, value }` pairs. `id` is the
  client's affix identifier. `value` is the rolled magnitude. Ranges are
  not carried in the interchange payload: a build represents one specific
  rolled item, and range metadata lives in the game data pack that both
  tools ship, not in the build itself.

### 5.4 The join between producing tool and consuming tool

The receiver runs the incoming `id` through a mapping table it maintains. A
seed mapping between this project's `exe_key` values and HSPlanner's `hsId`
values exists in
[data/extracted/crosscheck_disagreements.json](../data/extracted/crosscheck_disagreements.json).
Of 899 rows in that file, roughly half carry both identifiers together
(counted by grep on 2026-08-19: 899 rows total, 451 rows with `exe_key: null`,
the remainder resolved). The mapping is imperfect. Some items cannot be
resolved from public data alone. See §6 for what to do when an entry
cannot be resolved.

Neither tool is expected to store the other tool's id space internally.
The mapping table lives at each tool's import boundary, not in its data
model.

## 6. Unknown fields must be preserved, never dropped

**This is the most important rule in the document.**

An implementation of `hsbi/1` may encounter fields it does not understand.
At the envelope level, at the build level, inside a gear entry, inside a
socket, inside an affix, anywhere. When it does, it must carry those
fields through unchanged on any subsequent export.

### 6.1 Why this rule exists

Both planners are actively evolving. Fields the other tool tracks today,
and fields either tool adds tomorrow, will not all be known to every
implementation at the moment a build is passed between them. If an
importer drops what it does not recognise, a build that round-trips through
tool A on its way from tool B to tool B loses whatever tool A could not
render, silently. The user's build was destroyed by software that meant
well.

That is the failure mode this rule exists to prevent. It is not a nice-to-
have. Any implementation that drops unknown fields is non-conforming.

### 6.2 How to implement it

An implementation has two acceptable techniques. Either is fine; the choice
is internal.

**Technique A, structural preservation.** The importer parses the document
into its internal model, and separately keeps the original JSON object
tree. On export, it merges its model back onto the retained tree, so any
key not overwritten by the model is emitted unchanged. This is the more
robust technique and is recommended for any implementation that intends to
support round-trips.

**Technique B, extra buckets.** The importer walks the document and, at
each object level, buckets any key it does not recognise into a reserved
`_extra` object on the corresponding node in its internal model. On export
it re-emits those keys from `_extra` at the same level. This is easier to
build and works cleanly for shallow unknowns. It does require the
implementer to remember `_extra` in every place they emit an object.
The key name `_extra` is reserved by this spec for this purpose. An
implementation must not use it for its own recognised fields.

### 6.3 The one exception

An implementation may drop unknown fields when the user has explicitly
asked for a clean re-export, and the tool has surfaced what will be dropped
so the user can decide. "Clean re-export" is a user-visible destructive
action, not a default.

### 6.4 What this rule does not allow

- It does not allow silent conversion of an unrecognised affix into a
  recognised one that happens to have the same numeric value. Preserve the
  original.
- It does not allow deletion of a socket the tool cannot render. Preserve
  the socket and mark the item as "renders partially" in the tool's UI, or
  refuse the import per §7. The build data is not the tool's to trim.
- It does not allow renaming a field to a tool's preferred casing on
  export. Emit the field with the key the tool received.

## 7. Unresolvable entries on import

**AWAITING THE OWNER'S DECISION.** This section presents the two live
options and this document's recommendation. Nothing here should be treated
as final until the owner signs off.

An unresolvable entry is one where the importer's mapping table has no
translation for a given `id`, or where the entry references a slot,
enchant, or subsystem the importer's data pack does not know about. This
is distinct from §6, where the field is understood but semantically
unfamiliar. Here the reference itself cannot be resolved.

### Option A. Import what resolves, name what was dropped.

The importer accepts the document, produces a build from every entry it can
resolve, and returns a machine-readable list of every entry it could not
resolve, alongside a human-readable summary. The user sees the build come
in with holes and the tool tells them exactly where the holes are.

**Pros.** The user gets to work with as much of the build as the tool can
represent. Partial builds are still useful for comparison, screenshots, and
theorycraft. The user is never surprised, because the tool reports the
gaps.

**Cons.** A partial build silently changes the character's computed
performance. A player who compares DPS between tools may not realise the
difference is that half the affixes did not import. Even a clearly labelled
partial import is easily missed in a screenshot shared out of context.

### Option B. Refuse the whole import when any entry is unresolvable.

The importer refuses the document and returns the list of unresolvable
entries as the reason. The user can then either fix their source tool's
export, or wait for the receiving tool's data pack to catch up, or choose a
different build to import.

**Pros.** The tool never displays a build that is not the build the author
sent. Comparisons between tools remain trustworthy. There is no silent
damage.

**Cons.** The user cannot get any of the build across when a single entry
does not translate. When the two tools' data packs are close but not
identical, a common state today, most imports would fail on a single
missing affix.

### Recommendation

**Option A, with a hard error surface that is not dismissable inside the same
gesture.** The user must see the "N entries did not import" report and
acknowledge it before the build enters the working editor. This preserves
usefulness in the common case (the two tools disagree on a small handful of
new items) while making silent damage difficult. The gesture split (import
plus separate acknowledgement) is what distinguishes it from a toast the
user will train themselves to dismiss.

A conforming implementation must document which option it takes so that a
user reading the tool's release notes knows whether their partial imports
are silent or hard-gated.

This choice is deferred to the owner. Whichever way the owner rules, this
section will be edited to state the ruling and mark the alternative as
rejected.

## 8. Compatibility and versioning

- **Scheme changes** are signalled by the `spec` field. `"hsbi/1"` is the
  version described here. A future `"hsbi/2"` is a separate document, and
  an implementation must refuse a scheme it does not know rather than
  attempt to interpret it.
- **Additive changes within v1** are allowed by §6: a tool that adds a
  field can rely on other tools carrying that field through. There is no
  minor-version bump for adding a field, because the unknown-field rule
  handles it.
- **Breaking changes within v1 are not allowed.** If the shape of `build`
  or an item entry needs to change in a way that is not backwards
  compatible, that is `hsbi/2`.

## 9. A worked example

A minimal but real Viking build. All item and ability ids used are the ones
this project's extraction produces from the client, so an implementer can
diff their tables against them.

```json
{
  "spec": "hsbi/1",
  "game": {
    "id": "herosiege",
    "patch": "2.7.0.0",
    "patchDate": "2026-05-22"
  },
  "producedBy": {
    "tool": "HeroSiege Build Engine",
    "toolVersion": "0.1.0",
    "generatedAt": "2026-08-19T00:00:00Z"
  },
  "build": {
    "identity": {
      "class": "Viking",
      "level": 100,
      "difficulty": "inferno",
      "title": "Frost Shieldbearer"
    },
    "attributes": {
      "strength": 200,
      "dexterity": 20,
      "vitality": 495,
      "energy": 20
    },
    "skills": {
      "unspentPoints": 0,
      "allocated": [
        { "id": "weaponMaster", "rank": 20 }
      ]
    },
    "trees": {
      "class": { "allocated": [] },
      "ether": { "allocated": [] },
      "incarnation": { "allocated": [] }
    },
    "gear": {
      "boots": {
        "id": "boots_aetherflow_treads",
        "rarity": "satanic",
        "name": "Aetherflow Treads",
        "level": 85,
        "sockets": [
          { "type": "rune", "id": "ol" },
          { "type": "empty" }
        ],
        "affixes": [
          { "id": "life", "value": 62 },
          { "id": "lightning_resistance", "value": 20 },
          { "id": "movement_speed", "value": 18 }
        ]
      },
      "helmet": null
    },
    "charms": [
      {
        "id": "charms_air_melon",
        "rarity": "angelic",
        "name": "Air Melon",
        "affixes": []
      }
    ],
    "relics": [],
    "notes": ""
  }
}
```

The item id `boots_aetherflow_treads` and the charm id `charms_air_melon`
are real keys from
[data/extracted/crosscheck_disagreements.json](../data/extracted/crosscheck_disagreements.json)
(lines 38 to 43 and 87 to 89, respectively). The ability id `weaponMaster`
is the client's own identifier for the Viking's first Shield Bearer
talent, from
[data/extracted/class-trees.json](../data/extracted/class-trees.json) at the
Viking spec named `Shield Bearer`.

## 10. Terms

- **Producer.** The tool exporting the document.
- **Consumer.** The tool importing the document.
- **Recognised field.** A field the consumer's implementation of this
  spec understands and models.
- **Unrecognised field.** A well-formed JSON key the consumer does not
  model. See §6 for handling.
- **Unresolvable entry.** A well-modelled reference (an `id`, a
  `sockets[].id`, an `affixes[].id`) that the consumer's data pack does
  not know. See §7.
- **Client identifier.** A string this project extracts from the shipped
  game client. Stable within a patch. Changes across patches are
  possible and are why the envelope carries `game.patch`.

## 11. What this v1 does not cover, restated

To close: v1 does not describe multiple builds in one document, compressed
transport, character progression history, ranges or affix pools (those live
in the data pack, not in the build), migration rules between game patches,
or per-tool feature flags. Each of those is a valid future topic and is
deliberately left out so v1 can ship narrow and correct.

## 12. Provenance of this document

- Written 2026-08-19 as this project's first draft of the interchange
  format.
- HSPlanner facts cited (lz-string share URL; features list) are from the
  HSPlanner public README and CHANGELOG, fetched 2026-08-19 from
  github.com/HeroSiegePlanner/HSPlanner. No claims are made in this
  document about HSPlanner internals beyond what those documents state.
- Item id cross-references are from
  [data/extracted/crosscheck_disagreements.json](../data/extracted/crosscheck_disagreements.json),
  produced by this project's own extraction pipeline on the client build
  dated 2026-05-22.
- Class and ability id references are from
  [data/extracted/class-trees.json](../data/extracted/class-trees.json),
  produced by the same pipeline on the same client build.
