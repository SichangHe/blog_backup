# What Does an mdBook Preprocessor Do—Celebrating mdBook-KaTeX v0.5.1

With [the release of mdBook-KaTeX v0.5.1](https://github.com/lzanini/mdbook-katex/releases/tag/v0.5.1), I have finally cleaned up the mess it was in.

- The fake KaTeX renderer support is finally dropped in v0.5.0, specifying `[output.katex]` in `book.toml` now results in an error as opposed to a deprecation warning.
- Tokio has been removed as a dependency. In stead, we use Rayon for the parallelism.

But, first, what does [mdBook-KaTeX](https://github.com/lzanini/mdbook-katex) do? It is an mdBook preprocessor that pre-renders math expressions. For example:

```markdown
Define $f(x)$:

$$
f(x)=x^2\\
x\in\R
$$
```

would be pre-rendered as:

```markdown
Define <span class="katex"><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mord mathnormal" style="margin-right:0.10764em;">f</span><span class="mopen">(</span><span class="mord mathnormal">x</span><span class="mclose">)</span></span></span></span>:

<span class="katex-display"><span class="katex"><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mord mathnormal" style="margin-right:0.10764em;">f</span><span class="mopen">(</span><span class="mord mathnormal">x</span><span class="mclose">)</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.8641em;"></span><span class="mord"><span class="mord mathnormal">x</span><span class="msupsub"><span class="vlist-t"><span class="vlist-r"><span class="vlist" style="height:0.8641em;"><span style="top:-3.113em;margin-right:0.05em;"><span class="pstrut" style="height:2.7em;"></span><span class="sizing reset-size6 size3 mtight"><span class="mord mtight">2</span></span></span></span></span></span></span></span></span><span class="mspace newline"></span><span class="base"><span class="strut" style="height:0.5782em;vertical-align:-0.0391em;"></span><span class="mord mathnormal">x</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">∈</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.6889em;"></span><span class="mord mathbb">R</span></span></span></span></span>
```

before mdBook-KaTeX sends it back to mdBook.

It might look a bit scary, but this is what all HTML-based math renderer do—generate a load of nested tags, and it enables the expressions to look nice in a browser. Most renderers just do this after users load the webpage.

In this article, however, I want to focus on the other side of mdBook-KaTeX instead—the mdBook preprocessor side.

What does an mdBook preprocessor do? Well, in a nutshell, [mdBook preprocessors](https://rust-lang.github.io/mdBook/format/configuration/preprocessors.html) are used to customize [mdBook](https://github.com/rust-lang/mdBook/tree/master), the static site generator used to render [*The Rust Programming Language*](https://doc.rust-lang.org/book/). It does so by manipulating the loaded book data and passing it back to mdBook.

This sounds abstract, though, so let's dive into what mdBook-KaTeX does, with code, as a concrete example.

## What does mdBook-KaTeX do as an mdBook preprocessor

mdBook-KaTeX is, firstly, a Command Line Interface (CLI) App written in Rust. It uses [Clap](https://github.com/clap-rs/clap) to parse the arguments passed in:

```sh
$ mdbook-katex --help
A preprocessor that renders KaTex equations to HTML.

Usage: mdbook-katex [COMMAND]

Commands:
  supports  Check whether a renderer is supported by this preprocessor
  help      Print this message or the help of the given subcommand(s)

Options:
  -h, --help     Print help
  -V, --version  Print version
```

As we can see, mdBook-KaTeX only takes one command—`supports`. But, if no command is specified, it reads from StdIn:

```sh
$ mdbook-katex --help
# (Nothing happens).
# (Press control + D).
Error: Unable to parse the input

Caused by:
    EOF while parsing a value at line 1 column 0
```

We don't use the `mdbook-katex` command directly, though. Instead, mdBook would invoke it when it builds the book.

### The `supports` command

All preprocessors need to have this command so mdBook can check whether it supports a renderer.

Usually, we use mdBook to render Markdown into HTML with the `html` renderer. So, after loading the book from disk, mdBook invokes mdBook-KaTeX like this:

```sh
mdbook-katex supports html
```

In this case, mdBook-KaTeX would just output nothing with status code 0 to indicate that we support the `html` renderer.

### Reading and processing the book data from StdIn

If no command is specified, mdBook-KaTeX should read the book data as JSON from StdIn.

```rust
let pre = KatexProcessor;
let (ctx, book) = CmdPreprocessor::parse_input(io::stdin())?;
let processed_book = pre.run(&ctx, book)?;
serde_json::to_writer(io::stdout(), &processed_book)?;
```

Here, we read the context `ctx: PreprocessorContext` and the book data `book: Book` from StdIn using `mdbook::preprocess::cmd::CmdPreprocessor`, run it through our preprocessor `pre` and get the `processed_book: Book`, and print it back out to StdOut, where mdBook would catch it and use the book.

So far, the process above is basically universal for any mdBook preprocessors. Yes, you can copy the code from `main.rs` from mdBook-KaTeX and start your own preprocessor. The only change to use other preprocessors would be replacing the `KatexProcessor` with another `struct` that implements `mdbook::preprocess::Preprocessor`:

```rust
pub trait Preprocessor {
    fn name(&self) -> &str;
    fn run(&self, ctx: &PreprocessorContext, book: Book) -> Result<Book>;
    fn supports_renderer(&self, renderer: &str) -> bool;
}
```

`name` and `supports_renderer` are trivial, but `run` is where the fun lives. For `KatexProcessor`, it finds the math expressions in each chapter of `book` and render them.

### Processing `book`

<!-- TODO: -->
