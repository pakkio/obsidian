---
tags:
  - craftsim
  - ossl
  - lsl
  - examples
  - osdb
  - osjev
---

# OSSL Extension Examples - Complete Coverage - osDb examples

> Part of [[OSSL Extension Examples - Complete Coverage]]. Sections: lifecycle, queries, import/export, backup and restore.

## 1. osDb - lifecycle, queries, import/export

### 1.1 Players table with an index, an ordered report and housekeeping
Covers `osDbList`, `osDbAttach`, `osDbCreate`, `osDbRowCount`, `osDbInsert`, `osDbCreateIndex`, `osDbFindByIndex`, `osDbFindOrderBy`, `osDbFind`, `osDbDeleteWhere`, `osDbFlush`, `osDbDropIndex`. Secondary indexes are **not persisted**, so build them on every start; `osDbFind` does a full scan and does not use them; `osDbFindByIndex` returns row **ids**, not rows.

```lsl
key db;

string rowOf(string id, string name, string zone, integer score)
{
    return llList2Json(JSON_OBJECT, ["id" => id, "name" => name, "zone" => zone, "score" => score]);
}

default
{
    state_entry()
    {
        // osDbList is a flat list, stride 4: [id, nick, refcount, attachedHostsCsv, ...]
        list dbs = osDbList();
        integer i;
        for (i = 0; i < llGetListLength(dbs); i += 4)
            if (llList2String(dbs, i + 1) == "players")
                db = osDbAttach((key)llList2String(dbs, i));
        if (db == NULL_KEY || db == (key)"")
            db = osDbCreate("players", "players_data");     // nick, notecard name (both required)

        if (osDbRowCount(db) == 0)
        {
            osDbInsert(db, rowOf("p1", "Ada",   "north", 120));
            osDbInsert(db, rowOf("p2", "Bruno", "north",  80));
            osDbInsert(db, rowOf("p3", "Chen",  "south", 200));
            osDbInsert(db, rowOf("p4", "Dora",  "south",  15));
        }

        // Secondary indexes are not persisted: build them on every start.
        osDbCreateIndex(db, "zone");

        // osDbFindByIndex returns row ids, not rows.
        list north = osDbFindByIndex(db, "zone", "north");
        llOwnerSay("north rows: " + (string)llGetListLength(north));

        // top 3 scores >= 50, descending; each element is the row as JSON text
        list top = osDbFindOrderBy(db, "score", ">=", "50", "score", FALSE, 3);
        for (i = 0; i < llGetListLength(top); ++i)
            llOwnerSay("#" + (string)(i + 1) + " " + llList2String(top, i));

        list low = osDbFind(db, "score", "<", "50");       // full scan: ignores secondary indexes
        llOwnerSay("low scorers: " + (string)llGetListLength(low));

        integer gone = osDbDeleteWhere(db, "score", "<", "50");
        llOwnerSay("deleted " + (string)gone + ", remaining " + (string)osDbRowCount(db));
        osDbFlush(db);
        osDbDropIndex(db, "zone");
    }
}
```

**Output** (run 2026-10-17):

```text
north rows: 2
#1 {"id":"p3","name":"Chen","zone":"south","score":200}
#2 {"id":"p1","name":"Ada","zone":"north","score":120}
#3 {"id":"p2","name":"Bruno","zone":"north","score":80}
low scorers: 1
deleted 1, remaining 3
```

This is the **first** run. The table `players` is registered in the region and keeps its rows, so a second run finds the three rows left (Dora, score 15, was deleted) and prints `low scorers: 0`, `deleted 0, remaining 3`: the script is safe to run again, and the inserts only happen on an empty table.

### 1.2 Backup and restore: `osDbExport`, `osDbImport`, `osDbDetach`, `osDbDrop`
`osDbDetach` persists and frees the in-memory store when the last attached prim lets go; `osDbDrop` removes the registry entry (the notecard stays). Export returns `1` here, not a row count; import returns the number of rows appended.

```lsl
key live;
key scratch;

default
{
    state_entry()
    {
        live = osDbCreate("orders", "orders_data");
        integer i;
        for (i = 0; i < 20; ++i)
            osDbInsert(live, llList2Json(JSON_OBJECT, ["id" => "o" + (string)i, "qty" => i % 5, "sku" => "S" + (string)(i % 3)]));

        // Snapshots go to NEW notecards in THIS prim (an existing one with the same name is overwritten).
        integer a = osDbExport(live, "orders-backup.csv",   "csv");
        integer b = osDbExport(live, "orders-backup.jsonl", "jsonl");
        llOwnerSay("exported csv=" + (string)a + " jsonl=" + (string)b);

        // Restore into a second store. Import APPENDS (it never clears), and CSV values come back as strings.
        scratch = osDbCreate("orders_restore", "orders_restore_data");
        integer n = osDbImport(scratch, "orders-backup.jsonl", "jsonl");
        llOwnerSay("imported " + (string)n + ", rows now " + (string)osDbRowCount(scratch));

        // Detach persists and releases rows from memory when the last prim lets go;
        // Drop removes the registry entry (the notecard itself is left alone).
        osDbDetach(scratch);
        osDbDrop(scratch);
        osDbDetach(live);
        llOwnerSay("done");
    }
}
```

**Output** (run 2026-10-17):

```text
exported csv=1 jsonl=1
imported 20, rows now 20
done
```

---

---
Back to [[OSSL Extension Examples - Complete Coverage]].
