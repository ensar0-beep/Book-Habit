# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, no-build, vanilla JS/HTML/CSS web app: `index.html`. It's a personal reading-habit tracker ("Kitap Takip", Turkish UI) that gamifies daily page-count streaks. There is no package.json, no bundler, no test suite, and no other source files — the entire app (markup, styles, and logic) lives in this one HTML file.

## Running / developing

There is no build or install step. To work on it:
- Open `index.html` directly in a browser, or serve it statically, e.g. `python3 -m http.server` from this directory and visit the file.
- Edit HTML/CSS/JS in place inside the single file (structure: `<style>` block, then markup, then one big `<script>` at the bottom).
- There is no linter, formatter, or test runner configured — verify changes manually in the browser.

## Architecture

**Everything lives in one global `S` object** (see `freshState()`), persisted to `localStorage` (key `kitap_v1`) after every mutation via `saveLocal()`, and mirrored to a public, unauthenticated Firebase Realtime Database (`FB_URL`) via `fbGet`/`fbSet` REST calls (`syncToFb()`). On load, `doLogin()` merges local state with whatever's in Firebase (remote wins for most fields; history entries are merged by date). There's no real auth — `MEMBERS` is a hardcoded single-user array driving auto-login on `window.load`.

**Habit engine is history-driven, not incrementally mutated.** `S.history` (array of `{date, pages, success, bookId?/bookPages?}`) is the source of truth. Whenever an entry is added/edited (`applyReading()`), `recalcAll()` replays the *entire* history chronologically to deterministically rederive `levelIndex`, `streak`, and `totalDays` — never trust/update these fields directly. The progression itself is defined by the `LEVELS` array: 8 stages (1, 3, 5, 10, 20, 30, 40, 50 pages/day), each requiring a run of consecutive successful days (`days`) before advancing; a failed/missed day drops a level and resets the streak.

**Books model**: `S.books[]` holds books currently being read (id, name, totalPages, currentPage, coverId), with `S.selectedBookId` marking the active one and `S.todaySessions` (map of bookId → pages read today) supporting logging pages across multiple books in one day. `migrateBooks()` upgrades the older single-book schema (`S.bookName`/`S.currentPage`/etc.) into `books[]` for backward compatibility with existing localStorage/Firebase data — keep this migration working when changing the book schema. Finished books move into `S.finishedBooks[]`.

**External integrations**: book search/autofill and cover images come from the Open Library API (`openlibrary.org/search.json`, `covers.openlibrary.org`) — no API key, called directly from the browser.

**UI**: two tabs — "Bugün" (today: page slider/quick-set buttons, current books, stats) and "Yolculuk" (journey: chain progress visual, level list, monthly calendar with tap-to-edit-past-days via a modal). Rendering is manual DOM manipulation (`renderMe()`, `renderChain()`, `renderLevels()`, `renderCal()`, etc.) — no framework, no virtual DOM; after any state change, call `saveLocal()` + `syncToFb()` then the relevant `render*()` function(s).
