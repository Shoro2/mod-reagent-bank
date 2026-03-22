# CLAUDE.md — mod-reagent-bank

AzerothCore-Modul: Reagenzienbank für WoW 3.3.5a (WotLK). Spieler können über einen NPC alle stapelbaren Handwerksmaterialien (Trade Goods, Gems) einlagern und nach Subklasse sortiert wieder abholen.

## Projekt-Kontext

Dieses Modul gehört zum Custom-WoW-Server-Projekt (AzerothCore-basiert). Siehe [share-public CLAUDE.md](https://github.com/Shoro2/share-public/blob/master/CLAUDE.md) für die Gesamtarchitektur.

## Repository-Struktur

```
mod-reagent-bank/
├── CLAUDE.md
├── LICENSE
├── include.sh                          # Build-System: SQL-Pfade für db_assembler.sh
├── conf/
│   ├── conf.sh.dist                    # SQL-Pfade für Auth/Characters/World DB
│   └── reagent_bank.conf.dist          # Konfiguration (ReagentBank.Enable = 0|1)
├── data/
│   └── sql/
│       ├── db-characters/base/
│       │   └── create_table.sql        # Erstellt custom_reagent_bank Tabelle
│       └── db-world/base/
│           └── reagent_bank_NPC.sql    # Spawnt NPC (Entry 290011, "Ling")
└── src/
    ├── ReagentBank.h                   # Header: Konstanten, Enums (MAX_OPTIONS, MAX_PAGE_NUMBER)
    ├── ReagentBank.cpp                 # Hauptlogik: CreatureScript "npc_reagent_banker"
    └── ReagentBank_loader.cpp          # Script-Registrierung: Addmod_reagent_bankScripts()
```

## Architektur

### Typ: CreatureScript (NPC + Gossip-Menü)

Das Modul verwendet das AzerothCore Gossip-System — kein AIO, keine Client-Addons.

```
Spieler → NPC "Ling" (Entry 290011) → Gossip-Menü
    ├── Kategorien (15 Subklassen: Parts, Explosives, Cloth, Herb, ...)
    │   └── Items anzeigen (paginiert, MAX_OPTIONS=23 pro Seite)
    │       └── Klick → Withdraw (1 Stack)
    └── "Deposit All Reagents" → Alle Trade Goods + Gems aus Inventar einlagern
```

### Datenbank

**Tabelle:** `custom_reagent_bank` (acore_characters)

| Spalte | Typ | Beschreibung |
|--------|-----|-------------|
| `character_id` | int(11) | PK — Character GUID |
| `item_entry` | int(11) | PK — Item Template ID |
| `item_subclass` | int(11) | Subklasse für Kategorisierung |
| `amount` | int(11) | Gelagerte Menge |

### Klassen-Übersicht

**`npc_reagent_banker`** (CreatureScript) — Einzige Klasse, alles in `ReagentBank.cpp`:

| Methode | Funktion |
|---------|----------|
| `OnGossipHello` | Zeigt Hauptmenü mit 15 Kategorien + "Deposit All" |
| `OnGossipSelect` | Router: Withdraw (ID > 700), Deposit All, Kategorie, Hauptmenü |
| `ShowReagentItems` | Async DB-Query → paginierte Item-Liste (23 pro Seite) |
| `DepositAllReagents` | Scannt Inventar + Taschen → `REPLACE INTO` für alle Trade Goods/Gems |
| `WithdrawItem` | Sync DB-Query → gibt 1 Stack zurück (oder alles wenn ≤ Stackgröße) |
| `GetItemLink` | Erzeugt WoW Item-Link mit Farbe und Name (lokalisiert) |
| `GetItemIcon` | Erzeugt Icon-String aus DisplayInfo |
| `UpdateItemCount` | Hilfs-Methode: Zählt Items, sortiert Gems zu Jewelcrafting, löscht Original |

### Wichtige Konstanten (ReagentBank.h)

| Konstante | Wert | Bedeutung |
|-----------|------|-----------|
| `MAX_OPTIONS` | 23 | Items pro Gossip-Seite |
| `MAX_PAGE_NUMBER` | 700 | Werte > 700 werden als Item-IDs interpretiert (Withdraw) |
| `NPC_TEXT_ID` | 4259 | Standard-NPC-Text im Gossip-Fenster |
| `DEPOSIT_ALL_REAGENTS` | 16 | Gossip-Action: Alle einlagern |
| `MAIN_MENU` | 17 | Gossip-Action: Zurück zum Hauptmenü |

### Akzeptierte Item-Klassen

Nur Items mit `MaxStackSize > 1` und einer dieser Klassen:
- `ITEM_CLASS_TRADE_GOODS` (Subklassen: Parts, Explosives, Devices, Cloth, Leather, Metal&Stone, Meat, Herb, Elemental, Enchanting, Nether Material, Other, Armor Vellum, Weapon Vellum)
- `ITEM_CLASS_GEM` (wird intern als `ITEM_SUBCLASS_JEWELCRAFTING` kategorisiert)

### NPC

| Eigenschaft | Wert |
|-------------|------|
| Entry | 290011 |
| Name | Ling |
| Subname | Reagent Banker |
| ScriptName | `npc_reagent_banker` |
| DisplayID | 15965 |
| Faction | 35 (Freundlich) |

## Code-Konventionen

- C++17 (AzerothCore-Standard)
- Sync-Query für Withdraw (`CharacterDatabase.Query`), Async für Deposit + ShowItems (`AsyncQuery` + `WithCallback`)
- Gossip-Paginierung über `gossipPageNumber` Parameter (Seite 0, 1, 2, ...)
- Transactions für Batch-Inserts (`BeginTransaction` + `CommitTransaction`)
- `REPLACE INTO` statt INSERT+UPDATE für Upsert-Logik

## Build & Config

1. Modul in `modules/` Verzeichnis des AzerothCore-Builds platzieren
2. CMake konfigurieren und bauen (Modul wird automatisch erkannt)
3. SQL aus `data/sql/` wird über `db_assembler.sh` / `include.sh` eingespielt
4. `ReagentBank.Enable = 1` in worldserver.conf setzen
