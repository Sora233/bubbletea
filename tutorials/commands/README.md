Commands in Bubble Tea
======================

This is the second tutorial for Bubble Tea covering commands, which deal with
I/O. The tutorial assumes you have a working knowlege of Go and a decent
understanding of [the first tutorial][basics].

You can find the non-annotated version of this program [on GitHub][source].

[basics]: http://github.com/charmbracelet/bubbletea/tree/master/tutorials/basics
[source]: https://github.com/charmbracelet/bubbletea/master/tutorials/commands

## Let's Go!

For this tutorial we're building a very simple program that makes an HTTP
request to a server over HTTP and reports the status code of the response.

We'll import a few necessary packages and put the URL we're going to check in
a `const`.

```go
    package main

    import (
        "fmt"
        "net/http"
        "os"
        "time"

        tea "github.com/charmbracelet/bubbletea"
    )

    const url = "https://charm.sh/"
```

## The Model

Next we'll define our model. The only things we need to store are the HTTP
response and a possible error.

```go
    type model struct {
        status int
        err    error
    }
```

## Commands and Messages

`Cmd`s are functions that perform some I/O and then return a `Msg`. Checking the
time, ticking a timer, reading from the disk, and network stuff are all I/O and
should be run through commands. That might sound harsh, but it will keep your
Bubble Tea program staightforward and simple.

Anyway, let's write a `Cmd` that makes a request to a server and returns the
result as a `Msg`.

```go
    func checkServer() tea.Msg {

        // Create an HTTP client and make a GET request.
        c := &http.Client{Timeout: 10 * time.Second}
        res, err := c.Get(url)

        if err != nil {
            // There was an error making our request. Wrap the error we received
            // in a message and return it.
            return errMsg(err)
        }
        // We received a response from the server. Return the HTTP status code
        // as a message.
        return statusMsg(res.StatusCode)
    }

    type statusMsg int
    type errMsg error
```

And notice that we've defined two new `Msg` types. They can be any type, even a
struct. We'll come back to them later later in our update function.
First, let's write our initialization function.

## The Initialization Function

The initilization function is incredibly simple. We return an empty model and
fire off the `Cmd` we made earlier.

```go
    func initialize() (tea.Model, tea.Cmd) {
        return model{}, checkServer
    }
```

## The Update Function

Internally, `Cmd`s run asynchronously in a goroutine. The `Msg` they return is
collected and sent to our update function for handling. Remember those message
types we made earlier when we were making the `checkServer` command? We handle
them here. This makes dealing with many asynchronous operations very easy.

```go
    func update(msg tea.Msg, mdl tea.Model) (tea.Model, tea.Cmd) {
        m, _ := mdl.(model)

        switch msg := msg.(type) {

        case statusMsg:
            // The server returned a status message. Save it to our model. Also
            // exit because we have nothing else to do.
            m.status = int(msg)
            return m, tea.Quit

        case errMsg:
            // There was an error. Note it in the model. And exit the program.
            m.err = msg
            return m, tea.Quit

        case tea.KeyMsg:
            // Ctrl+c quits. Even with short running programs it's good to have
            // an quit key, just incase your logic is off. Users will be very
            // annoyed if they can't exit.
            if msg.Type == tea.KeyCtrlC {
                return m, tea.Quit
            }
        }

        // If we happen to get any other messages, don't do anything.
        return m, nil
    }
```

## The View Function

Our view is very straightforward. We look at the current model and build a
string accordingly:

```go
    func view(mdl tea.Model) string {
        m, _ := mdl.(model)

        // If there's an error, print it out and don't do anything else.
        if m.err != nil {
            return fmt.Sprintf("\nWe had some trouble: %v\n\n", m.err)
        }

        // Tell the user we're doing something.
        s := fmt.Sprintf("Checking %s ... ", url)

        // When the server responds with a status, add it to the current line.
        if m.status > 0 {
            s += fmt.Sprintf("%d %s!", m.status, http.StatusText(m.status))
        }

        // Send off whatever we came up with above for rendering.
        return "\n" + s + "\n\n"
    }
```

## Run the program

The only thing left to do is run the program, so let's do that!

```go
    func main() {
        if err := tea.NewProgram(initialize, update, view).Start(); err != nil {
            fmt.Printf("Uh oh, there was an error: %v\n", err)
            os.Exit(1)
        }
    }
```

And that's that! There's one more thing you should is helpful to know about
`Cmd`s, though.

## One More Thing About Commands

`Cmd`s are defined in Bubble Tea as `type Cmd func() Msg`. So they're just
functions that don't take any arguments and return a `Msg`, which can be
anything. If you need to pass arguments to a command, you just make a function
that returns a command. For example:

```go

    func checkSomeUrl(url string) tea.Cmd {
        return func() tea.Msg {
            c := &http.Client{Timeout: 10 * time.Second}
            res, err := c.Get(url)
            if err != nil {
                return errMsg(err)
            }
            return statusMsg(res.StatusCode)
        }
    }

```

Just make sure you do as much work as you can in the innermost function, because
that's the one that runs asynchronously.

## Anyway, Now What?

After doing this tutorial and [the previous one][basics] you should be ready
to build a Bubble Tea program of your own.

We also recommend that you look at the Bubble Tea [example programs][examples]
as well as [Bubbles][bubbles], a component library for Bubble Tea.

And, of course, check out the [Go Docs][docs].

### Bubble Tea in the Wild

For some Bubble Tea programs in production, see:

* [Glow](https://github.com/charmbracelet/glow): a markdown reader, browser and online markdown stash
* [The Charm Tool](https://github.com/charmbracelet/charm): the Charm user account manager

[examples]: http://github.com/charmbracelet/bubbletea/tree/master/examples
[docs]: https://pkg.go.dev/github.com/charmbracelet/glow?tab=doc
[bubbles]: https://github.com/charmbracelet/bubbles

### Libraries we use with Bubble Tea

* [Bubbles][bubbles] various Bubble Tea components we've built
* [Termenv][termenv]: Advanced ANSI styling for terminal applications
* [Reflow][reflow]: ANSI-aware methods for reflowing blocks of text
* [go-runewidth][runewidth]: Get the physical width of strings in terms of terminal cells. Many runes, such as East Asian charcters and emojis, are two cells wide, so measuring a layout with `len()` often won't cut it!

[termenv]: https://github.com/muesli/termenv
[reflow]: https://github.com/muesli/reflow
[bubbles]: https://github.com/charmbracelet/bubbles
[runewidth]: https://github.com/mattn/go-runewidth

### Feedback

We'd love to hear your thoughts on this tutorial. Feel free to drop us a note!

* [Twitter](https://twitter.com/charmcli)
* [The Fediverse](https://mastodon.technology/@charm)