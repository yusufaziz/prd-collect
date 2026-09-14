
## 9.4 Search Scoped by Define

```bash
python code-search.py --model="MODEL-ABC" \
    --content="calibrate" --define=FEATURE_A
```

## 9.5 Force Rebuild the Index

```bash
python code-search.py --index="build/MODEL-ABC" --rebuild
```

## 9.6 Search With Auto-Refresh

Just run a search. If any source, `map.txt`, `.axf`, or `compile_commands.json`
changed since the last index, the model is re-indexed automatically before the
search runs.

---

# 10. Extension Points

| Feature | How to add |
|---|---|
| **Vector search** | Add `embeddings(fid INTEGER, vec BLOB)` table. Replace `do_search` body with cosine top-K; keep the `built=1` filter. |
| **Preprocessed bodies** | Add `preprocessed_body TEXT` column. Populate by running `armcc -E` on each TU with flags from `compile_commands.json`, cached by `sha1(abs_path + tu_defines + header mtimes)`. |
| **Include graph** | Add `includes(file_id INTEGER, header_rel TEXT)` table populated from `*.d` / `.ninja_deps`. Invert for "who includes this header". |
| **Reachability from entry point** | After indexing, BFS from `Reset_Handler`/`main` over the `calls` table. Store `reachable INTEGER` on `functions`. |
| **Recent-change ranking** | Parse `.ninja_log` for per-object timestamps; add `built_at INTEGER` column. Boost recent hits. |
| **Doxygen enrichment** | Parse Doxygen XML; join on `(rel_path, start_line)` to fill a `doc_comment` column. |
| **Instruction-level context** | Emit `armcc -S` per TU; join assembly ranges onto function rows. Only worth it for hot paths. |
| **Call graph precision** | Replace name-based call edges with DWARF `DW_AT_abstract_origin` resolution, or disassembly-based `BL`/`BLX` parsing. |

## Notes on the `map.txt` Parser

The regex in `parse_map()` is intentionally loose to handle common armlink
variations. If your project uses a specific armlink version whose map format
differs, tighten the two patterns:

- `removed_sec` — matches `.text.<symbol_name> 0x...` lines under the
  "Removed Sections" / "Discarded input sections" heading.
- `symbol_re` — matches `    <symbol>  0x<hex>` lines in the Symbol Table.

Test on a real `map.txt` first with a one-liner:

```python
from code_search import parse_map
from pathlib import Path
linked, discarded = parse_map(Path("build/MODEL-ABC/map.txt"))
print(len(linked), "linked,", len(discarded), "discarded")
```

If the counts look wrong, paste the first 30 lines of the Symbol Table section
and adjust the regexes.

---

*End of document.*