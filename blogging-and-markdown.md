# Blogger and Markdown

I have been writing on [Blogger](https://www.blogger.com), on and off, since 2007. In all that time, I've always wanted to find a way to write my blog posts more easily without having to resort to HTML.

I am very happy to say that technology has finally caught up with my wish and my last few posts have been written using a combination of:

- [Markdown](https://daringfireball.net/projects/markdown/)
- [Visual Studio Code](https://code.visualstudio.com/), with the following plugins:
    - [Auto-Open Markdown Preview](https://marketplace.visualstudio.com/items?itemName=hnw.vscode-auto-open-markdown-preview)
    - [markdownlint](https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint)
    - [vscode-pandoc](https://marketplace.visualstudio.com/items?itemName=DougFinke.vscode-pandoc)
- [Pandoc](http://pandoc.org/)

Before I go any further, I would like to extend many, many thanks to the authors of all this amazing software and to Dave Johnson for his blog post, [Build an Amazing Markdown Editor Using Visual Studio Code and Pandoc](http://thisdavej.com/build-an-amazing-markdown-editor-using-visual-studio-code-and-pandoc/), which helped me put it all together.

At a basic level, I followed Dave's steps pretty closely, so I won't repeat those here. Instead, I will focus on what I had to do to get this setup working.

## Software installation

All software installation went very smoothly on my Windows machine, with no real problems. This post was made with the following versions of software:

- Visual Studio Code: 1.9.1
    - Auto-Open Markdown Preview: 0.0.3
    - markdownlint: 0.6.2
    - vscode-pandoc: 0.0.8
- Pandoc: 1.19.2.1

## Blogger line break quirks

Blogger made a weird ([though apparently well documented](http://webapps.stackexchange.com/a/8783)) decision to use `<br>` tags instead of `<p>` tags for line breaks in their WYSIWYG editor. Not only do they use only `<br>` tags, but the WYSIWYG editor will actively, and incorrectly, convert `<p>` tags to `<br>`. This is a **huge** pain until you realize what Blogger is doing. Luckily, there is a workaround.

- Once Pandoc converts the Markdown into HTML, paste it into the editor's HTML mode.
- **Do not go to the editor's Compose mode after pasting your HTML.**
- If you want to see what your page will look like, use the Preview button.

When you stay out of Compose mode, all your `<p>` tags will be nicely preserved.

## Pandoc's HTML vs. Blogger's expectations

Blogger expects you to provide the `<body>` portion of an HTML file for each blog post. But Pandoc produces full HTML files, including a header section. Again, there is a workaround.

When Pandoc creates an HTML 5 file, there are certain lines which it includes in the file that can be safely removed and, upon removal, will result in Blogger-compatible HTML.

In the segment below, the first set of ellipses represent your CSS for syntax highlighting (more on that below) and the second set of ellipses represent your actual blog content.

You want to **remove** all the lines that are shown in the code snippet below.

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="generator" content="pandoc">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes">
  <title></title>
... <!-- Keep - CSS -->
</head>
<body>
... <!-- Keep - blog content -->
</body>
</html>
```

## Syntax highlighting

Blogger does not come with any syntax highlighting function / mode. However, Pandoc provides a number of [syntax highlighting styles](http://pandoc.org/MANUAL.html#syntax-highlighting). You can use these styles in Blogger to make your code easier to read.

First of all, please note that when Pandoc converts Markdown to HTML, it normally tries to produce multiple files - CSS and HTML in this instance.

You can force Pandoc to inline all the CSS code using the option `-s`, which will produce a standalone HTML file. You can then sanitize the HTML file per the above instructions.

## Configuring markdownlint

The markdownlint plugin is extremely helpful in writing good Markdown files, especially for someone new to the format (like me). However, I chose to disable certain options (slightly different from what Dave Johnson did).

MD026 warns you when you have trailing punctuation in headers. For a few posts, my headers are questions and need the question mark at the end. I found the warning annoying and so turned it off.

MD007 warns you when sub-lists are indented more than 2 spaces. For Pandoc Markdown, I have to indent by 4 spaces, so I disabled this warning.

The contents of my `.markdownlint.json` file are below.

```javascript
{
    "default": true,
    "MD026": false,
    "MD007": false,
    "no-hard-tabs": false,
    "line-length": false
}
```

## Configuring vscode-pandoc

The vscode-pandoc plugin is the tool that connects Visual Studio Code, Pandoc, and your default browser. The plugin comes with 3 options - one each for the 3 output formats. I only changed the HTML-related option, since I am not interested in producing PDF or Docx files at this time.

My configuration for vscode-pandoc is as follows:

```javascript
{
    ... // other user settings

    "pandoc.htmlOptString": "-s -f markdown -t html5 --highlight-style=pygments"
}
```

What do these options do?

-s
  ~ Produces a standalone file (combined CSS and HTML).

-f markdown
  ~ Sets the input type to Pandoc Markdown.
  ~ You can change this to original Markdown, GitHub-flavored Markdown, etc.
  ~ This "definition list" I am currently typing is courtesy of Pandoc Markdown.

-t html5
  ~ Produces a HTML 5 file.

--highlight-style=pygments
  ~ Sets the code highlight style to be similar to [Pygments](http://pygments.org/).
  ~ I recommend looking at the [various options you have available (#18)](http://pandoc.org/demos.html) and picking the style that suits your needs best.
  ~ Pygments is actually the default style, though I did test out all available options. I left this option in here in case Pandoc decides to change the default style in the future.

## Unfulfilled items from my wish list

The one thing that I don't get with this setup is a sort of Intellisense / type inference / type annotation functionality that I've seen on some other blogs. This is an extremely cool feature that I can see being very helpful (almost akin to a more literate style of programming).

Specifically, there are some blogs out there which show you type information for F#, like function signatures, when you mouse-over the code in the post. I do not know at this time what plug-ins those writers are using to achieve this functionality. However, I will leave that investigation for another day.

## Conclusion

I hope this blog post is helpful to people who are interested in using Markdown for their documentation needs. So far, I have found the experience of writing blog posts extremely enjoyable thanks to this setup and can see myself using it for the foreseeable future.
