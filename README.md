# saycut

Type a sentence, get a signed Apple Shortcut.

```
prompt › Claude › cherri › sign › .shortcut
```

saycut sends your sentence to Claude, which writes [Cherri](https://github.com/electrikmilk/cherri)
code. The Cherri compiler turns that into a Shortcut and `shortcuts sign` signs it. If
compilation fails, the error goes back to Claude and it tries again.

Status: work in progress. This is a Go learning project, written by hand.

## Learning guide

- [docs/go-guide.en.md](docs/go-guide.en.md): English
- [docs/go-guide.html](docs/go-guide.html): Slovak, interactive (open it in a browser)

Progress is tracked in [issues](../../issues) and [milestones](../../milestones).

## Requirements

- macOS (for `shortcuts sign`)
- Go (latest stable)
- [cherri](https://github.com/electrikmilk/cherri)
- An Anthropic API key

## License

MIT
