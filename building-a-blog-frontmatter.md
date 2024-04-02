+++
title = "Building a Blog in Go: Metadata via Frontmatter"
description = "My Go blog has a way to render individual posts, but it is still missing metadata about each post. Information like the author of the post, when it was published, etc. In this part of the build a blog series we focus on adding frontmatter so that we can properly render this for each blog post, but also in preparation for when we want to show a list of all of our blog posts."
date = 2024-04-01

[author]
name = "Jon Calhoun"
email = "jon@calhoun.io"
+++

I have my application where it can render each blog post using a layout, but I need to get some additional information about each blog post to properly use the layout. To do this I am going to use frontmatter.

Frontmatter is basically just a section at the start of a markdown file that will provide some metadata. It can often be provided in a variety fo formats including YAML, TOML, and JSON. I am going to use TOML simply because it is what my current blog uses, so it is what I am familiar with right now.

The first thing I did was add some metadata to the top of each of my test blog posts. TOML frontmatter is typically indicated with the `+++` characters, and then the TOML is written much like a block of code inside the characters. Below is an example of this on the `how-to-boil-eggs.md` demo blog post I created.

```md
+++
title = "How to Boil Eggs"
description = "Learn how to quickly and easily boil 6 eggs!"
date = 2024-03-03

[author]
name = "Jon Calhoun"
email = "jon@calhoun.io"
+++

# How to boil eggs

Ingredients: 6 eggs

Step 1: Boil 4 cups of water.
Step 2: Add the eggs to the water and boil for 10 minutes.
Step 3: Rinse eggs under cold water.
```

I added the main things I need right now, which include the author of the post and the title of the post. I also added a few pieces of information that I suspect I'll want later on, such as a short description of the post and the date. These two will be useful when showing a list of all of my blog posts.

Next I started to look for ways to parse the frontmatter. One option is to find or create an extension for [goldmark](https://github.com/yuin/goldmark), my markdown library. Another option is to look for a standalone library that will parse the frontmatter and give me the remainder of the markdown file which I can then provide to goldmark to process.

I found libraries for both options, but ultimately decided to go with a standalone library named [frontmatter](https://github.com/adrg/frontmatter) because it felt easier and more intuitive to use. Your mileage may vary, so feel free to check out other options.

Now that I have a library picked I installed it.

```bash
go get github.com/adrg/frontmatter
```

I then proceeded to create a struct to represent a blog post. This replaced the `PostData` type I used temporarily in the last part of the exercise, and it has `toml` tags in it so I can more easily parse the frontmatter.

```go
type Post struct {
	Title   string `toml:"title"`
	Slug    string `toml:"slug"`
	Content template.HTML
	Author  Author `toml:"author"`
}

type Author struct {
	Name  string `toml:"name"`
	Email string `toml:"email"`
}
```

With the type created, I updated the `PostHandler` function to parse the frontmatter into a `Post`.

```go
func PostHandler(sl SlugReader) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var post Post
		post.Slug = r.PathValue("slug")
		postMarkdown, err := sl.Read(post.Slug)
		if err != nil {
			// TODO: Handle different errors in the future
			http.Error(w, "Post not found", http.StatusNotFound)
			return
		}
		rest, err := frontmatter.Parse(strings.NewReader(postMarkdown), &post)
		if err != nil {
			http.Error(w, "Error parsing frontmatter", http.StatusInternalServerError)
			return
		}
    // ...
  }
}
```

At this point one thing I do want to note is that our `Read` method is returning a string, and then we immediately use `strings.NewReader` to turn this into an `io.Reader`. What this means is that we could have likely just had our `SlugReader` interface return an `io.Reader` instead of a string, which I may do later, but for exercises like this I'll often start with a string out of simplicity and familiarity, then adapt later once I know what the best fit is.

From here I proceeded to update the `PostHandler` function to use the remainder of the markdown file's contents when converting it to HTML and then I passed the `post` variable with all of its data into the template.

```go
func PostHandler(sl SlugReader) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// ...

		err = mdRenderer.Convert(rest, &buf)
		if err != nil {
			http.Error(w, "Error converting markdown", http.StatusInternalServerError)
			return
		}
    // ... unchanged
		post.Content = template.HTML(buf.String())
		err = tpl.Execute(w, post)
	}
}
```

Due to a change in how I structured the `Author` field, I needed to also update my template quickly. The only part that changed was in the `<div>` that contained the blog post, so I'll show that below.

```html
<div class="container mx-auto">
  <h1 class="text-4xl font-bold text-center">{{.Title}}</h1>
  {{with .Author}}
  <div class="text-center mt-4">
    <p class="text-gray-500">Author: <a href="mailto:{{.Email}}">{{.Name}}</a></p>
  </div>
  {{end}}
  <div class="prose max-w-full">
    {{.Content}}
  </div>
</div>
```

My blog posts were showing the title twice at this point, so my final task was to remove the `#` title piece from each blog post. This seems to be pretty standard with a lot of blog platforms and themes, but if we really wanted to keep the title heading inside of our markdown we could likely figure out a way to extend goldmark to ignore the first `h1` heading in each file. I won't be doing that, but feel free to explore goldmark a bit to see how you might do it if you want!
