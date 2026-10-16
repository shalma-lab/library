# Athenaeum — Library Management App

A simple Library Management web app built with **React**, **React Router**,
**custom hooks**, and the **Context API**.

## Features

- Search books by title, and separately by author
- Filter books by category and by availability
- View full book details on their own page (`/books/:id`)
- Borrow, return, and renew books
- Place a hold on a book that's checked out
- View reading history and simple reading statistics
- View personalized recommendations
- View member profile and stats
- Borrowing data (loans, history, holds) is saved to `localStorage`, so it
  survives a page refresh

## Project structure

```
library-app/
  public/
    index.html
  src/
    hooks/
      useBooks.js         -> book data, title search, author search,
                              category filtering, availability filtering
      useBorrowing.js      -> loans, due dates, renewals, holds, reading
                              stats, and syncing with localStorage
    context/
      LibraryContext.jsx  -> combines useBooks + useBorrowing and shares
                              them with every page via useLibrary()
    components/
      Header.jsx           -> top navigation bar (reads overdue count from context)
      Footer.jsx            -> simple footer
      BookCard.jsx           -> small reusable card for one book
      BookList.jsx             -> search/filter controls + book grid (reads
                                  everything from context, composes
                                  SearchBar + FilterPanel below)
      SearchBar.jsx              -> reusable title/author search inputs (props-only)
      FilterPanel.jsx             -> reusable category/availability filter controls (props-only)
      Status.jsx / RecommendationsSection.jsx -> see "REST API" section
      Recommendations.jsx          -> reusable row of suggested books (props-only,
                                       used on both Home and the Recommendations page)
    pages/
      Home.jsx
      Books.jsx
      BookDetail.jsx
      BorrowedBooks.jsx
      ReadingHistory.jsx
      Recommendations.jsx
      Profile.jsx
    data/
      sampleData.js        -> sample books, member, loans and history (no backend)
    App.jsx                -> wraps everything in <LibraryProvider> and <Routes>
    index.js                -> app entry point, wraps <App /> in <BrowserRouter>
    index.css                -> all styling (plain CSS, responsive)
  package.json
```

## Custom hooks

### `useBooks()`
Owns the book catalog and every search/filter control:
- State: `books`, `titleQuery`, `authorQuery`, `categoryFilter`, `availabilityOnly`
- `useEffect` loads the starting catalog once on mount
- `useMemo` builds `categories`, `authors`, and the final `filteredBooks` list
- `useCallback` provides `findBookById` and `adjustAvailability` (used by `useBorrowing`)

### `useBorrowing({ getBook, adjustAvailability })`
Owns everything about the member's borrowing activity:
- State: `loans`, `history`, `holds`
- `useEffect` loads saved borrowing data from `localStorage` on mount, and
  saves it back to `localStorage` every time it changes
- `useMemo` builds `overdueLoans`, `dueSoonLoans`, and `readingStats`
- `useCallback` provides `borrowBook`, `returnBook`, `renewBook`, `placeHold`,
  and `clearBorrowing`

## Context API

`LibraryContext` (in `context/LibraryContext.jsx`) calls both hooks and
shares the combined result through `useLibrary()`. Every page —
`BookList`/`Books`, `BookDetail`, `BorrowedBooks`, `Profile`, and
`ReadingHistory` — reads book and member data directly with `useLibrary()`
instead of receiving it as props from `App.jsx`. `App.jsx` itself holds no
shared state anymore; it just wraps the routes in `<LibraryProvider>`.

## Conditional rendering messages

- **"No books found"** — Books page, when filters/search match nothing
- **"Book currently unavailable"** — Book Detail page, when there are no
  copies left to borrow
- **"No borrowed books"** — Borrowed Books page, when nothing is checked out
- **"Book renewal is not available"** — Borrowed Books page, once the
  renewal limit has been reached for a book
- **"No reading history available"** — Reading History page, before
  anything has been returned

## Authentication (Experiment 4)

- `context/AuthContext.jsx` -> `AuthProvider` + `useAuth()` (`member`,
  `isAuthenticated`, `isInitializing`, `login`, `logout`). Restores the
  session on refresh; `isInitializing` is true until that check finishes.
- `services/authService.js` -> login / restoreSession / logout. The project
  has no backend, so accounts live in `data/authData.js` (salted SHA-256
  password hash, never plain text). Replace this one file to use a real API.
- `pages/Login.jsx` (`/login`) -> controlled form with validation, loading
  state, double-submit protection and an invalid-credentials message.
- `utils/validators.js` -> email, Member ID (`M-1001` style) and password
  (8+ chars, letter + number) rules.
- Header shows Login / Log out. `AuthProvider` is mounted in `index.js`
  inside `BrowserRouter`, alongside the existing `LibraryProvider`.
- Demo sign-in: Member ID `M-1001` (or `priya.natarajan@athenaeum.example`),
  password `Library@2024`.

## REST API & async data (Experiment 1)

The project had no backend, so `server/server.cjs` (zero dependencies) was
added. It serves the **existing** `src/data/sampleData.js` over REST; swap it
for a real backend any time -- the React app only depends on this contract.

| Purpose | Endpoint |
|---|---|
| Books (all; optional `search`, `category`, `available`) | `GET /api/books` |
| Book details | `GET /api/books/:id` (400 bad id, 404 not found) |
| Related books | `GET /api/books/:id/related` |
| Member | `GET /api/members/:id` |
| Recommendations | `GET /api/members/:id/recommendations?exclude=1,2&categories=A,B` |

Frontend layout:
- `services/apiClient.js` -> shared Axios instance (`REACT_APP_API_BASE_URL`,
  default `/api`), safe error messages, abort helpers
- `services/bookService.js`, `memberService.js` -> Axios;
  `recommendationService.js` -> Fetch (checks `response.ok`)
- `hooks/useBooks.js` (catalog + `booksStatus`), `useBook.js` (one book + related),
  `useMember.js`, `useRecommendations.js` -> loading / success / error state,
  `AbortController` cancellation, retry
- `components/Status.jsx` -> `LoadingState` / `ErrorState` (with retry)
- `components/RecommendationsSection.jsx` -> API recommendations; if the API fails
  it shows the error plus the old in-browser suggestions, labelled as local

Search and category filters run in memory on the loaded catalog (no request per
keystroke). The title box also matches ISBN when a book has one. Borrowing is
unchanged and still lives in the browser (`localStorage`), so live copy counts
override the count returned by the API.

Run it with two terminals: `npm run start:api` (port 5001) and `npm start`.
Set `API_DELAY_MS=1500` before `npm run start:api` to see loading states.

## Running the app locally

1. Make sure you have [Node.js](https://nodejs.org) installed (v16 or newer).
2. Open a terminal in the `library-app` folder.
3. Install dependencies:
   ```
   npm install
   ```
4. Start the REST API (terminal 1), then the React app (terminal 2):
   ```
   npm run start:api
   npm start
   ```
5. The app will open automatically at `http://localhost:3000`.

## Notes

- The catalog and member now come from a small REST API (`server/`) backed by
  the same sample data — there is no database. Borrowing activity is saved to `localStorage`,
  so it will persist across refreshes; the catalog itself resets each time.
- Built with `react-router-dom` v6 (`Routes`, `Route`, `Link`, `NavLink`,
  `useParams`, `useNavigate`).

## Experiment 6 – Library State Management (Zustand)

### Chosen library
**Zustand** (v5) with its official `persist` middleware.

**Reason for selection**
- The project previously used React Context + custom hooks (`useBooks`, `useBorrowing`, `useMember`) and had no Redux Toolkit or Zustand dependency.
- Zustand is a minimal, modern global store that fits a medium-sized SPA without Provider boilerplate.
- Its `persist` middleware maps cleanly onto the existing `localStorage` pattern used for loans / history / holds.
- Redux Toolkit was considered first (preferred when a structured multi-slice store is needed), but Zustand was chosen for lighter integration and zero peer-dependency friction in this environment.

### Global store (`src/store/libraryStore.js`)
Centralised state categories:

| Category | State |
|---|---|
| **Books** | `books`, `booksStatus`, `booksError`, search/filter UI state (`titleQuery`, `authorQuery`, `categoryFilter`, `availabilityOnly`) |
| **Borrowed** | `loans`, `holds`, `actionLoading`, `lastMessage` |
| **Reading history** | `history` |
| **Member** | `member`, `memberStatus`, `memberError`, `memberId` |

### Actions
- `fetchBooks` / `fetchMember` – async, AbortController-aware, update status/error.
- `borrowBook`, `returnBook`, `renewBook`, `placeHold` – pure client-side (matches existing backend contract which only exposes GET endpoints). Validate availability / eligibility, update loans/history/holds and book `availableCopies`, set user-facing `lastMessage`.
- `clearBorrowing` / `clearMemberState` – used on logout and demo reset.
- Filter setters + `resetFilters`, `findBookById`, `adjustAvailability`.

### Middleware / monitoring
A lightweight development-only logger wraps the store. It records timestamps and non-sensitive state changes (counts of loans/history/holds, status flags, lastMessage) without logging member PII or full book arrays. Production builds are silent.

### Browser storage synchronisation
Zustand `persist` middleware (key `athenaeum_library_store_v1`) writes only durable, non-sensitive fields:
- `loans`, `history`, `holds`, `memberId`
Books and the live member profile are always re-fetched from the API on load so server data wins over any stale cache. Transient flags (`booksStatus`, `actionLoading`, etc.) are never persisted.

### Integration
`LibraryProvider` (still the single entry point used by every page) now reads from and writes to the Zustand store. The public shape of `useLibrary()` is unchanged, so existing pages and components required no structural rewrites.

### New / modified files
- **New**: `src/store/libraryStore.js`
- **Modified**: `src/context/LibraryContext.jsx` (backed by the store), `src/context/AuthContext.jsx` (clears member-specific store state on logout), `package.json` (adds `zustand`)

Existing hooks (`useBooks`, `useBorrowing`, …) remain in the tree for reference but are no longer the source of truth; the store is.

### How to run
```bash
npm install
npm run start:api   # terminal 1
npm start           # terminal 2
```
