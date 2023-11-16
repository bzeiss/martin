# Copying, Diffing, and Patching MBTiles

## `mbtiles copy`
Copy command copies an mbtiles file, optionally filtering its content by zoom levels.

```shell
mbtiles copy src_file.mbtiles dst_file.mbtiles \
        --min-zoom 0 --max-zoom 10
```

This command can also be used to generate files of different [supported schema](##supported-schema).

```shell
mbtiles copy normalized.mbtiles dst.mbtiles \
         --dst-mbttype flat-with-hash
```

## `mbtiles copy --diff-with-file`
Copy command can also be used to compare two mbtiles files and generate a delta (diff) file. The diff file can be applied to the `src_file.mbtiles` elsewhere, to avoid copying/transmitting the entire modified dataset.  The delta file will contain all tiles that are different between the two files (modifications, insertions, and deletions as `NULL` values), for both the tile and metadata tables.

There is one exception: `agg_tiles_hash` metadata value will be renamed to `agg_tiles_hash_in_diff`, and a new `agg_tiles_hash` will be generated for the diff file itself. This is done to avoid confusion when applying the diff file to the original file, as the `agg_tiles_hash` value will be different after the diff is applied. The `apply-diff` command will automatically rename the `agg_tiles_hash_in_diff` value back to `agg_tiles_hash` when applying the diff.

```shell
mbtiles copy src_file.mbtiles diff_file.mbtiles \
         --diff-with-file modified_file.mbtiles
```

## `mbtiles copy --apply-patch`

Copy a source file to destination while also applying the diff file generated by `copy --diff-with-file` command above to the destination mbtiles file. This allows safer application of the diff file, as the source file is not modified.

```shell
mbtiles copy src_file.mbtiles dst_file.mbtiles \
        --apply-patch diff_file.mbtiles
```

## `mbtiles apply-patch`

Apply the diff file generated from `copy` command above to an mbtiles file. The diff file can be applied to the `src_file.mbtiles` elsewhere, to avoid copying/transmitting the entire modified dataset.

Note that the `agg_tiles_hash_in_diff` metadata value will be renamed to `agg_tiles_hash` when applying the diff. This is done to avoid confusion when applying the diff file to the original file, as the `agg_tiles_hash` value will be different after the diff is applied.

```shell
mbtiles apply_diff src_file.mbtiles diff_file.mbtiles
```

#### Applying diff with SQLite
Another way to apply the diff is to use the `sqlite3` command line tool directly. This SQL will delete all tiles from `src_file.mbtiles` that are set to `NULL` in `diff_file.mbtiles`, and then insert or update all new tiles from `diff_file.mbtiles` into `src_file.mbtiles`, where both files are of `flat` type. The name of the diff file is passed as a query parameter to the sqlite3 command line tool, and then used in the SQL statements.  Note that this does not update the `agg_tiles_hash` metadata value, so it will be incorrect after the diff is applied.

```shell
sqlite3 src_file.mbtiles \
  -bail \
  -cmd ".parameter set @diffDbFilename diff_file.mbtiles" \
  "ATTACH DATABASE @diffDbFilename AS diffDb;" \
  "DELETE FROM tiles WHERE (zoom_level, tile_column, tile_row) IN (SELECT zoom_level, tile_column, tile_row FROM diffDb.tiles WHERE tile_data ISNULL);" \
  "INSERT OR REPLACE INTO tiles (zoom_level, tile_column, tile_row, tile_data) SELECT * FROM diffDb.tiles WHERE tile_data NOTNULL;"
```