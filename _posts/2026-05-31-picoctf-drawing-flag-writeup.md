---

title: "picoCTF — drawing.flag.svg Writeup"
date: 2026-05-31 10:00:00 -0500
categories: [CTF, Forensics]
tags: [picoctf, forensics, svg, xml, grep]
description: "A short picoCTF forensics writeup about extracting hidden text from an SVG file using terminal tools."
pin: false
----------

> This writeup is based on a picoCTF challenge that is no longer part of an active competition. Try solving the challenge first before reading.
> {: .prompt-info }

## Challenge Overview

The challenge provided a single file:

```text
drawing.flag.svg
```

Running `file` showed that the file was an SVG image:

```bash
file drawing.flag.svg
```

Output:

```text
drawing.flag.svg: SVG Scalable Vector Graphics image
```

At first glance, this looks like a normal image file. However, SVG files are not regular binary images. They are XML documents, which means their contents can be inspected directly as text.

## Initial Analysis

Since SVG is XML-based, the first step was to search the file for suspicious strings related to the flag format:

```bash
grep -iE "pico|ctf|flag|\{|\}" drawing.flag.svg
```

This revealed text stored inside SVG `<tspan>` elements:

```xml
id="tspan3764">F { 3 n h 4 n </tspan><tspan
id="tspan3752">c 3 d _ a a b 7 2 9 d d }</tspan></text>
```

![File command output](/assets/img/posts/drawingflag/grepFlag.png)

The flag content was split across multiple text spans and included extra spaces.

## Extracting Text from the SVG

To extract text from all `<tspan>` elements, I used Perl with multiline matching:

```bash
perl -0777 -ne 'while (/<tspan\b[^>]*>(.*?)<\/tspan>/sg) { print "$1\n" }' drawing.flag.svg
```

Then I removed all whitespace and joined the extracted fragments:

```bash
perl -0777 -ne 'while (/<tspan\b[^>]*>(.*?)<\/tspan>/sg) { print "$1" }' drawing.flag.svg | tr -d '[:space:]'
```

The visible part of the flag was:

```text
F{3nh4nc3d_aab729dd}
```

Adding the standard picoCTF prefix gives the final flag:

```text
picoCTF{3nh4nc3d_aab729dd}
```

## Why This Works

SVG files are text-based XML documents. Unlike PNG or JPG files, their contents can include readable tags, metadata, comments, styles, and text objects.

In this challenge, the flag was not hidden in image pixels. It was stored as text inside the SVG structure, specifically inside `<tspan>` elements.

## Key Takeaways

* SVG files should be inspected as text.
* `file` helps identify the real file type.
* `grep` is useful for quickly finding flag-like strings.
* XML-based formats may contain hidden or split text.
* Not every forensics challenge requires image manipulation; sometimes reading the source is enough.
