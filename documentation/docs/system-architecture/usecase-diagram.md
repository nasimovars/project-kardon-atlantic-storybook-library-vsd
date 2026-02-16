---
sidebar_position: 6
---

# Use Case Diagrams

Sequence diagrams showing the data flow for all use cases. One sequence diagram corresponds to one use case and different use cases should have different corresponding sequence diagrams.

---

## Use Case 1 – Account Creation (User / Caretaker)

**Goal**: As a user, I want to create an account so I can save and manage my storybooks.

```mermaid
sequenceDiagram
    participant User
    participant App as Application/Frontend
    participant Backend as Server/Backend
    participant DB as Database
    User->>App: Open application
    App-->>User: Display landing page
    User->>App: Select "Register"
    App-->>User: Display registration form
    User->>App: Enter username, email, password
    App->>+Backend: POST /register {username, email, password}
    activate Backend
    Backend->>+DB: Check for existing user (query)
    DB-->>-Backend: No existing user
    Backend->>DB: Insert new user record
    DB-->>Backend: Insertion successful
    Backend-->>-App: 201 Created + confirmation
    deactivate Backend
    App-->>User: Show confirmation message
    App->>User: Redirect to home/library page


```
---
## Use Case 2 – Signing In (User / Caretaker)

**Goal**: As a user, I want to log into my account so I can access my saved storybooks.

```mermaid
sequenceDiagram
    participant User
    participant App as Application/Frontend
    participant Backend as Server/Backend
    participant DB as Database
    alt Successful Login
        User->>App: Open application
        App-->>User: Display landing page
        User->>App: Select "Login"
        App-->>User: Display login form
        User->>App: Enter email, password
        App->>+Backend: POST /login {email, password}
        activate Backend
        Backend->>+DB: Validate credentials (query + hash compare)
        DB-->>-Backend: Credentials valid
        Backend-->>-App: 200 OK + session/JWT token
        deactivate Backend
        App->>User: Redirect to home page with library
    else Invalid Credentials
        User->>App: Enter email, password
        App->>+Backend: POST /login {email, password}
        activate Backend
        Backend->>+DB: Validate credentials
        DB-->>-Backend: Credentials invalid
        Backend-->>-App: 401 Unauthorized + error message
        deactivate Backend
        App-->>User: Notify "Invalid credentials"
    end
```
---
## Use Case 3 – Uploading Book (User / Caretaker)

**Goal**: As a user, I want to upload a book so I can create a VSD storybook.

```mermaid
sequenceDiagram
    participant User
    participant App as Application/Frontend
    participant Backend as Server/Backend
    participant Storage as File Storage (Supabase Storage)
    Note over User,App: User must be authenticated
    User->>App: Navigate to library page
    App-->>User: Display library view
    User->>App: Select "Upload Book"
    App-->>User: Open file picker dialog
    User->>App: Choose PDF or image file(s)
    App->>+Backend: POST /upload {multipart file, metadata}
    activate Backend
    Backend->>+Storage: Upload file to bucket
    Storage-->>-Backend: Success + public URL / file ID
    Backend->>DB: Create book record (title from file?, user_id, file_url)
    DB-->>Backend: Book ID created
    Backend-->>-App: 200 OK + book ID / metadata
    deactivate Backend
    App->>User: Open uploaded book in VSD Edit Mode
```
---
## Use Case 4 – Edit VSD Pages (User / Caretaker)

**Goal**: As a user, I want to add interactive vocabulary to the pages.

```mermaid
sequenceDiagram
    participant User
    participant App as Application/Frontend
    participant Backend as Server/Backend
    participant DB as Database

    Note over User,App: User is authenticated and has opened a book in Edit Mode
    User->>App: Enter Edit Mode for a book
    App-->>User: Display page editor canvas
    User->>App: Select object(s) on page
    App->>App: Highlight selected object(s)
    User->>App: Assign word / vocabulary to object
    App->>+Backend: PATCH /books/{bookId}/pages/{pageId} {hotspot: {word, coordinates, shape}}
    activate Backend
    Backend->>DB: Update or insert hotspot record
    DB-->>Backend: Update successful
    Backend-->>-App: 200 OK + updated page data
    deactivate Backend
    alt Add comment
        User->>App: Add comment to page
        App->>+Backend: POST /pages/{pageId}/comments {text}
        Backend->>DB: Insert comment
        DB-->>Backend: Comment ID
        Backend-->>-App: 201 Created
    end
    User->>App: Save VSD layout changes
    App->>+Backend: PATCH /books/{bookId} {pages: [...]}
    Backend->>DB: Persist all page/hotspot/comment changes
    DB-->>Backend: Save successful
    Backend-->>-App: 200 OK
    App-->>User: Show save confirmation
    App->>DB: Refresh library view (optional sync)

```
---
## Use Case 5 – Save and Manage Storybook Library (User / Caretaker)

**Goal**: As a user, I want to save and reopen books so I can edit them later.

```mermaid
sequenceDiagram
    participant User
    participant App as Application/Frontend
    participant Backend as Server/Backend
    participant DB as Database
    Note over User,App: User is authenticated and on library or edit view
    alt Save current book
        User->>App: Select "Save Book"
        App->>+Backend: PATCH /books/{bookId} {title?, pages?, status: "saved"}
        activate Backend
        Backend->>DB: Update book record & related pages/hotspots
        DB-->>Backend: Update successful
        Backend-->>-App: 200 OK + updated metadata
        deactivate Backend
        App-->>User: Show "Book saved" notification
    end
    User->>App: View library
    App->>+Backend: GET /users/{userId}/books
    Backend->>DB: Query user's books
    DB-->>Backend: List of books
    Backend-->>-App: 200 OK + book list
    App-->>User: Display library grid/list
    alt Manage book
        User->>App: Select book → Rename / Edit / Delete
        alt Rename
            App->>+Backend: PATCH /books/{bookId} {title: "New Title"}
            Backend->>DB: Update title
            Backend-->>-App: 200 OK
        else Delete
            App->>+Backend: DELETE /books/{bookId}
            Backend->>DB: Soft-delete or remove book + pages/hotspots
            Backend-->>-App: 204 No Content
            App-->>User: Remove from library view
        end
    else Reopen
        User->>App: Click book to reopen
        App->>+Backend: GET /books/{bookId}?include=pages,hotspots
        Backend->>DB: Fetch full book data
        Backend-->>-App: 200 OK + book details
        App->>User: Open in Edit or Read Mode
    end

```
---
## Use Case 6 – Pick and Read a Book (Reader / AAC User)

**Goal**: As a user, I want to be able to pick and read a book from my library.

```mermaid
sequenceDiagram
    participant User as Reader / AAC User
    participant App as Application/Frontend
    participant Backend as Server/Backend
    participant DB as Database
    participant TTS as Text-to-Speech Service

    Note over User,App: User is authenticated and on Library page

    User->>App: Select a book from library
    App->>+Backend: GET /books/{bookId}?include=pages,hotspots
    activate Backend
    Backend->>DB: Retrieve book, pages, hotspots, vocabulary
    DB-->>Backend: Return structured book data
    Backend-->>-App: 200 OK + full book JSON
    deactivate Backend

    App-->>User: Display first page in Read Mode

    User->>App: Flip to next page
    App->>App: Render next page locally

    User->>App: Tap hotspot (e.g., "cat")
    App->>TTS: Request audio for vocabulary word
    TTS-->>App: Return audio stream
    App-->>User: Play vocabulary audio

    alt User uses Talk-to-Text
        User->>App: Activate speech input
        App->>App: Capture voice input
        App-->>User: Highlight matching vocabulary
    end

    User->>App: Close book
    App-->>User: Return to library view
