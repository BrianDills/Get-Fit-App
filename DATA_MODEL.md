# Iron Log data model

## Versioning

Iron Log currently uses **data model version 1** and **backup format version 2**.

These are deliberately separate versions:

- `dataModelVersion` describes the meaning and shape of workout records.
- `version` describes the downloadable backup envelope and its migration rules.

A backup envelope may change without forcing a rewrite of every locally stored workout. The app continues to read its existing localStorage keys so this foundation change does not rewrite or discard the user's current program or sessions.

## Current local records (model v1)

### Program

The active program is stored under `program-data` as an array of days:

- Day: `id`, `num`, `title`, `subtitle`, `exercises[]`
- Exercise: `id`, `name`, `notes`, `superset`, `userNote`, `sets[]`
- Planned set: `id`, `type` (for example `warmup` or `normal`), `target`

### Completed strength session

Each day has a `history:<dayId>` key containing an array of sessions:

- Session: `date` (ISO-8601 timestamp), `exercises`
- Logged exercise: `name`, `sets`
- Logged set: `label`, `weight` (number or null), `reps` (number or null)

The older `latest:<dayId>` shape is migrated lazily by the existing compatibility function. It is not silently reinterpreted as a new record type.

## Resistance and equipment semantics

The existing `weight` field is a legacy user-entered numeric value. It does **not** reliably identify:

- pounds versus kilograms;
- per-hand versus combined load;
- a free-weight load versus a machine's displayed resistance setting;
- a Bowflex Xceed dial/setting versus a claimed equivalent free-weight load.

Therefore model v1 preserves that number exactly and does not infer a unit or convert it. In particular, Bowflex Xceed settings must remain distinguishable from dumbbell/barbell loads. Future records should use an explicit resistance descriptor rather than overloading a bare number:

```json
{
  "value": 5,
  "unit": null,
  "method": "machine-setting",
  "equipment": {
    "type": "home-gym",
    "make": "Bowflex",
    "model": "Xceed"
  },
  "basis": "displayed-setting"
}
```

For external loads, the descriptor should explicitly state the unit and basis (for example, pounds and per-hand). Unknown values stay unknown; migrations must never invent equivalencies.

## Backup format v2

A v2 backup keeps the existing portable `program`, `history`, and `legacyLatest` payloads and adds:

- `dataModelVersion: 1`
- `model.resistanceValues`: states that resistance numbers are user-entered and not automatically interpreted.
- `model.legacyCompatibility`: describes preservation of older records.

The app accepts v1 backups and migrates their **envelope metadata only** to v2. Program and session records are copied unchanged. A v1 restore is still validated before the user is asked to confirm it, and the restore process retains its rollback behavior if a storage write fails.

## Migration policy

1. Never overwrite local data during startup merely to upgrade a schema.
2. Validate a backup before writing anything.
3. Migrate a copy of imported data; do not mutate the source object or file.
4. Preserve unknown fields so newer records can survive an older app's round-trip where possible.
5. Do not infer resistance units, machine equivalencies, or per-hand/total load.
6. Add future fields as optional fields first; only make them required after a versioned migration exists.
7. Keep backup-envelope migrations separate from local-storage migrations.

## Future record extensions

The model is intended to support, without conflating them:

- strength sets (reps, resistance descriptor, optional RPE and notes);
- timed work (duration in seconds);
- distance-based work (distance in a canonical unit plus display preference);
- cardio sessions independent of a strength-program day;
- bodyweight and tape measurements;
- progress-photo references with local/private handling.

These are extension targets, not claims that all are already implemented in the current UI.
