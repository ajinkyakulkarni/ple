# PLE - a  Pure Lua Editor

A small, self-contained text editor for the Unix console or standard terminal emulators. 

### Objective

Have a small, self-contained text editor that does not rely on any external library (no termcap, terminfo, ncurses, no POSIX libraries).

This is not intended to compete with large established editors (emacs, vim) or smaller ones (joe, zile) or, for example in the Lua world, the sophisticated Textadept with a ncurse interface.  PLE is rather intended to be used with a statically compiled Lua in the sort of environment where busybox is usually found.

The editor is entirely written in Lua.  The only external dependencies are the terminal itself which must support basic ANSI sequences and the unix command `stty` which is used to save/restore the terminal mode, and set the terminal in raw mode (required to read keys one at a time).

It is written and tested with Lua 5.3. (there may be one or two goto and labels, and integer divisions somewhere...).  

PLE has been inspired by Antirez' [Kilo](https://github.com/antirez/kilo) editor. PLE structure and code have nothing to do with Kilo, but the idea to directly use standard ANSI sequences and bypass the usual terminfo and ncurses libraries comes in part from Kilo (and also from Antirez' more established [linenoise](https://github.com/antirez/linenoise) library)

### Limitations

PLE is ***Work in Progress***! It is not intended to be used for anything serious, at least for the moment.

The major limitations the brave tester should consider are:
- no UTF8 support. PLE displays 1-byte characters, and only the printable characters (code 32-126 and 160-255). Others characters are displayed as a centered dot (code 183).
- TAB length is set to 8 for all buffers
- no word- or sentence- or paragraph-based movement.
- the search and replace functions do not work with special characters, new lines and regular expressions - plain text in a line, case sensitive only.
- no provision for automatic backup files.
- no automatic redimensioning of windows for X terminals 
(no SIGWINCH handling - but the redisplay function (^L) ) can be invoke to refresh the screen to the new size)
- no syntax coloring.
- no clean API yet for extension modules ("modes" a la emacs, syntax coloring, etc.)
- probably many others things that you would expect from a text editor!

On the other hand, the editor already support:
- a keybinding  a la Emacs (but much limited!)
- basic editing functions
- basic search and replace functions (plain text only)
- support for long lines (horizontal scroll, no provision for wrapping long lines)
- selection, selection highlight, cut and paste (mark, wipe and yank in emacs parlance)
- crude undo/redo functions
- multiple buffers (but just one window at a time for the moment)
- read, write, save files.
- simple macro recording and playing (a la emacs)
- a minimal help command (F1 or ^X^H - give a short description of the basic bindings)

Configuration -- the editor can be configured / customized with a 
Lua file loaded at initialization. The Lua configuration file is loaded 
with the Lua function `loadfile`. It is executed in the environment of the editor. The `editor` global object is visible and can be used by the configuration script.

The configuration file is looked for in sequence at the following locations:
- the file which pathname is in the environment variable `PLE_INIT`
- `./ple_init.lua`
- `~/config/ple/ple_init.lua`

The first file found, if any, is loaded. 

A sample `ple_init.lua` file is provided. It defines an editor command to execute the current line as a shell command and insert the result in the current buffer at the cursor.

--

At the moment, the complete editor is 40KB. It has been tested on xterm, rxvt and the Linux console. 

As they say, *it works on my PC...*


### Default key bindings

```


Cursor movement
	Arrows, PageUp, PageDown, Home, End
	^A ^E           go to beginning, end of line
	^B ^F           go backward, forward
	^N ^P           go to next line, previous line
	esc-<           go to beginning of buffer
	esc->           go to end of buffer
	^S              forward search (plain text, case sensitive)
	^R              search again (string previously entered with ^S)
 	esc-g           prompt for a line number, go there
  
Edition
	^D, Delete      delete character at cursor
	^H, bcksp       delete previous character
	^K              cut from cursor to end of line
	esc-k           cut from cursor to beginning of next line
	                (if repeated, lines are appended to the paste buffer)
	^space, ^@      mark  (set beginning of selection)
	^W              wipe (cut selection)
	^Y              yank (paste)
	esc-5           replace
	esc-7           replace again (with same strings)

Files, buffers
	^X^F            prompt for a filename, read the file in a new buffer
	^X^W            prompt for a filename, write the current buffer
	^X^S            save the current buffer
	^X^B            create a new, empty buffer
	^X^N            switch to the next buffer
	^X^P            switch to the previous buffer

Misc.
	^X^C            exit the editor
	^G              abort the current command
	^Z              undo 
	esc-z           redo 
	^X(             record macro
	^X)             stop recording macro
	^Xe, ^]         play macro
	^L              redisplay the screen (useful if the screen was 
                        garbled or its dimensions changed)
	F1, ^X^H        this help text

]]
```

### Usage

`lua ple.lua [filename]`


### The term module

The `term` module includes all the basic functionnalities to display strings in various colors, move the cursor, erase lines and read keys.

This module is at the beginning of the ple.lua file. It does not use any other external library.  The only reason for embedding it within ple.lua is to deliver the editor as a single lua file.

To make it available for other applications, it is also distributed as a separate module. See the [plterm](https://github.com/philanc/plterm) repository.

Like [linenoise](https://github.com/antirez/linenoise), it does not use ncurses, terminfo or termcap. It uses only very common ANSI sequences that are supported by (at least) the Linux console, xterm and rxvt.

The origin of the term module is some code [contributed](http://lua-users.org/lists/lua-l/2009-12/msg00937.html) by Luiz Henrique de Figueiredo on the Lua mailing list some time ago.

I added some functions for input, getting the cursor position or the screen dimension, and stty-based mode handling .

The input function reads and parses the escape sequences sent by function keys (arrows, F1-F12, insert, delete, etc.). See the definitions in `term.keys`.


Term functions:
```
clear()     -- clear screen
cleareol()  -- clear to end of line
golc(l, c)  -- move the cursor to line l, column c
up(n)
down(n)
right(n)
left(n)     -- move the cursor by n positions (default to 1)
color(f, b, m)
            -- change the color used to write characters
			   (foreground color, background color, modifier)
			   see term.colors
hide()
show()      -- hide or show the cursor
save()
restore()   -- save and restore the position of the cursor
reset()     -- reset the terminal (colors, cursor position)

input()     -- input iterator (coroutine-based)
		       return a "next key" function that can be iteratively called 
			   to read a key (escape sequences returned by function keys 
			   are parsed)
rawinput()  -- same, but escape sequences are not parsed.
getcurpos() -- return the current position of the cursor
getscrlc()  -- return the dimensions of the screen 
               (number of lines and columns)
keyname()   -- return a printable name for any key
               - key names in term.keys for function keys,
			   - control characters are represented as "^A"
			   - the character itself for other keys

tty mode management functions

setrawmode()       -- set the terminal in raw mode
setsanemode()      -- set the terminal in a default "sane mode"
savemode()         -- get the current mode as a string
restoremode(mode)  -- restore a mode saved by savemode()

```

### License

This code is published under a MIT license. Feel free to fork!




