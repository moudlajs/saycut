# Go from zero: saycut

> English version of [`go-guide.html`](go-guide.html) (Slovak, interactive).
> Keep the two in sync when you change one.

Type a sentence, get a signed Apple Shortcut. You build it yourself and along the way
you learn why Go looks exactly the way it does.

**What your program will do in the end:**

```
prompt › Claude › cherri › sign › .shortcut
```

```
prompt: "When I get home, send me the camera status"
Claude writes Cherri (attempt 1/3)
cherri: line 4: unknown action "sendNotifcation"      ← compile fails
the error goes back into the conversation (attempt 2/3)
cherri: compile OK
shortcuts sign --mode anyone: OK
dist/domov.signed.shortcut, tap it and import
```

Notice what happens when compilation fails: the error is fed back to Claude, which fixes its code.

## Chapters

0. [What we're building](#0-what-were-building)
1. [Why Go looks weird](#1-why-go-looks-weird)
2. [Variables and types](#2-variables-and-types)
3. [Structs instead of classes](#3-structs-instead-of-classes)
4. [Pointers: & and *](#4-pointers--and-)
5. [Errors without exceptions](#5-errors-without-exceptions)
6. [Talking to Claude](#6-talking-to-claude)
7. [Running other programs](#7-running-other-programs)
8. [Pipeline and the retry loop](#8-pipeline-and-the-retry-loop)
9. [main.go and the first run](#9-maingo-and-the-first-run)
10. [Interfaces and the first test](#10-interfaces-and-the-first-test)
11. [What's next: Telegram bot](#11-whats-next-telegram-bot)

---

## 0. What we're building

The MVP has 6 Go files, roughly 300 lines and **zero external dependencies**. The Go
standard library does everything. All files live in one package, `package main`, so Go
sees them as one program. You split it into packages later, when main gets cramped.

```
saycut/
├── go.mod              module name + Go version
├── main.go             entry: arguments → pipeline → result
├── config.go           env variables + validation
├── llm.go              HTTP call to the Claude API
├── cherri.go           runs the cherri compiler
├── sign.go             runs shortcuts sign
├── pipeline.go         retry loop, the glue of everything
├── pipeline_test.go    first test
├── prompts/system.md   how Claude should write Cherri
├── examples/hello.cherri  hand-verified example
└── Makefile
```

Starting the project:

```sh
mkdir saycut && cd saycut
go mod init github.com/moudlajs/saycut
go run .          # runs the package in the current directory
```

> **Phase 0 first.** Compile and sign `hello.cherri` by hand and verify the exact flags of
> `cherri` and `shortcuts sign`. The code below assumes cherri produces `name.shortcut`
> next to the source. If it does it differently, or signs by itself, you change one line.

## 1. Why Go looks weird

Your feeling of "C crossed with something" is spot on. Go was designed at Google by
people who built Unix and C (Ken Thompson, Rob Pike). They took curly braces and pointers
from C and deliberately simplified the rest:

- **The type comes after the name**: `name string` instead of `string name`. You read it
  left to right: "name is a string". With complex types like `func(int) []string` it stays
  readable.
- **No semicolons.** The compiler inserts them, which is why `{` must be on the same line
  as `if` or `func`.
- **One loop**: `for`. No `while`, no `do`.
- **No classes, inheritance or exceptions.** It's not something missing, it's intentional.
  Less magic means you see everything directly in the code.
- **Formatting is handled by `gofmt`**, so there are no arguments about whitespace.

### Weird-symbol decoder

This list covers almost everything that will trip you up in Go.

| Symbol | Meaning | Explanation | Example |
|---|---|---|---|
| `:=` | Create a variable and infer its type | Shorthand for `var x T = value`. Only works inside functions; at package level you must write `var`. | `name := "saycut"` is the same as `var name string = "saycut"` |
| `&x` | Address of a variable | "Give me a link to x." The result has type `*T`. Use it when a function should change the original. | `setModelP(&cfg)`, `json.Unmarshal(raw, &out)` |
| `*T` | Pointer to T (in a type) | In a type declaration it means "link to T". In an expression, `*p` means "the value p points to". | `func NewLLM(cfg Config) *LLM`, `var p *Config = &cfg` |
| `[]T` | Slice, a dynamic list | Grows with `append`. Careful: `append` returns a new slice, always assign it back. | `files = append(files, "b.go")` |
| `map[K]V` | Dictionary | Key → value. Reading a missing key returns the zero value. Check existence with "comma ok". | `v, ok := env["X"]  // ok == false` |
| `func (p *T)` | Receiver, a method of a type | Instead of a class with methods, you attach a function to a type. `p` is basically `this`, just explicitly named. | `func (p *Pipeline) Run(...) error { /* p.cfg, p.llm */ }` |
| `_` | Discard a value | Go doesn't allow unused variables, so what you don't need goes to `_`. | `for _, c := range out.Content { ... }` |
| `...` | Variadic / unpacking | In a parameter: the function takes any number of values. In a call: unpack a slice into its elements. | `func log(args ...any)`, `all = append(all, more...)` |
| `` `json:"x"` `` | Struct tag | Metadata for another package. Here it tells the JSON encoder what the field is called in JSON. | ``MaxTokens int `json:"max_tokens"` `` |
| `err != nil` | Did an error happen? | `nil` = "nothing". If the error is nil, everything went fine. You'll write this line a thousand times. | `if err != nil { return fmt.Errorf("x: %w", err) }` |
| `%w` | Wrap an error | In `fmt.Errorf` it keeps the original error inside the new one. `%v` would turn it into plain text. | `fmt.Errorf("compile: %w", err)` |
| `defer` | Run on the way out | Schedules a call for the end of the function, however it leaves. Ideal for `Close()` and `cancel()`. | `defer resp.Body.Close()` |
| `<-` | Channel: send or receive | The arrow shows the direction of data. `ch <- x` sends, `x := <-ch` receives. You'll meet it in the Telegram bot. | `ch := make(chan string)`; `go func() { ch <- "hi" }()`; `msg := <-ch` |
| `any` | Any type | Alias for `interface{}`, the empty interface every type satisfies. Use it sparingly. | `func Print(v any)` |

## 2. Variables and types

```go
var retries int              // zero value: 0
var name string              // zero value: ""
model := "claude-sonnet-5"   // Go infers the type string

files := []string{"main.go", "llm.go"}   // slice = dynamic list
files = append(files, "sign.go")         // append returns a NEW slice

env := map[string]string{"SAYCUT_MODEL": "claude-sonnet-5"}
m, ok := env["SAYCUT_MODEL"]              // ok == false if the key is missing
```

> **Zero values.** Go has no "undefined". Every variable starts with a sensible value:
> `0`, `""`, `false`, or `nil` for pointers, maps and slices. So you don't have to
> initialise everything by hand.

Types are static and Go converts nothing silently. An env variable is always a `string`,
and you convert it to a number explicitly with `strconv.Atoi`, which may return an error.
This is typical Go: what can fail is visible right in the function signature.

Functions can return **multiple values**. All of error handling is built on this:

```go
n, err := strconv.Atoi("3")    // n == 3, err == nil
n, err = strconv.Atoi("tri")   // n == 0, err != nil
```

**Quiz:** What is the value of the variable after `var count int`?
`undefined` / `nil` / `0` / the compiler reports an error

<details><summary>Answer</summary>

`0`. The zero value for `int` is 0. Nothing in Go is "undefined".
</details>

## 3. Structs instead of classes

A **struct** is just a bag of named fields. No constructor, no inheritance, no `this`. You
add behaviour with methods, and a "constructor" is an ordinary function that by
convention is called `NewX` or `LoadX`.

**config.go**

```go
package main

import (
	"errors"
	"fmt"
	"os"
	"strconv"
)

type Config struct {
	APIKey     string
	Model      string
	MaxRetries int
	OutDir     string
}

func LoadConfig() (Config, error) {
	cfg := Config{
		APIKey:     os.Getenv("ANTHROPIC_API_KEY"),
		Model:      os.Getenv("SAYCUT_MODEL"),
		MaxRetries: 3,
		OutDir:     "dist",
	}
	if cfg.APIKey == "" {
		return Config{}, errors.New("ANTHROPIC_API_KEY is not set")
	}
	if cfg.Model == "" {
		return Config{}, errors.New("SAYCUT_MODEL is not set")
	}
	if v := os.Getenv("SAYCUT_MAX_RETRIES"); v != "" {
		n, err := strconv.Atoi(v)
		if err != nil {
			return Config{}, fmt.Errorf("SAYCUT_MAX_RETRIES must be a number: %w", err)
		}
		cfg.MaxRetries = n
	}
	return cfg, nil
}
```

- **Uppercase means public.** Other packages can see `Config` and `APIKey`, but not
  `apiKey`. Instead of `public` and `private`, Go decides by the case of the first letter.
- `Config{APIKey: ...}` is a struct literal. Fields you don't fill get their zero value.
- `if v := ...; v != ""` is an if with a short initialiser. The variable `v` exists only
  inside that if.
- On error we return `Config{}`, an empty struct, plus the error. The caller looks at
  the error first.

You add a method with a **receiver**, a parameter in front of the function name:

```go
func (c Config) Summary() string {
	return fmt.Sprintf("%s, %d attempts", c.Model, c.MaxRetries)
}
// usage: cfg.Summary()
```

**Quiz:** A struct has a field `apiKey`. Can another package see it?
Yes / No, lowercase means unexported / Only through a pointer

<details><summary>Answer</summary>

No. Visibility is decided by the first letter. `apiKey` is private to the package,
`APIKey` would be public.
</details>

## 4. Pointers: & and *

When you pass a value to a function in Go, the function gets a **copy**, like a
photocopy of a sheet of paper. A pointer, on the other hand, is a **link to a Google
Doc**: whoever has it edits the original.

- `&cfg` means "give me a link to cfg", i.e. its address.
- `*Config` in a type means "link to a Config".
- `*p` in an expression means "open the link", i.e. the value at that address. For struct
  fields Go does it for you, so you write `p.Model`, not `(*p).Model`.

```go
func setModel(c Config)   { c.Model = "haiku" }  // changes the copy
func setModelP(c *Config) { c.Model = "haiku" }  // changes the original

cfg := Config{Model: "sonnet"}
setModel(cfg)     // cfg.Model is still "sonnet"
setModelP(&cfg)   // cfg.Model is now "haiku"
```

What happens in each call:

| Call | Inside the function | `cfg` afterwards |
|---|---|---|
| `setModel(cfg)` | `c` is a photocopy; its Model changes to `"haiku"`, then the copy disappears when the function ends | `"sonnet"` |
| `setModelP(&cfg)` | `c = 0xc000010030` (the address of cfg); a change through the link is a change to the original | `"haiku"` |

**When to use a pointer:** when you want to change the original, when the struct is big,
or when it's something shared like an HTTP client. For your stateful types (`*LLM`,
`*Pipeline`) use a pointer receiver. Small value things like `Config` are fine to copy.

**Quiz:** Why does `json.Unmarshal(raw, &out)` need the `&`?
It's faster / Unmarshal has to write into your variable, not a copy / Because out is a string

<details><summary>Answer</summary>

Unmarshal has to write into your variable. Without `&` the function would get a copy and
fill that. Your variable would stay empty.
</details>

## 5. Errors without exceptions

Go has no `try/catch`. An error is an ordinary value of type `error` that a function
returns last. You'll see this pattern a hundred times:

```go
out, err := doSomething()
if err != nil {
	return fmt.Errorf("do something: %w", err)
}
```

Each layer adds context and passes the error up. `%w` ("wrap") wraps the original error
instead of throwing it away. At the end you get the whole story:

```
gave up after 3 attempts: cherri failed: exit status 1
line 4: unknown action "sendNotifcation"
```

> At first glance it looks like a lot of repetition. The upside is that every place
> where the program can fail is visible right in the code, with no invisible jumps. For
> the explicit style you like in IaC, it's basically ideal.

`errors.Is(err, os.ErrNotExist)` can find a specific error even inside a wrapped chain.
`panic` exists, but only for situations where the program is completely broken. Don't use
it for ordinary errors.

**Quiz:** What's the difference between `%w` and `%v` in `fmt.Errorf`?
None / `%w` keeps the original error inside, so `errors.Is` works / `%w` is only for warnings

<details><summary>Answer</summary>

`%w` keeps the original error inside. `%v` turns the error into text; `%w` wraps it, so
you can search the chain later.
</details>

## 6. Talking to Claude

You don't need an SDK. An HTTP POST with a JSON body is enough, and it lets you practise
structs, tags and pointers all at once.

**llm.go**

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"time"
)

type Message struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

type apiRequest struct {
	Model     string    `json:"model"`
	MaxTokens int       `json:"max_tokens"`
	System    string    `json:"system"`
	Messages  []Message `json:"messages"`
}

type apiResponse struct {
	Content []struct {
		Type string `json:"type"`
		Text string `json:"text"`
	} `json:"content"`
}

type LLM struct {
	apiKey string
	model  string
	client *http.Client
}

func NewLLM(cfg Config) *LLM {
	return &LLM{
		apiKey: cfg.APIKey,
		model:  cfg.Model,
		client: &http.Client{Timeout: 90 * time.Second},
	}
}

func (l *LLM) Ask(ctx context.Context, system string, msgs []Message) (string, error) {
	body, err := json.Marshal(apiRequest{
		Model: l.model, MaxTokens: 4096, System: system, Messages: msgs,
	})
	if err != nil {
		return "", fmt.Errorf("encode request: %w", err)
	}

	req, err := http.NewRequestWithContext(ctx, http.MethodPost,
		"https://api.anthropic.com/v1/messages", bytes.NewReader(body))
	if err != nil {
		return "", fmt.Errorf("build request: %w", err)
	}
	req.Header.Set("x-api-key", l.apiKey)
	req.Header.Set("anthropic-version", "2023-06-01")
	req.Header.Set("content-type", "application/json")

	resp, err := l.client.Do(req)
	if err != nil {
		return "", fmt.Errorf("call API: %w", err)
	}
	defer resp.Body.Close()

	raw, err := io.ReadAll(resp.Body)
	if err != nil {
		return "", fmt.Errorf("read response: %w", err)
	}
	if resp.StatusCode != http.StatusOK {
		return "", fmt.Errorf("API returned %d: %s", resp.StatusCode, raw)
	}

	var out apiResponse
	if err := json.Unmarshal(raw, &out); err != nil {
		return "", fmt.Errorf("decode response: %w", err)
	}
	for _, c := range out.Content {
		if c.Type == "text" {
			return c.Text, nil
		}
	}
	return "", errors.New("API response has no text")
}
```

- **`` `json:"max_tokens"` `` is a struct tag.** It tells the `encoding/json` package what
  the field is called in JSON. In Go it's `MaxTokens`, in the API `max_tokens`.
- `apiRequest` is lowercase because nobody outside the package needs it. Its fields are
  uppercase though, because `json` is another package and wouldn't see lowercase fields.
- `Content []struct{...}` is an **anonymous struct**: a data shape you need in only one
  place and don't have to name.
- `json.Unmarshal(raw, &out)` gets a pointer because it has to *write into your variable*.
  With a copy it would write into thin air. This is chapter 4 in practice.
- `defer resp.Body.Close()` runs when the function exits, whichever `return` it leaves
  through. The cleanup is written right next to the opening.
- `ctx` (context) carries a timeout and cancellation. When it expires, the request is
  aborted. By convention in Go you pass it as the first parameter.
- `for _, c := range` iterates over a slice. `_` discards the index you don't need.

## 7. Running other programs

Cherri and `shortcuts` are CLI tools. Go runs them through `os/exec`, much as if you ran
them in a terminal.

**cherri.go**

```go
package main

import (
	"context"
	"fmt"
	"os/exec"
	"strings"
)

// Compile returns the path of the compiled .shortcut.
// On failure the error contains cherri's output, so the LLM can fix its code.
func Compile(ctx context.Context, srcPath string) (string, error) {
	out, err := exec.CommandContext(ctx, "cherri", srcPath).CombinedOutput()
	if err != nil {
		return "", fmt.Errorf("cherri failed: %w\n%s", err, out)
	}
	// ASSUMPTION: foo.cherri -> foo.shortcut. Verify in phase 0.
	return strings.TrimSuffix(srcPath, ".cherri") + ".shortcut", nil
}
```

**sign.go**

```go
package main

import (
	"context"
	"fmt"
	"os/exec"
)

func Sign(ctx context.Context, in, out string) error {
	cmd := exec.CommandContext(ctx, "shortcuts", "sign",
		"--mode", "anyone", "--input", in, "--output", out)
	if b, err := cmd.CombinedOutput(); err != nil {
		return fmt.Errorf("shortcuts sign failed: %w\n%s", err, b)
	}
	return nil
}
```

- `CombinedOutput()` returns stdout and stderr together as `[]byte`, i.e. raw bytes. `%s`
  prints them as text.
- We put the compiler output into the error on purpose. Exactly this text goes back to
  Claude as "fix this".
- Arguments are passed as separate strings, not as one shell command, so there's no risk
  of shell injection.

> **Watch out:** if phase 0 shows that cherri signs by itself on macOS, you delete
> `sign.go` and the pipeline gets one step shorter.

## 8. Pipeline and the retry loop

This is the core of the whole app, exactly what you saw in the simulation at the top.
When compilation fails, we add the error to the conversation and try again.

**pipeline.go**

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"os"
	"path/filepath"
	"strings"
)

type Pipeline struct {
	cfg    Config
	llm    *LLM
	system string
}

func (p *Pipeline) Run(ctx context.Context, name, prompt string) (string, error) {
	msgs := []Message{{Role: "user", Content: prompt}}
	var lastErr error

	for attempt := 1; attempt <= p.cfg.MaxRetries; attempt++ {
		slog.Info("generating", "name", name, "attempt", attempt)

		src, err := p.llm.Ask(ctx, p.system, msgs)
		if err != nil {
			return "", err // API is down: retrying compile won't help
		}
		src = stripFences(src)

		srcPath := filepath.Join(p.cfg.OutDir, name+".cherri")
		if err := os.WriteFile(srcPath, []byte(src), 0o644); err != nil {
			return "", fmt.Errorf("write source: %w", err)
		}

		unsigned, err := Compile(ctx, srcPath)
		if err != nil {
			lastErr = err
			slog.Warn("compile failed", "attempt", attempt, "err", err)
			msgs = append(msgs,
				Message{Role: "assistant", Content: src},
				Message{Role: "user", Content: "Compilation failed:\n" + err.Error() +
					"\nFix it. Reply with Cherri code only."},
			)
			continue
		}

		signed := filepath.Join(p.cfg.OutDir, name+".signed.shortcut")
		if err := Sign(ctx, unsigned, signed); err != nil {
			return "", err
		}
		slog.Info("done", "file", signed, "attempts", attempt)
		return signed, nil
	}
	return "", fmt.Errorf("gave up after %d attempts: %w", p.cfg.MaxRetries, lastErr)
}

// stripFences removes ```cherri ... ``` if the model wraps its answer.
func stripFences(s string) string {
	s = strings.TrimSpace(s)
	s = strings.TrimPrefix(s, "```cherri")
	s = strings.TrimPrefix(s, "```")
	s = strings.TrimSuffix(s, "```")
	return strings.TrimSpace(s)
}
```

- `for attempt := 1; attempt <= N; attempt++` is the classic three-part loop, the same
  as in C.
- `[]Message{{Role: ...}}` is a slice with one struct. The inner `{}` is the struct and
  the outer one the slice. Go infers the type inside.
- The conversation grows through `append`: Claude sees its original code and the error,
  which is why it can fix it precisely.
- `continue` jumps to the next attempt. `lastErr` remembers the last error for the final
  message.
- `0o644` is an octal number, the same permissions as `chmod 644`.
- `slog` logs in a structured way, as key–value pairs. Later you switch it to JSON output
  with one line.

## 9. main.go and the first run

**main.go**

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"os"
	"time"
)

func main() {
	if err := run(); err != nil {
		slog.Error("saycut failed", "err", err)
		os.Exit(1)
	}
}

func run() error {
	if len(os.Args) < 3 {
		return fmt.Errorf(`usage: saycut <name> "<what the shortcut should do>"`)
	}
	name, prompt := os.Args[1], os.Args[2]

	cfg, err := LoadConfig()
	if err != nil {
		return err
	}
	system, err := os.ReadFile("prompts/system.md")
	if err != nil {
		return fmt.Errorf("read system prompt: %w", err)
	}
	if err := os.MkdirAll(cfg.OutDir, 0o755); err != nil {
		return fmt.Errorf("create out dir: %w", err)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Minute)
	defer cancel()

	p := &Pipeline{cfg: cfg, llm: NewLLM(cfg), system: string(system)}
	path, err := p.Run(ctx, name, prompt)
	if err != nil {
		return err
	}
	fmt.Println(path)
	return nil
}
```

> **Why `main` + `run`?** `os.Exit` ends the program immediately and `defer` no longer
> runs. That's why all the logic lives in `run()`, which returns an error, and `main`
> only decides the exit code. You get one exit point and all defers run.

Running it:

```sh
export ANTHROPIC_API_KEY=sk-ant-...
export SAYCUT_MODEL=claude-sonnet-5
go run . domov "When I get home, send me a Welcome home notification"
# dist/domov.signed.shortcut
open dist/domov.signed.shortcut      # import dialog
```

**Makefile**

```make
build:
	go build -o bin/saycut .
test:
	go test ./...
lint:
	go vet ./...
```

## 10. Interfaces and the first test

An interface in Go is a **list of methods**. Every type that has those methods satisfies
it *automatically*, without `implements`. It's duck typing checked by the compiler.

```go
type Generator interface {
	Ask(ctx context.Context, system string, msgs []Message) (string, error)
}

type Pipeline struct {
	cfg    Config
	llm    Generator   // instead of *LLM
	system string
}
```

`*LLM` already has the `Ask` method, so you don't have to change anything else. In tests
you slip it a fake generator that doesn't call the API or spend any tokens. Later you do
the same with a `Compiler` interface and test the whole retry logic without cherri.

The first test, in Go's "table-driven" style:

**pipeline_test.go**

```go
package main

import "testing"

func TestStripFences(t *testing.T) {
	tests := []struct {
		name, in, want string
	}{
		{"plain", "a", "a"},
		{"fenced", "```cherri\na\n```", "a"},
		{"spaces", "  a  ", "a"},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := stripFences(tt.in); got != tt.want {
				t.Errorf("got %q, want %q", got, tt.want)
			}
		})
	}
}
```

A file ending in `_test.go` is compiled only for `go test`. The table of cases is a slice
of anonymous structs: you add a new case with one line.

**Quiz:** How do you say that `*LLM` implements `Generator`?
`implements Generator` / You don't; it's enough that it has an `Ask` method with the right signature / With a struct tag

<details><summary>Answer</summary>

You don't. Interfaces in Go are satisfied implicitly. The compiler checks it by itself
when you assign `*LLM` to a field of type `Generator`.
</details>

## 11. What's next: Telegram bot

Once the CLI works end to end, the bot is just another input into the same pipeline.
You'll get your first external dependency (a Telegram library via `go get`) and a new
concept Go is best known for:

```go
for update := range updates {   // updates is a channel
	go handle(update)          // each message in its own goroutine
}
```

- A **goroutine** (`go f()`) is a very cheap concurrent thread. While one message waits
  20 seconds for Claude, the bot answers the next.
- A **channel** (`chan`, `<-`) is a pipe goroutines use to send each other data. The
  library hands you new messages through one.
- Before the bot, don't forget a chat ID whitelist and sanitising `name`, because you
  build a file path from it.
- When `package main` outgrows itself, split it into `internal/llm`,
  `internal/pipeline` etc. and add `cmd/saycut` and `cmd/bot`. That's where you learn
  packages and imports.

The approach: write it yourself following this guide first, and use Claude Code only as
a reviewer. The other way round wouldn't teach you Go.
