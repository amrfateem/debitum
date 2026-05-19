# AGENTS.md

## Project Summary

Debitum is a single-module Android app for tracking personal IOUs. It stores all data locally on-device and supports:

- monetary transactions
- item lending/return tracking
- person records with optional linked contacts
- image attachments on transactions
- backup and restore of database, preferences, and images

The codebase is compact and mostly Java-based. There is one Android app module: `app`.

## Stack And Build

- Language: Java
- Build system: Gradle
- Android Gradle Plugin: `7.2.1`
- Compile SDK / Target SDK: `32`
- Min SDK: `24`
- App id: `org.ebur.debitum`
- Current version in repo: `1.7.2` (`versionCode 23`)
- Main libraries:
  - AndroidX AppCompat / Preference
  - Lifecycle ViewModel + LiveData
  - Room
  - Navigation
  - Material Components
  - RecyclerView Selection
  - WebKit

Useful commands from repo root on Windows:

```powershell
.\gradlew.bat test
.\gradlew.bat assembleDebug
.\gradlew.bat assembleRelease
.\gradlew.bat checkLicenses
.\gradlew.bat updateLicenses
.\gradlew.bat generateLicensePage
```

## Repo Map

### Root

- `README.md`: product overview, features, screenshots.
- `FAQ.md`: user-facing behavior details.
- `CHANGELOG.md`: release history.
- `TODO.md`: backlog and release checklist.
- `TRANSLATION.md`: translation workflow and locale caveats.
- `MIGRATION.md`: manual migration notes from U.O.me.
- `build.gradle`: top-level repositories, AGP version, shared dependency versions.
- `settings.gradle`: includes only `:app`.
- `gradle.properties`: AndroidX and Gradle JVM settings.
- `.github/workflows/codeql-analysis.yml`: CodeQL only. No broader CI workflow is present.
- `artwork/`: editable source art for launcher and status icons.
- `fastlane/metadata/android/`: store listing text, localized changelogs, screenshots, app icon metadata.
- `gradle/wrapper/`: wrapper JAR and properties.
- `.idea/`: IDE project files. Usually not part of feature work unless the user explicitly wants IDE config changes.

### `app/`

- `build.gradle`: Android module config, dependencies, version, Room schema export path.
- `licenses.yml`: license-tools input.
- `proguard-rules.pro`: ProGuard config.
- `schemas/org.ebur.debitum.database.AppDatabase/`: Room schema snapshots for versions `1` through `6`. Update this when changing the Room schema.
- `src/main/AndroidManifest.xml`: `READ_CONTACTS`, `MainActivity`, `FileProvider`.
- `src/main/assets/`:
  - `licenses.html`, `changelog.html`
  - legacy localized guide HTML files
- `src/main/java/org/ebur/debitum/`:
  - `database/`: Room entities, DAOs, repositories, migrations, relation models
  - `viewModel/`: screen and shared UI state
  - `ui/`: activity, settings, dialogs
  - `ui/list/`: list fragments, adapters, selection/action mode logic
  - `ui/edit_transaction/`: transaction editor dialog and image list
  - `util/`: formatting, color, file, animation helpers
- `src/main/res/`:
  - `layout/`: activity, fragments, rows, headers
  - `navigation/nav_graph_main.xml`: app destinations and global actions
  - `menu/`: toolbar, bottom nav, contextual action mode menus
  - `xml/root_preferences.xml`: settings definition
  - `values*`: strings, colors, styles, arrays, locale-specific resources
  - `drawable*`, `mipmap*`, `font/`
- `src/test/`: local unit tests for core model and utility behavior
- `src/androidTest/`: minimal instrumentation example only

## Architecture

The app uses a lightweight MVVM-ish structure:

- `ui/*` renders screens and dialogs, handles view wiring and navigation.
- `viewModel/*` holds screen state and shared state between screens.
- `database/*` wraps Room access through DAOs and simple repositories.
- `util/*` contains reusable helper logic.

There is no DI framework. Repositories are created directly inside `AndroidViewModel` classes using the `Application`.

### Main Navigation

Entry point: `app/src/main/java/org/ebur/debitum/ui/MainActivity.java`

Main destinations from `nav_graph_main.xml`:

- `people_dest` -> `PersonSumListFragment`
- `money_dest` -> `TransactionListFragment`
- `item_dest` -> `ItemTransactionListFragment`
- `settings_dest` -> `SettingsFragment`
- dialog destinations for editing person, editing transaction, licenses, changelog

`MainActivity` owns:

- toolbar wiring with NavigationUI
- bottom navigation
- shared FAB behavior
- automatic "what's new" popup based on `BuildConfig.VERSION_CODE` and `SettingsFragment.PREF_KEY_CHANGELOG`

## Domain Model And Invariants

### `Person`

File: `app/src/main/java/org/ebur/debitum/database/Person.java`

- Stored in table `person`
- Fields: `idPerson`, `name`, `note`, `linkedContactUri`
- `colorIndex` is not stored in DB; it is derived from the name via MD5 hash for avatar coloring
- `Person(-1)` is used as a placeholder for "new person" in editor flows

### `Transaction`

File: `app/src/main/java/org/ebur/debitum/database/Transaction.java`

- Stored in table `txn`
- Fields: `idTransaction`, `amount`, `idPerson`, `description`, `isMonetary`, `timestamp`, `timestampReturned`, `hasImages`
- Sign convention is important:
  - monetary: negative means user gave money, positive means user received money
  - items: signed quantity is still used, but lists often display absolute values
- Monetary amounts are stored as integers in minor units according to the current decimals setting
- `timestampReturned` is meaningful for item transactions only
- `hasImages` is a persisted optimization for list icon display

### Relations

- `TransactionWithPerson`: embeds `Transaction` plus related `Person`
- `PersonWithTransactions`: embeds `Person` plus their transactions

These relation classes also contain aggregation helpers used by list screens.

## Database

Core file: `app/src/main/java/org/ebur/debitum/database/AppDatabase.java`

- Room DB name: `transaction_database`
- Current schema version: `6`
- Entities: `Transaction`, `Person`, `Image`
- Journal mode is `TRUNCATE` to simplify export/backup
- Global executor: `databaseTaskExecutor` with 4 threads

### Existing migrations

- `1 -> 2`: add `person.note`
- `2 -> 3`: add `txn.timestamp_returned`
- `3 -> 4`: add `person.linked_contact_uri`
- `4 -> 5`: create `image`
- `5 -> 6`: add `txn.has_images`

If you change schema:

1. update entity/DAO code
2. add the next migration in `AppDatabase`
3. bump `version`
4. ensure Room exports a new schema JSON into `app/schemas/...`
5. verify backup/restore still works with the new fields

### DAO responsibilities

- `PersonDao`: fetch persons, existence checks, delete person plus their transactions and image links
- `TransactionDao`: fetch monetary or item transactions, fetch grouped people summaries, update/insert/delete, bulk decimal shifting
- `ImageDao`: track image filenames linked to transactions, remove broken links

### Repository layer

Repositories are thin wrappers over DAOs:

- `PersonRepository`
- `TransactionRepository`
- `ImageRepository`

They use `Future#get()` for some synchronous reads. Be careful when adding more blocking calls from UI code.

## Shared ViewModels

Important shared state is held at activity scope:

- `PersonFilterViewModel`: current person filter for money/items lists
- `ItemReturnedFilterViewModel`: current items filter mode
- `ListOrderViewModel`: ordering for people summary list
- `ContactsHelper`: permission state, contact name/photo lookup cache, avatar generation

When behavior appears to "persist across tabs", check whether the ViewModel is scoped to the activity.

## UI Areas

### People List

Key files:

- `ui/list/PersonSumListFragment.java`
- `ui/list/PersonSumListAdapter.java`
- `ui/list/PersonSumListViewHolder.java`
- `viewModel/PersonSumListViewModel.java`

Behavior:

- shows one row per person with at least one transaction
- supports sorting by name, date, amount through `ListOrderViewModel`
- tapping a person filters transaction lists by that person
- contextual action mode supports edit/delete and selected-sum display
- avatar comes from linked contact photo if available, otherwise generated colored circle

### Money List

Key files:

- `ui/list/TransactionListFragment.java`
- `ui/list/TransactionListAdapter.java`
- `ui/list/TransactionListViewHolder.java`
- `viewModel/TransactionListViewModel.java`

Behavior:

- shows monetary transactions
- optional person filter bar driven by `PersonFilterViewModel`
- contextual action mode supports edit/delete
- single selected transaction plus FAB can be used as a preset source
- single selected money transaction exposes "repay" shortcut via `miReturned`

### Items List

Key file:

- `ui/list/ItemTransactionListFragment.java`

Differences from money list:

- consumes `getItemTransactions()`
- filters by returned/unreturned/all using `ItemReturnedFilterViewModel`
- header total displays count of items, not currency
- contextual action mode can mark an item returned

### Edit Transaction Dialog

Key files:

- `ui/edit_transaction/EditTransactionFragment.java`
- `ui/edit_transaction/EditTransactionImageAdapter.java`
- `ui/edit_transaction/EditTransactionImageViewHolder.java`
- `viewModel/EditTransactionViewModel.java`

This is the most coupled flow in the app.

Important behavior:

- handles both create and edit flows
- uses nav args for presets and existing transaction ids
- supports creating a new person inline via `NewPersonRequestViewModel`
- images are copied into `context.getExternalFilesDir(null)/transaction-images`
- DB only stores image filenames, not paths
- image cleanup is split across:
  - link updates in `ImageDao`
  - orphan file deletion in `EditTransactionViewModel.deleteOrphanedImageFiles`
  - broken DB link cleanup in `deleteBrokenImageLinks`
- `dismiss()` also deletes orphaned image files

If you touch image behavior, verify:

- add image
- remove image before save
- edit existing transaction with images
- delete transaction or person
- backup and restore with images

### Edit Person Dialog

Key files:

- `ui/EditPersonFragment.java`
- `viewModel/EditPersonViewModel.java`
- `viewModel/ContactsHelper.java`

Behavior:

- creates or updates a person
- prevents duplicate names
- optional contact linking through `READ_CONTACTS`
- updates avatar preview immediately when name/contact changes
- can return the newly created name back to `EditTransactionFragment`

### Settings

Key files:

- `ui/SettingsFragment.java`
- `viewModel/SettingsViewModel.java`
- `res/xml/root_preferences.xml`

Settings include:

- date format
- color inversion
- decimals
- dismiss-filter behavior
- default item returned filter
- backup / restore
- FAQ / GitHub / licenses / changelog / version info

The decimals setting is high-risk. Changing it triggers `TransactionDao.changeDecimals()` and may round persisted values when decreasing precision.

## Backup / Restore

Main code:

- `database/AppDatabase.java`
- `ui/SettingsFragment.java`
- `util/FileUtils.java`
- `ui/edit_transaction/EditTransactionFragment.java` for image dir

What is backed up:

- database file
- exported shared preferences XML
- all transaction image files

Backup format:

- user-picked `.zip` via SAF

Restore flow:

1. unzip into a temporary restore directory inside the image directory
2. restore DB
3. import preferences
4. copy restored images
5. restart app

This flow is filesystem-heavy and easy to regress. If you change filenames, directories, or DB schema, re-test it.

## Resources, Localization, And Content

### Strings And Locales

- Base strings: `app/src/main/res/values/strings.xml`
- Localized strings and some localized `uri.xml` files live under `values-*`
- Localized release/store text lives under `fastlane/metadata/android/<locale>/`

Current app resource locales include:

- `ar`, `ast`, `cs`, `de`, `es`, `fa`, `fr`, `he`, `hi`, `it`, `nb-rNO`, `nl`, `pt`, `pt-rBR`, `ro`, `ru`, `vi`, `zh-rCN`

Current fastlane locales include:

- `de-DE`, `en-US`, `es`, `fa`, `fr`, `he`, `hi`, `it`, `nb-NO`, `nl`, `pt`, `pt-BR`, `ro`, `ru`, `vi`

If you add user-facing strings, check whether:

- only app resources need updates
- fastlane store descriptions also need updates
- `TRANSLATION.md` guidance is affected

### Assets

- `assets/licenses.html` and `licenses.yml` relate to third-party licenses
- `assets/changelog.html` should stay in sync with `CHANGELOG.md`
- `assets/guide_*.html` are legacy in-app guides still present in the repo

### Art And Screenshots

- `artwork/`: editable icon sources
- `fastlane/.../images/`: store screenshots and icon

## Testing

Local unit tests exist for:

- `Transaction`
- `Person`
- `PersonWithTransactions`
- `TransactionWithPerson`
- `Utilities`

Instrumentation coverage is minimal.

When changing business logic, prefer adding or updating local unit tests in `app/src/test/java/org/ebur/debitum/`.

Especially worth testing:

- amount formatting and parsing
- decimal migration behavior
- relation aggregation helpers
- equality helpers used by adapters or filtering

## Known Sharp Edges

- Transaction sign conventions are easy to invert by mistake.
- Monetary amounts are integer-backed and coupled to the user-configurable decimals preference.
- `EditTransactionFragment` mixes UI state, nav args, DB writes, and image file handling; changes there often require coordinated updates.
- Person filtering state is shared via activity-scoped ViewModel, not fragment arguments alone.
- Deleting persons or transactions removes image links immediately, but actual image files are cleaned later by edit-dialog logic.
- Backup/restore assumes specific filenames:
  - `debitum.db`
  - `debitum-preferences.xml`
- The CodeQL workflow exists, but there is no broad automated CI safety net in-repo.

## Change Playbooks

### Add a new field to a transaction or person

1. update entity class
2. update any constructors, `equals`, formatting, or parceling methods
3. add Room migration and bump DB version
4. export new schema JSON
5. update edit dialog UI if user-editable
6. update backup/restore assumptions if needed
7. add or update tests

### Add a new screen behavior that depends on filters or ordering

Check whether state should live in:

- fragment args for one-off navigation input
- a fragment-scoped ViewModel for local state
- an activity-scoped ViewModel for cross-tab persistence

### Change transaction list row content

Likely files:

- layout row XML
- `TransactionListAdapter`
- `TransactionListViewHolder`
- possibly `Transaction.hasImages`, formatting helpers, or colors

### Change backup/image behavior

Touch and verify together:

- `SettingsFragment`
- `AppDatabase`
- `FileUtils`
- `EditTransactionFragment`
- `EditTransactionViewModel`
- `ImageDao` / `ImageRepository`

## Practical Guidance For Future Agents

- Start with `MainActivity`, `nav_graph_main.xml`, and the relevant fragment before changing behavior.
- For list behavior, inspect the fragment, adapter, view holder, and shared ViewModel together.
- For persisted data changes, inspect the entity, DAO, repository, Room schema JSONs, backup/restore flow, and tests together.
- For translation-sensitive UI text, check both `strings.xml` and `TRANSLATION.md`.
- Prefer small, coordinated changes over isolated edits in this repo; many user-visible behaviors are implemented across XML, fragment code, ViewModel, and utility layers.

## Safe Defaults

- Do not remove or rename Room schema JSON history unless explicitly asked.
- Do not assume legacy guide HTML files are unused just because FAQ links replaced them in settings.
- Do not change amount sign semantics without auditing every list, total, preset, and settlement flow.
- Do not change image directory or backup filenames without auditing restore logic.
- Do not edit `.idea/` unless the task is specifically about project configuration.
