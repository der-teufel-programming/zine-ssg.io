---
{
  "title": "Documentation",
  "date": "2020-07-06T00:00:00",
  "author": "Sample Author",
  "draft": false,
  "layout": "documentation.html",
  "tags": []
}  
--- 
## Warning
Zine is currently alpha software. 

Many features are missing and currently Zine is only capable of building relatively simple websites. 

All the common features (like i18n, an asset system, etc) are planned, but will take some time to be implemented.

Using Zine today means participating in its development process.

## Requirements
Zine is a collection of tools orchestrated by the Zig build system.

Zine only depends on the Zig compiler, see [the official website](https://ziglang.org) for more information on how to download and install Zig.

## Project Setup
Your website only needs the following two files to get started:

***build.zig.zon***
```zig
.{
    .name = "sample website",
    .version = "0.0.0",
    .dependencies = .{
        .zine = .{
            .url = "git+https://github.com/kristoff-it/zine.git#11d7742c5d77fdda6c7b5e99c1947f63a2002d34",
            .hash = "122072392929926ae3f247287f753cfae38d5167c5d1bd251ca6e50f0a4d9b4eb5cb",
        },
    },
    .paths = .{"."},
}
```

***build.zig***
```zig
const std = @import("std");
const zine = @import("zine");

pub fn build(b: *std.Build) !void {
    try zine.addWebsite(b, .{
        .layouts_dir_path = "layouts",
        .content_dir_path = "content",
        .static_dir_path = "static",
        .site = .{
            .base_url = "https://sample.com",
            .title = "Sample Website",
        },
    });
}  
```

Once you create the 3 directories mentioned in your `build.zig` file (`layouts`, `content`, `static`, but they can also be named however you like), you are ready to start working on your website.

### Content
The content directory contains your markdown files.

Zine currently supports the following naming scheme:

- `content/mypage/_index.md` signifies that `mypage` can also have subdirectories.
- `content/mypage/index.md` signifies that `mypage` will not have any other public subdirectory.

### Layouts
The layouts directory contains the html layouts that
will be applied to your content.

See the layouting section for more information about creating layouts and templates.

### Static
The static directory will be copied verbatim into the output directory.


# CLI
#### `$ zig build`
Builds your website and places it in `zig-out`. Pass `-p some/path/` to change the output directory.

Use `zig build --help` for more information about flags supported by `zig build`.

#### `$ zig build serve`
Builds your website and starts the development server. Making changes to any of your input directories (ie content, layouts, static) will automatically trigger a rebuild and a page reload.

Pass `-Dport=8080` to set the listening port to 8080.

# Templating
In Zine templates live under the `layout` directory.

A normal HTML document will be a valid Zine Template.

***`layouts/base.html`***
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>My Blog</title>
  </head>
  <body>
    Hello World!
  </body>
</html>
```
## Extending templates

The first feature of Super templates is extending a template by using the `<super/>` element.

The principle is simple: the super tag defines an insertion point into the *parent* element which will have to be fulfilled by the *Super template* (the template that will extend the current one, what would be called the parent template in OOP lingo).

**NOTE: once a template defines an extension point, it must be moved inside the `layouts/templates/` directory. Only complete templates (called *layouts*) can live directly in `layouts/`.**

***`layouts/templates/base.html`***
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title id="title"><super/> - My Blog</title>
  </head>
  <body>
    Hello World!
  </body>
</html>
```

This template can then be finalized by a *layout* like so:

***`layouts/homepage.html`***
```html
<extend template="base.html"/>

<title id="title">Homepage</title>
```

This will result in the following generated output:

***`zig-out/homepage.html`***
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title id="title">Homepage - My Blog</title>
  </head>
  <body>
    Hello World!
  </body>
</html>
```

## Extension chains
Extension chains can be longer than two, and can involve multiple templates. Here's a more complex example.


***`layouts/templates/base.html`***
```html
<!DOCTYPE html>
<html>
  <head id="head">
    <meta charset="UTF-8">
    <title id="title"><super/> - My Blog</title>
    <super/>
  </head>
  <body id="body">
    <super/>
  </body>
</html>
```
***`layouts/templates/with-menu.html`***
```html
<extend template="base.html"/>

<title id="title"><super/></title>

<head id="head">
  <script>console.log("Hello World!");</script>
</head>

<!-- at the top level only comments 
     and block definitions are allowed -->
<body id="body">
    <nav>
        <a>Home</a>
        <a>About</a>
    </nav>
    <div id="content">
        <super/>
    </div>
</body>
```
Note how `with-menu.html` is both fulfilling the *interface* (ie the extension points) of `base.html` and at the same time it's creating new ones for another *Super template* to fulfill in turn.

***`layouts/page.html`***
```html
<extend template="with-menu.html"/>

<head id="head"></head>

<title id="title" var="$page.title"></title>

<div id="content" var="$page.content"></div>
```

In this last example `page.html` is featuring Scripty, the scripting language used to refer to your content from templates.

## Logic
Template logic is based on two main variables: `$site` and `$page`, representing the global site configuration and the current page respectively.

Example markdown file:

***`content/foo/index.md`***
```md
---
{
  "title": "My Post",
  "date": "2020-07-06T00:00:00",
  "author": "Sample Author",
  "draft": false,
  "layout": "post.html",
  "tags": ["tag1", "tag2", "tag3"]
}  
--- 
The content
```


#### `var`
Prints the contents of a Scripty variable.

***`layouts/post.html`***
```html
<span var="$page.title"></span>  
```

***`output`***
```html
<span>My Blog post</span>
```


#### `if`
Toggles the body of an element based on the condition.


***`layouts/post.html`***
```html
<div if="$page.title.len().eq(1)" id="foo">
    <b>Won't be rendered</b>
</div>  
```

***`output`***
```html
<div id="foo"></div>
```

#### `loop`
Repeats the body of an element based on the condition. Inside an element with a `loop` attribute, `$loop` becomes available.


***`layouts/post.html`***
```html
<ul loop="$page.tags" id="tags">
   <li var="$loop.it"></li>
</ul>  
```

***`output`***
```html
<ul id="tags">
   <li>tag1</li>
   <li>tag2</li>
   <li>tag3</li>
</ul>  
```

#### `inline-loop`
Repeats the **entire** element based on the condition. Inside an element with a `loop` attribute, `$loop` becomes available.

***`layouts/post.html`***
```html
<div inline-loop="$page.tags" var="$loop.it"></div>
```

***`output`***
```html
<div>tag1</div>
<div>tag2</div>
<div>tag3</div>
```

## Scripty Reference
To learn about all the types present in Scripty and their builting operations, [see the full reference](scripty/).
