# Chord DB :musical_note:

A Postgres database of chords, scales and modes, with views for working out which chords fit a key and a voice-leading engine for generating chord progressions.

It powers [modal.chordbuildr.com](https://modal.chordbuildr.com/).

<hr>
<h3>Setup</h3>

Execute ```docker compose up``` to create a postgres database named chordanalyzr on ```localhost:5432``` with user credentials from ```flyway/flyway.conf```. Flyway applies the schema, seed data and views on startup.

Schema and seed data live in `flyway/sql/versioned/`, and the views and functions in `flyway/sql/views/` (repeatable migrations, re-applied whenever they change).

<hr>
<h3>chord views</h3>

```chord_view``` displays all permutations of root note and chord type, with the notes comprising each chord aggregated in csv.

Sample usage:

```
SELECT cv.chord_name,
       cv.chord_note_names,
       cv.chord_notes
FROM chord_view cv
WHERE note_name = 'F' AND
      chord_type = '7sus4';
```
Returns:
```
"F7sus4",   "F, Bb, C, D#",   "6, 11, 13, 16"
```

<br>

To capture each note by row use ```chord_note_view```

<hr>
<h3>mode views</h3>

```mode_view``` displays all permutations of key and mode, with the notes comprising each scale aggregated in csv.

Sample usage:

```
SELECT mv.mode,
       mv.key_name,
       mv.mode_note_names,
       mv.mode_notes
FROM mode_view mv
WHERE key_name = 'A';
```
Returns:
```
"mode",              "key_name",   "mode_note_names",          "mode_notes"
Aeolian,             A,            "A, B, C, D, E, F, G",      "10, 12, 13, 15, 17, 18, 20"
Altered Scale,       A,            "A, Bb, C, Db, Eb, F, G",   "10, 11, 13, 14, 16, 18, 20"
Dorian,              A,            "A, B, C, D, E, F#, G",     "10, 12, 13, 15, 17, 19, 20"
Dorian b2,           A,            "A, Bb, C, D, E, F#, G",    "10, 11, 13, 15, 17, 19, 20"
Ionian,              A,            "A, B, C#, D, E, F#, G#",   "10, 12, 14, 15, 17, 19, 21"
Locrian,             A,            "A, Bb, C, D, Eb, F, G",    "10, 11, 13, 15, 16, 18, 20"
Locrian #2,          A,            "A, B, C, D, Eb, F, G",     "10, 12, 13, 15, 16, 18, 20"
Lydian,              A,            "A, B, C#, D#, E, F#, G#",  "10, 12, 14, 16, 17, 19, 21"
Lydian Augmented,    A,            "A, B, C#, D#, E#, F#, G#", "10, 12, 14, 16, 18, 19, 21"
Lydian Dominant,     A,            "A, B, C#, D#, E, F#, G",   "10, 12, 14, 16, 17, 19, 20"
Melodic Minor,       A,            "A, B, C, D, E, F#, G#",    "10, 12, 13, 15, 17, 19, 21"
Mixolydian,          A,            "A, B, C#, D, E, F#, G",    "10, 12, 14, 15, 17, 19, 20"
Mixolydian b6,       A,            "A, B, C#, D, E, F, G",     "10, 12, 14, 15, 17, 18, 20"
Phrygian,            A,            "A, Bb, C, D, E, F, G",     "10, 11, 13, 15, 17, 18, 20"
```
<br>

To capture each note by row use ```mode_note_view```

<hr>
<h3>mode chord views</h3>

```mode_scale_chord_relation_view``` displays all permutations of chord mode and key with any distinct notes listed and counted. This can be used for identifying chords that complement a given scale.

Sample usage:

The following query will return all chords that have a root note of A and whose comprising notes all fall within the C Ionian scale.

```
SELECT mscv.mode,
       mscv.key_name,
       mscv.chord_name,
       mscv.chord_notes,
       mscv.mode_chord_note_diff,
       mscv.mode_chord_note_diff_count
FROM mode_scale_chord_relation_view mscv
WHERE mscv.mode = 'Ionian' AND
      mscv.key_name = 'C' AND
      mscv.chord_note_name = 'A' AND
      mscv.mode_chord_note_diff_count = 0
```
Returns:
```
"mode"      "key_name"  "chord_name"    "chord_notes"   "mode_chord_note_diff"  "mode_chord_note_diff_count"
"Ionian"    "C"         "Am"            "{1,5,10}"      "{}"                     "0"
"Ionian"    "C"         "Am7"           "{1,5,8,10}"    "{}"                     "0"
"Ionian"    "C"         "Am9"           "{1,5,8,10}"    "{}"                     "0"
"Ionian"    "C"         "A5"            "{5,10}"        "{}"                     "0"
"Ionian"    "C"         "Asus2"         "{5,10,12}"     "{}"                     "0"
"Ionian"    "C"         "Asus4"         "{3,5,10}"      "{}"                     "0"
"Ionian"    "C"         "Am11"          "{1,3,5,8,10}"  "{}"                     "0"
"Ionian"    "C"         "A7sus4"        "{3,5,8,10}"    "{}"                     "0"
"Ionian"    "C"         "Am7#5"         "{1,6,8,10}"    "{}"                     "0"
"Ionian"    "C"         "A9sus4"        "{3,5,8,10}"    "{}"                     "0"
```

<br>

<hr>
<h3>voice leading & harmonic pathfinding</h3>

The same chord/scale relations are also exposed as a **weighted graph**. Each in-scale chord is a node, and every pair of chords is joined by an edge weighted by *voice-leading distance*: the minimal total semitone motion needed to move one chord's notes to another's (common tones cost nothing, so lower-weight moves are smoother). This lets the database work as a live progression engine for questions too combinatorially large to precompute into static data.

Generate the smoothest four-chord progression in C Ionian, starting on `Cmaj7`:

```
SELECT * FROM fn_smooth_progression('Ionian', 'C', 'Cmaj7', 4);
```
Returns (columns `step, chord, vl_from_prev, total_cost`), e.g.:
```
 step | chord     | vl_from_prev | total_cost
------+-----------+--------------+------------
    1 | Cmaj7     |            0 |          4
    2 | Am add(2) |            2 |          4
    3 | Esus4     |            1 |          4
    4 | Fb5       |            1 |          4
```

Supporting building blocks: `fn_voice_leading_distance(int[], int[])` (metric between two pitch-class sets), `mode_key_chord_view` (graph nodes, the diatonic chords of a key/mode), and `mode_key_chord_edge_view` (weighted edges). See [`flyway/sql/views/R__voice_leading.sql`](flyway/sql/views/R__voice_leading.sql) for the implementation.

<hr>
<h3>tests</h3>

[`flyway/tests/voice_leading_test.sql`](flyway/tests/voice_leading_test.sql) holds regression tests for the voice-leading functions. Run it against a migrated database:

```
docker compose exec -T postgres psql -U postgres -d chordanalyzr \
    -v ON_ERROR_STOP=1 -f - < flyway/tests/voice_leading_test.sql
```

<hr>
<h3>static data export</h3>

`data-generator/` exports the chord and mode views (including `mode_scale_chord_relation_view`) to JSON files, so a client can use the data without a live database. With the database running:

```
docker compose --profile export up data-generator
```

The files are written to `./output`.
