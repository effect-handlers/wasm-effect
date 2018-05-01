# wasm-effect

Effect handlers for WASM.


# Rendering

The main document is `wasm-effect.md`. It can be viewed as standard
markdown but also rendered nicely using Madoko for HTML and PDF. When you
edit the document use 4 backticks to delimit mathematical content, and 5
backticks for inference rules. The `wasm-style.mdk` lets you style and
transform any content.

## Command Line

Madoko can be run from the command line as:
```
> madoko -v --odir=out wasm-effect.md
```

This generates a HTML file in `out/wasm-effect.html`. You can generate
a PDF too using:  
```
> madoko -v --odir=out --pdf wasm-effect.md
```

You can install madoko using `npm`, the [NodeJS](https://www.nodejs.org) package manager, using:
```
> npm install -g madoko
```

## Web editor

Go to <https://www.madoko.net> to edit the document in a live editor.
You can open the document directly from your Github repository and
when syncing, `ctrl+s`, you can specify commit messages. 

Using Dropbox is even nicer as any changes are saved automatically.