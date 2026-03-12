# Kilo

**Project:** [antirez/kilo](https://github.com/antirez/kilo)
**Language:** C
**Size:** 1,308 lines, one file
**Author:** Salvatore Sanfilippo (antirez)

---

The author of Redis wrote a text editor in a single C file in a few hours. It has syntax highlighting, search, and a status bar. It does not have undo.

That last sentence is the reading.

## What It Is

Kilo is a terminal text editor. One file, `kilo.c`. No dependencies beyond POSIX — no curses, no readline, no termcap database. It talks directly to the terminal using VT100 escape sequences, the way programs did before libraries existed to protect you from having to.

The comment on the function that puts the terminal into raw mode says: "Raw mode: 1960 magic shit." That's the voice of the entire project. The problem is taken seriously. The code is not.

## The Terminal As Material

Most terminal applications treat the terminal as an API — call curses, let it handle the details. antirez treats the terminal as raw material. He writes escape codes directly into a buffer and flushes them in a single `write()` call.

The `getWindowSize` function is the clearest example of this relationship. First, try `ioctl(TIOCGWINSZ)`. If that fails — and it does, on enough terminals to matter — move the cursor to position 999,999 (far past any real screen boundary), then ask the terminal where the cursor actually ended up. The terminal clamps it to the real dimensions. Read those back. Restore the original position.

This is not defensive programming. This is someone who has spent decades inside terminals and knows that the clean path fails often enough that the ugly path isn't optional. linenoise, his readline replacement, taught him this. The knowledge isn't theoretical.

## Two Representations of Every Line

Each row in the editor carries two versions of itself:

```c
typedef struct erow {
    int size;           /* Size of the row, excluding the null term. */
    int rsize;          /* Size of the rendered row. */
    char *chars;        /* Row content. */
    char *render;       /* Row content "rendered" for screen (for TABs). */
    unsigned char *hl;  /* Syntax highlight type for each character in render. */
} erow;
```

`chars` is what's in the file. `render` is what the terminal needs to show. The gap between them is almost entirely tabs: a tab in `chars` becomes up to 8 spaces in `render`. Every time a row changes, both representations are rebuilt. The syntax highlighter runs against `render`, not `chars`, because that's what maps to screen positions.

This dual representation is the core architectural decision. Everything downstream — cursor positioning, search matches, horizontal scrolling — works against one representation or the other, and the choice of which one matters each time. Get it wrong and the cursor drifts from the text. A richer editor would add a third layer for Unicode grapheme clusters. Kilo doesn't handle Unicode. That's a decision, not an oversight.

## The Keyword Hack

The syntax highlighting database stores keywords as string arrays. To distinguish between two classes of keywords — language keywords like `if` and type keywords like `int` — antirez appends a trailing pipe character:

```c
"int|","long|","double|","float|","char|","unsigned|","signed|",
"void|","short|","auto|","const|","bool|",NULL
```

The highlighter checks the last character: if it's `|`, decrement the length and use color 2 instead of color 1. No separate data structure. No mapping table. No enum. The information is encoded in the data itself, using a character that can't appear in a C keyword.

The entire highlight database — `HLDB` — ships with one entry: C/C++. The infrastructure supports any number of languages. He shipped with one. Not because more would be hard; the comment block above HLDB explains exactly how to add more, in detail. He just didn't need more right then.

## What Isn't There

No undo. No copy/paste. No line wrapping. No multiple files. No configuration file. No Unicode. No mouse support. No auto-indent. The README calls it "alpha stage."

The TODO file has two sections. Under IMPORTANT: "Testing and stability to reach 'usable' level." Under MAYBE: "Improve internals to be more understandable."

This is an editor written by someone who builds systems used by millions, and the TODO doesn't mention any features. It mentions testing and legibility. The features he left out aren't things he doesn't know how to build. He built Redis. He knows what an undo log looks like — it's a write-ahead log, and he's written those. The absence is deliberate. Not minimalism as aesthetic; minimalism as structural commitment. Once you add undo, you need redo. Once you add redo, you need a transaction model. Once you have transactions, you're maintaining state that the simple array-of-rows model can't support without fundamental changes.

The save function is the clearest expression of this philosophy:

```c
if (ftruncate(fd,len) == -1) goto writeerr;
if (write(fd,buf,len) != len) goto writeerr;
```

Truncate, then write. Not a temp-file-and-rename. Not `fsync`. The comment says: "a bit safer, under the limits of what we can do in a small editor." He knows this can corrupt on power loss. He knows the safe pattern. He chose not to use it because the safe pattern requires tempfile management, which means path manipulation, which means another twenty lines that have nothing to do with editing text. The boundary is drawn and labeled.

## The Conversation Between Projects

antirez says in the README that kilo was written "taking code from my other two projects, load81 and linenoise." linenoise is his readline replacement — 1,100 lines that handle the same raw terminal manipulation, escape sequence parsing, and history that readline does in 30,000. load81 is a Lua programming environment for kids, built on SDL.

The raw mode setup, the escape sequence reader, the `getWindowSize` fallback — these aren't invented for kilo. They're extracted from linenoise, battle-tested over six years of production use. The comment says "1960 magic shit" because by 2016, antirez has been writing this same code since 2010 and it still feels like magic.

What kilo adds on top of linenoise's terminal foundation is structure: the row abstraction, the dual representation, the highlight engine, the viewport math. linenoise handles a single line. Kilo handles a file. The vertical axis is the new dimension.

## The Refresh Strategy

The screen refresh paints the entire visible area on every keystroke. Build a buffer. Write every row. Write the status bar. Write the cursor position. Flush once.

```c
struct abuf {
    char *b;
    int len;
};
```

This append buffer — six lines of code — exists to avoid flicker. Without it, each escape sequence would be a separate `write()` call, and the terminal would render intermediate states. With it, the screen updates atomically from the terminal's perspective. The buffer grows via `realloc` on every append, which is technically quadratic in the number of appends per refresh. He doesn't care. The buffer is rebuilt from scratch on every keypress and freed immediately after. For a screen-sized buffer, the realloc cost is noise.

A "real" editor would maintain a damage model — track which parts of the screen changed and only repaint those. Kilo repaints everything, every time. For a file you're editing on a modern terminal, this is fast enough that the engineering cost of partial updates would never be recovered. antirez knows what partial updates look like — Redis uses them extensively. He chose full repaint because the problem doesn't require the optimization.

## The Search

The find mode is a self-contained loop inside `editorFind()`. It saves the cursor position, enters its own read-key/refresh cycle, and either commits the new position (Enter) or restores the original (Escape). While searching, it highlights matches by temporarily overwriting the syntax highlight array and restoring it when the match moves.

This is an event loop inside an event loop. The outer loop is `main()` calling `editorProcessKeypress` forever. When Ctrl-F is pressed, control enters `editorFind`, which runs its own loop until the user exits search mode. No state machine. No mode flag in the global state. The function *is* the mode.

That works because there's no concurrency. No signals to handle mid-search (SIGWINCH will queue). No async I/O. No threads. The program is a single thread reading one byte at a time from a file descriptor. The simplest possible execution model, chosen because the problem permits it.

## What It Tells You

Kilo is interesting not because of what it does — plenty of small editors exist — but because of who wrote it and what he chose to leave out. antirez has the skill to build a production-grade editor. He built a teaching one instead, and the distance between the two reveals exactly where the complexity in text editing actually lives.

The complexity isn't in putting characters on screen. The refresh loop is thirty lines. The complexity is in the relationship between the logical document and its visual representation — the `chars`/`render` split, the cursor math that accounts for tabs, the horizontal scrolling offset that must be applied differently to the cursor and to the text. And the complexity is in the state model: once you add undo, the row array stops being the document and becomes a view of the document. Everything changes.

antirez drew the line right before that change. The editor is complete in the way a sketch is complete — it has everything it needs and nothing it doesn't, and you can see exactly where the next stroke would go.

The last commit was in January 2025 — nine years after the initial release. A one-line fix for a function declaration. The editor itself hasn't changed since 2016. He got it right the first time, within the boundaries he set, and left it there.
