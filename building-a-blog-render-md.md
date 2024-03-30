# Building a Blog in Go: Rendering Markdown

## Converting Markdown to HTML

At this point my blog is reading files from disk and rendering their markdown content as plain text. My next step is going to be converting the markdown to proper HTML using some sort of markdown library.

After looking at a few library options, I decided to use [goldmark](https://github.com/yuin/goldmark). The main reason for this choice is that it is designed to be extensible. This means I can add custom markdown syntax to my posts and update the parser to handle them correctly, and it sounded like a fun thing to try out since I very commonly add `<aside>` blocks to my write-ups that will have additional optional information, like links to related resources or common bugs users encounter and how to address them.

I first installed the goldmark library.

```bash
go get github.com/yuin/goldmark
```

Then I updated my code to convert the markdown into HTML. This is done using the [Convert](https://pkg.go.dev/github.com/yuin/goldmark#Convert) function provided by goldmark. Finally, I copy the HTML to the `http.ResponseWriter`.

```go
func PostHandler(sl SlugReader) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		slug := r.PathValue("slug")
		postMarkdown, err := sl.Read(slug)
		var buf bytes.Buffer
		err = goldmark.Convert([]byte(postMarkdown), &buf)
		if err != nil {
			http.Error(w, "Error converting markdown", http.StatusInternalServerError)
			return
		}
		if err != nil {
			// TODO: Handle different errors in the future
			http.Error(w, "Post not found", http.StatusNotFound)
			return
		}
		io.Copy(w, &buf)
	}
}
```

Now if I restart my Go server and visit a page like `/posts/io-reader` I will see the markdown being rendered as HTML.

## Syntax Highlighting

Next I wanted to make my code look a little better. Most notably, I would like to use syntax highlighting. This is anther reason I opted to use goldmark - it has a highlighting extension already built, so I just needed to install it and use it.

```bash
go get github.com/yuin/goldmark-highlighting/v2
```

Auto-imports for this library didn't always work for me, but the import I used is shown below.

```go
import (
	"bytes"
	"io"
	"log"
	"net/http"
	"os"

	"github.com/yuin/goldmark"
	highlighting "github.com/yuin/goldmark-highlighting/v2"
)
```

With the import in place I updated my PostHandler function to use the extension. To do this, I needed to stop using the goldmark's Convert function, and instead use the [New](https://pkg.go.dev/github.com/yuin/goldmark#New) function. This allows me to create a new markdown converter with my own custom options.

```go
func PostHandler(sl SlugReader) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		slug := r.PathValue("slug")
		postMarkdown, err := sl.Read(slug)
		mdRenderer := goldmark.New(
			goldmark.WithExtensions(
				highlighting.Highlighting,
			),
    )
    var buf bytes.Buffer
		err = mdRenderer.Convert([]byte(postMarkdown), &buf)
		if err != nil {
			http.Error(w, "Error converting markdown", http.StatusInternalServerError)
			return
		}
		if err != nil {
			// TODO: Handle different errors in the future
			http.Error(w, "Post not found", http.StatusNotFound)
			return
		}
		io.Copy(w, &buf)
	}
}
```

The highlighting library supports multiple themes, and I tend to use [Dracula](https://draculatheme.com/) theme, so I opted to use that for my Go blog as well.

```go
mdRenderer := goldmark.New(
  goldmark.WithExtensions(
    highlighting.NewHighlighting(
      highlighting.WithStyle("dracula"),
    ),
  ),
)
```

This is far from perfect, but it is a great start.

## Using a Layout

The next thing I want to focus on is adding a layout to the application. That way I can add links to other pages, add some custom styling, and more. To do this I opted to use Go's `html/template` library.

I started by creating a template file.

```bash
code post.gohtml
```

I plan to provide each post with at least the following information:

- Blog Content (HTML generated from Markdown)
- The Author
- The Post Title (for things like the `<title>` tag)

I also intend to use Tailwind CSS to style everything, so I'm going to include it via their CDN. I also know from past experience that I will want their [Typography](https://github.com/tailwindlabs/tailwindcss-typography) plugin, since this makes it really easy to style generated HTML like I will have from the markdown.

Putting this all together, plus a little copilot magic to speed things up, I generated the following HTML:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <script src="https://cdn.tailwindcss.com?plugins=typography"></script>
  <title>{{.Title}} | Jon's Blog</title>
</head>
<body>
  <nav class="flex items-center justify-between bg-gray-800 p-6 mb-4">
    <div class="flex items-center flex-shrink-0 text-white mr-6">
      <span class="font-semibold text-xl tracking-tight">Jon's Blog</span>
    </div>
    <div class="block">
      <ul class="flex space-x-4">
        <li><a href="#" class="text-gray-300 hover:bg-gray-700 px-3 py-2 rounded">Home</a></li>
        <li><a href="#" class="text-gray-300 hover:bg-gray-700 px-3 py-2 rounded">About</a></li>
        <li><a href="#" class="text-gray-300 hover:bg-gray-700 px-3 py-2 rounded">Blog</a></li>
        <li><a href="#" class="text-gray-300 hover:bg-gray-700 px-3 py-2 rounded">Contact</a></li>
      </ul>
    </div>
  </nav>
  <div class="container mx-auto">
    <h1 class="text-4xl font-bold text-center">{{.Title}}</h1>
    <div class="text-center mt-4">
      <p class="text-gray-500">Author: {{.Author}}</p>
    </div>
    <div class="prose max-w-full">
      {{.Content}}
    </div>
  </div>
</body>
</html>
```

I then updated my PostHandler to use this new template:

```go
func PostHandler(sl SlugReader) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
    // ...

    tpl, err := template.ParseFiles("post.gohtml")
		if err != nil {
			http.Error(w, "Error parsing template", http.StatusInternalServerError)
			return
		}
		err = tpl.Execute(w, PostData{
			Title:   "My First Post",
			Content: template.HTML(buf.String()),
			Author:  "Jon Calhoun",
		})
		// io.Copy(w, &buf)
	}
}
```

This gets my blog looking better, but the title and author are hardcoded. The first `#` in my markdown also conflicts a bit with the title I am rendering in my HTML. To fix both of these problems I am going to add code to parse frontmatter from my blog posts, which will contain various pieces of metadata we might need to render the post. That will have to wait until the next part in this series though!

See the source code so far here: <https://github.com/joncalhoun/jonblog/tree/p2>
