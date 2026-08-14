# RoomMVVMBasics

A small Android app demonstrating local contact persistence with Room and the MVVM pattern. On launch, it reads saved contacts, inserts a sample contact, and displays the first contact's name.

## Architecture at a glance

```mermaid
flowchart LR
    UI["MainActivity\nView + lifecycle owner"]
    BINDING["ActivityMainBinding\ntvValue TextView"]
    FACTORY["ContactViewModelFactory\nInjects ContactDAO"]
    VM["ContactViewModel\nUI state + viewModelScope"]
    DAO["ContactDAO\nCRUD SQL contract"]
    ROOM["ContactDatabase\nRoomDatabase"]
    DBHELPER["DBHelper\nBuilds contactDB + returns DAO"]
    SQLITE[("contactDB\nSQLite database")]
    MODEL["Contact entity\nid, name, phone"]

    UI --> BINDING
    UI -->|"getInstance(context)"| DBHELPER
    DBHELPER --> ROOM
    ROOM --> DAO
    ROOM <--> SQLITE
    UI -->|"ContactDAO"| FACTORY
    FACTORY -->|"creates with DAO"| VM
    UI -->|"insertContact(sample)"| VM
    VM -->|"insert / update / delete / query"| DAO
    DAO -->|"maps rows"| MODEL
    MODEL -->|"List<Contact>"| VM
    VM -->|"contacts: LiveData<List<Contact>>"| UI
    UI -->|"sets first contact name"| BINDING
```

### Layer responsibilities

| Layer | Current class(es) | Responsibility |
| --- | --- | --- |
| View | `MainActivity`, `activity_main.xml` | Creates the database access path, observes contacts, and displays a contact name. |
| Presentation | `ContactViewModel`, `ContactViewModelFactory` | Runs database work in `viewModelScope`, exposes contact state, and receives the injected DAO. |
| Persistence setup | `DBHelper`, `ContactDatabase` | Builds the Room database named `contactDB` and exposes its DAO. |
| Data access | `ContactDAO` | Declares suspend CRUD operations and the SQL query. |
| Data model | `Contact` | Defines the Room entity/table: `id`, `name`, and `phone`. |

> The current project has no repository layer: `ContactViewModel` calls `ContactDAO` directly. That is appropriate for this focused Room example; introduce a repository when multiple data sources or shared data rules are added.

## App-start and insert flow

```mermaid
sequenceDiagram
    autonumber
    participant A as MainActivity
    participant H as DBHelper
    participant D as ContactDatabase / Room
    participant F as ContactViewModelFactory
    participant V as ContactViewModel
    participant DAO as ContactDAO
    participant DB as contactDB (SQLite)

    A->>H: getInstance(context)
    H->>D: databaseBuilder(..., "contactDB").build()
    D-->>H: contactDAO()
    H-->>A: ContactDAO
    A->>F: ContactViewModelFactory(dao)
    A->>F: Create ContactViewModel
    F-->>A: ContactViewModel
    V->>V: init → fetchContacts()
    V->>DAO: getContact() in viewModelScope
    DAO->>DB: SELECT * FROM contact
    DB-->>DAO: Contact rows
    DAO-->>V: List<Contact>
    V-->>A: Publish contacts LiveData
    A->>A: Observe contacts and set tvValue

    A->>V: insertContact(Contact(0, name, phone))
    V->>DAO: insertContact(contact) in viewModelScope
    DAO->>DB: INSERT INTO contact
    DB-->>DAO: Insert completes
    V->>DAO: fetchContacts() → getContact()
    DAO->>DB: SELECT * FROM contact
    DB-->>V: Updated contact list
    V-->>A: Publish updated contacts LiveData
    A->>A: Render first contact name
```

## CRUD flows

```mermaid
flowchart TD
    START["UI calls ContactViewModel"] --> OP{"Operation"}
    OP -->|"insertContact"| INSERT["DAO @Insert\nINSERT INTO contact"]
    OP -->|"updateContact"| UPDATE["DAO @Update\nUPDATE contact"]
    OP -->|"deleteContact"| DELETE["DAO @Delete\nDELETE FROM contact"]
    OP -->|"initial fetch / post-insert refresh"| READ["DAO getContact()\nSELECT * FROM contact"]
    INSERT --> REFRESH["fetchContacts()"]
    REFRESH --> READ
    READ --> STATE["contacts LiveData"]
    STATE --> UI["MainActivity observer\nupdates tvValue"]
    UPDATE --> DB[("contactDB")]
    DELETE --> DB
    READ --> DB
    INSERT --> DB
```

## Project map

```text
app/src/main/java/com/projects/roombasics/
├── MainActivity.kt               # View setup, observer, sample insertion
├── ContactViewModel.kt           # Contact state and coroutine-backed CRUD calls
├── ContactViewModelFactory.kt    # Injects ContactDAO into the ViewModel
├── DBHelper.kt                   # Creates Room and provides ContactDAO
├── ContactDatabase.kt            # Room database definition
├── ContactDAO.kt                 # DAO CRUD methods and SELECT query
└── Contact.kt                    # Room entity/table model
```

## Database reference

```text
Database: contactDB
Table:    contact

contact
├── id: Long       (primary key, auto-generated)
├── name: String
└── phone: String
```

The insert currently supplies `id = 0`; because the key is auto-generated, Room assigns the stored contact ID.

## Notes for future changes

- `contacts[0]` assumes the database contains at least one item. Safely handle an empty list before rendering the first name.
- `getContact()` returns a one-time `List<Contact>`. Returning `LiveData<List<Contact>>` or `Flow<List<Contact>>` from the DAO would let Room automatically notify the UI after all writes, including update and delete.
- Only insertion refreshes the list in the current code. Refresh after update and delete too, or adopt an observable DAO query.
- Prefer a single database instance (for example, held as a singleton) rather than rebuilding it from `DBHelper.getInstance()` on each Activity creation.
- Add a repository layer when the app needs multiple data sources, caching rules, or easier unit testing.
