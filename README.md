# Todo

A single-page todo app. One HTML file, no build step, no dependencies, no server. Tasks are saved in your browser via `localStorage`.

## Run it

Open [index.html](index.html) in a browser — double-click the file, or:

```sh
open index.html          # macOS
```

If you prefer serving it over HTTP:

```sh
python3 -m http.server 8000
# then visit http://localhost:8000
```

## What it does

- Add tasks, mark them done, delete them
- Edit a task by double-clicking its text (`Enter` saves, `Esc` cancels)
- Filter by **All** / **Active** / **Done**
- Clear all completed tasks at once
- Live count of remaining tasks
- Tasks persist across reloads
- Light and dark themes, following your OS setting
- Responsive down to small screens; keyboard and screen-reader friendly

## Project layout

```
index.html    the entire app — markup, styles, and logic
README.md     this file
```

## Data

Everything lives under the `localStorage` key `todo.items.v1`, as an array of `{ id, text, done }`. Nothing leaves the browser — no network calls, no analytics. Clearing site data for the page wipes your tasks.

To reset manually, run this in the browser console:

```js
localStorage.removeItem("todo.items.v1");
```
