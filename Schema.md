# Duneri Campaign Wiki — Database Schema

This document captures the finalized data model for the Duneri lore/character web app, along with the reasoning behind key decisions. Intended as both a working reference and a record of design tradeoffs for later writeups (e.g. resume/interview talking points).

## Design Principles

A few consistent rules were applied across the schema:

1. **Structured vs. RAG-only**: An entity gets a real table only if the app needs to *query, filter, or relate to it*. Content that's purely narrative/reference (NPCs, Locations, mechanics docs, onboarding docs) lives in `LoreEntry` instead, and is handled by the RAG pipeline rather than relational queries.
2. **Fixed shape vs. JSON**: A field becomes a JSON column instead of relational columns/tables when its shape is *unpredictable* per row (e.g. a class has a variable number of abilities). Fields with a consistent, fixed shape across every row (e.g. every `Spell` always has exactly one casting time, one range, etc.) stay as real columns, since that keeps them queryable.
3. **Infobox + article model** (borrowed from Wikipedia): every structured entity that the DM might want to present creatively also gets a `content` field — a freeform, presentation-only layer (rich text / block JSON) separate from the mechanical columns. Structured columns remain the source of truth for anything the app calculates with; `content` never duplicates or overrides them.

## Tables

### User
| Column | Type | Notes |
|---|---|---|
| id | PK | |
| auth_provider_id | string | External ID from managed auth provider (Clerk/Auth0) |
| role | enum | DM / Player |
| display_name | string | |
| created_at | timestamp | |

Auth is handled by a managed provider rather than custom JWT — deliberate choice to avoid owning password/session security on a learning project with real users. Additional profile data (email, avatar) is pulled from the provider's API rather than duplicated here unless a specific need arises.

### Race
| Column | Type | Notes |
|---|---|---|
| id | PK | |
| category | string | Grouping label only (e.g. "Wayfarers", "Undead") — not a separate table, no hierarchy |
| name | string | |
| rarity | string | Flavor only, no mechanical effect |
| description | text | |
| age | text | |
| appearance | text | |
| alignment | text | |
| size | text | |
| language | text | |
| location | text | |
| stat_modifiers | JSON | Stats, movement, sight, immunities, resistances, vulnerabilities, vitals, magic eligibility |
| traits | JSON | Variable-length list of named race traits |
| revelations | JSON | Variable-length list of named perks; each entry may include a `prerequisites` array (names of other revelations in the same race) — no FK needed since prerequisites never cross races |
| content | JSON/rich text | DM's freeform presentation layer |

### Item
| Column | Type | Notes |
|---|---|---|
| id | PK | |
| name | string | |
| type | string | weapon / armor / firearm / consumable / etc. |
| rarity | string | Flavor only, DM controls loot manually |
| weight | number | |
| price | number | |
| description | text | |
| properties | JSON | Type-varying mechanical fields (damage, range, capacity, armor_value, slot, attunement, effect, duration, etc.), plus any item-granted abilities |
| content | JSON/rich text | DM's freeform presentation layer |

Field set varies significantly by item type (firearms vs. melee weapons vs. armor vs. consumables share almost no attributes), which is why type-specific fields live in `properties` rather than as fixed columns.

### Class
| Column | Type | Notes |
|---|---|---|
| id | PK | |
| name | string | |
| description | text | |
| hp_base | number | |
| hp_per_level | number | |
| mana_base | number | |
| mana_per_level | number | |
| abilities | JSON | Base Calling abilities (not tied to any module) |
| proficiencies | JSON | Weapons, armor, tools, substats_pool, saving_rolls — substat pick count is always 2, confirmed across all classes, so not stored separately |
| spellcasting_modifier | string | e.g. "Strength" |
| content | JSON/rich text | DM's freeform presentation layer |

Related to `SpellCategory` via the `class_spell_categories` join table.

### ClassModule
| Column | Type | Notes |
|---|---|---|
| id | PK | |
| class_id | FK → Class | |
| tier | int | 2, 3, or 4 |
| name | string | |
| description | text | |
| abilities | JSON | Variable-length list of named abilities |

Modules within the same tier are independent choices (no prerequisite chains between them, confirmed against all classes in the source material). Tier counts and option counts per tier vary by class; schema doesn't assume uniformity.

### SpellCategory
| Column | Type | Notes |
|---|---|---|
| id | PK | |
| name | string | |
| description | text | |

### Spell
| Column | Type | Notes |
|---|---|---|
| id | PK | |
| spell_category_id | FK → SpellCategory | Each spell belongs to exactly one category |
| name | string | |
| tier | int | |
| casting_time | string | |
| range | string | |
| target | string | |
| components | string | |
| duration | string | |
| uses | string | |
| description | text | |

Kept as fixed relational columns (not JSON) since every spell has exactly one instance of each field — no variability to accommodate, unlike ClassModule abilities.

### Character
| Column | Type | Notes |
|---|---|---|
| id | PK | |
| owner_id | FK → User | |
| race_id | FK → Race | |
| class_id | FK → Class | |
| level | int | Static reference only in v1 — no personalized "what have I unlocked" view yet (stretch feature) |
| stats/inventory | JSON | |
| substats_picked | JSON/array | The 2 substats chosen from the Class's substat pool |
| content | JSON/rich text | Player's freeform notes/backstory layer |

Related to `Spell` via the `character_spells` join table.

### LoreEntry
| Column | Type | Notes |
|---|---|---|
| id | PK | |
| title | string | |
| source_type | string | Free-text label (e.g. "mechanics", "history", "character_creation") — not a fixed enum |
| body | text | Raw source text — this is what the Week 3 RAG/chunking pipeline reads from |
| content | JSON/rich text | DM's freeform display layer — presentation only, never touched by the chunking pipeline, kept separate from `body` so display edits can't corrupt embeddings |
| created_at | timestamp | |
| updated_at | timestamp | Useful for detecting when re-embedding is needed |

Covers NPCs, Locations, world history/lore, game mechanics reference, and the character creation guide — none of these need structured/queryable fields, so they're handled entirely through RAG rather than relational tables. One row per whole document/entry; chunking happens downstream in Week 3, not at the schema level.

## Join Tables

- **class_spell_categories** — many-to-many between `Class` and `SpellCategory` (a class can access multiple spell categories; a category can be shared by multiple classes)
- **character_spells** — many-to-many between `Character` and `Spell` (a character can know multiple spells)

## Open / Deferred

- Personalized character view (showing exactly which abilities a character has unlocked based on level) — deferred as a stretch feature; v1 treats Class/ClassModule as a static reference doc.
- Final field-level polish on `LoreEntry.source_type` values as more source documents are reviewed.
