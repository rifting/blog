---
author: rifting
pubDatetime: 2024-09-17T19:04:36Z
modDatetime: 2024-09-17T19:04:36Z
title: Breaking Google's parental control suite, part 1
slug: family-link-1 
featured: true
draft: false
tags:
  - docs
description:
  How a group of friends and I exploited Google's parental control suite.
---

## Notice
This post is heavily inspired by CoolElectronics/Velzie's [Breaking CrOS](https://blog.coolelectronics.me/breaking-cros-1/) blog posts. I heavily recommend you go check them out if you haven't already.

## Some Context
Family Link is Google's parental control service. It allows parents to see the location of their children and teens, monitor app usage, and restrict when they can use certain apps, among many other things. What's notable about Family Link is that they have something called a "parent access code". The parent access code is a 6-digit TOTP code refreshed every hour that a parent can use to change settings on their child device if it's not connected to the internet.

Browsing the internet about Family Link (as a dumb child like me does), I came across a post on reddit suggesting the possibility of the TOTP code potentially being able to be reverse engineered. The idea defintely intrugied me, and I attempted to get in contact with some people in the post comments as well as a good friend of mine, [spencerpogo](https://github.com/spencerpogo).

## Motives
I decided to work on this project simply because I am a victim to Family Link myself, and found it annoying. I haven't ever actually had to use the TOTP codes, but the app does get on my nerves when it reports everything I do to my parents and enforces the bootloader to be locked with __NO WAY TO UNLOCK IT, EVEN WITH PARENTAL PERMISSION!!__

## Initial Brainstorming

As soon as I was in contact with everyone we decided to search for ways to find the TOTP code algorithim.
There were 2 main ideas we came up with
 - Reverse Engineering the Family Link website
 - Reverse Engineering the Family Link app

## The Website
Taking a look at the website for Family Link, I assume it won't be too hard. However, looking at the JavaScript that was compiled from dart, I realize just how unintentionally obfuscated this JavaScript truly is.

I looked for any calls to `Date.now();`, knowing the TOTP algorithim would be using a timestamp *somewhere*. Eventually I found a function, but it had contacted many other functions which contacted other functions.

I'm sure I could've reverse engineered the algorithim given enough time, but at this point I decided it was a better idea to reverse engineer the app instead. 

## The App
The Family Link app was made with the same codebase as the website; however the flutter was compiled to Java and not JavaScript.
We used [Jadx](https://github.com/skylot/jadx) to look into the family link codebase and the decompiled code was much more understandable than the insanely obfuscated dart-javascript monstrosity the website used.

However, everyone who I had gathered for the team, including me, had trouble making use of the decompiled app code too. It looked like we were at a dead end.

## Ash, the ChromeOS Window Manager/System UI comes to the rescue
That's right. Looking up anything related to the parent access code, ironically, on Google, yieleded some unit tests for Family Link.
The unit tests were useless on the own but they gave me the idea to look through Chromium codesearch, a website to search through the code of Chromium/ChromiumOS, for anything related to the parent access code.

To my suprise, under the ChromeOS WM folder, there lied C++ files for generating and validating the parent access codes.
I notified everyone on the development team and [spencerpogo](https://github.com/spencerpogo) made a rust-based POC for generating the parent access codes, essentially translated from the ash C++ files. The issue now though, was that we didn't know where the secret for the TOTP actually was. We knew it had to be sent to the parent somewhere, but it looked like we would have revist the website and/or app to find the shared secret.

## Finding the secret
We went back to the website and inspected the requests it would make, assuming the parent access code seed/secret would be there somewhere.
We found a long base32 string that looked like it was a TOTP code. We plugged it in to the code generator software and...

Nope. Didn't match the parent code.

It was demotivating, but we kept looking through the requests until we found a base64 string. By some miracle, we plugged it into the software and it worked!

 However, this would require the supervised child or teen to have physical access to their parents computer, which is a little privacy-invasive in my opinion. We decided to inform the entire team of the exploit anyway, and [r58playz](https://github.com/r58playz) made a bookmarklet to intercept the XHR and grab the shared secret.

 We informed a member of the team how to perform the exploit and he understood. He used the bookmarklet and got the seed.

Normally this would be nothing out of the oridinary, but then he told us he was on his supervised account.

That's right. For some godforsaken reason, some Google employee didn't see any problems in sending the shared secret to supervised users whenever they vist the flutter webapp. Even worse, supervised children and teens are signed into Chrome on their phones by default, allowing them to grab the shared secret and generate parent access codes in less than a minute.

Here is the list of frontmatter property for each post.

| Property           | Description                                                                                 | Remark                                        |
| ------------------ | ------------------------------------------------------------------------------------------- | --------------------------------------------- |
| **_title_**        | Title of the post. (h1)                                                                     | required<sup>\*</sup>                         |
| **_description_**  | Description of the post. Used in post excerpt and site description of the post.             | required<sup>\*</sup>                         |
| **_pubDatetime_**  | Published datetime in ISO 8601 format.                                                      | required<sup>\*</sup>                         |
| **_modDatetime_**  | Modified datetime in ISO 8601 format. (only add this property when a blog post is modified) | optional                                      |
| **_author_**       | Author of the post.                                                                         | default = SITE.author                         |
| **_slug_**         | Slug for the post. This field is optional but cannot be an empty string. (slug: ""❌)       | default = slugified file name                 |
| **_featured_**     | Whether or not display this post in featured section of home page                           | default = false                               |
| **_draft_**        | Mark this post 'unpublished'.                                                               | default = false                               |
| **_tags_**         | Related keywords for this post. Written in array yaml format.                               | default = others                              |
| **_ogImage_**      | OG image of the post. Useful for social media sharing and SEO.                              | default = SITE.ogImage or generated OG image  |
| **_canonicalURL_** | Canonical URL (absolute), in case the article already exists on other source.               | default = `Astro.site` + `Astro.url.pathname` |

> Tip! You can get ISO 8601 datetime by running `new Date().toISOString()` in the console. Make sure you remove quotes though.

Only `title`, `description` and `pubDatetime` fields in frontmatter must be specified.

Title and description (excerpt) are important for search engine optimization (SEO) and thus AstroPaper encourages to include these in blog posts.

`slug` is the unique identifier of the url. Thus, `slug` must be unique and different from other posts. The whitespace of `slug` should to be separated with `-` or `_` but `-` is recommended. Slug is automatically generated using the blog post file name. However, you can define your `slug` as a frontmatter in your blog post.

For example, if the blog file name is `adding-new-post.md` and you don't specify the slug in your frontmatter, Astro will automatically create a slug for the blog post using the file name. Thus, the slug will be `adding-new-post`. But if you specify the `slug` in the frontmatter, this will override the default slug. You can read more about this in [Astro Docs](https://docs.astro.build/en/guides/content-collections/#defining-custom-slugs).

If you omit `tags` in a blog post (in other words, if no tag is specified), the default tag `others` will be used as a tag for that post. You can set the default tag in the `/src/content/config.ts` file.

```ts
// src/content/config.ts
export const blogSchema = z.object({
  // ---
  draft: z.boolean().optional(),
  tags: z.array(z.string()).default(["others"]), // replace "others" with whatever you want
  // ---
});
```

### Sample Frontmatter

Here is the sample frontmatter for a post.

```yaml
# src/content/blog/sample-post.md
---
title: The title of the post
author: your name
pubDatetime: 2022-09-21T05:17:19Z
slug: the-title-of-the-post
featured: true
draft: false
tags:
  - some
  - example
  - tags
ogImage: ""
description: This is the example description of the example post.
canonicalURL: https://example.org/my-article-was-already-posted-here
---
```

## Adding table of contents

By default, a post (article) does not include any table of contents (toc). To include toc, you have to specify it in a specific way.

Write `Table of contents` in h2 format (## in markdown) and place it where you want it to be appeared on the post.

For instance, if you want to place your table of contents just under the intro paragraph (like I usually do), you can do that in the following way.

```md
---
# some frontmatter
---

Here are some recommendations, tips & ticks for creating new posts in AstroPaper blog theme.

## Table of contents

<!-- the rest of the post -->
```

## Headings

There's one thing to note about headings. The AstroPaper blog posts use title (title in the frontmatter) as the main heading of the post. Therefore, the rest of the heading in the post should be using h2 \~ h6.

This rule is not mandatory, but highly recommended for visual, accessibility and SEO purposes.

## Storing Images for Blog Content

Here are two methods for storing images and displaying them inside a markdown file.

> Note! If it's a requirement to style optimized images in markdown you should [use MDX](https://docs.astro.build/en/guides/images/#images-in-mdx-files).

### Inside `src/assets/` directory (recommended)

You can store images inside `src/assets/` directory. These images will be automatically optimized by Astro through [Image Service API](https://docs.astro.build/en/reference/image-service-reference/).

You can use relative path or alias path (`@assets/`) to serve these images.

Example: Suppose you want to display `example.jpg` whose path is `/src/assets/images/example.jpg`.

```md
![something](@assets/images/example.jpg)

<!-- OR -->

![something](../../assets/images/example.jpg)

<!-- Using img tag or Image component won't work ❌ -->
<img src="@assets/images/example.jpg" alt="something">
<!-- ^^ This is wrong -->
```

> Technically, you can store images inside any directory under `src`. In here, `src/assets` is just a recommendation.

### Inside `public` directory

You can store images inside the `public` directory. Keep in mind that images stored in the `public` directory remain untouched by Astro, meaning they will be unoptimized and you need to handle image optimization by yourself.

For these images, you should use an absolute path; and these images can be displayed using [markdown annotation](https://www.markdownguide.org/basic-syntax/#images-1) or [HTML img tag](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img).

Example: Assume `example.jpg` is located at `/public/assets/images/example.jpg`.

```md
![something](/assets/images/example.jpg)

<!-- OR -->

<img src="/assets/images/example.jpg" alt="something">
```

## Bonus

### Image compression

When you put images in the blog post (especially for images under `public` directory), it is recommended that the image is compressed. This will affect the overall performance of the website.

My recommendation for image compression sites.

- [TinyPng](https://tinypng.com/)
- [TinyJPG](https://tinyjpg.com/)

### OG Image

The default OG image will be placed if a post does not specify the OG image. Though not required, OG image related to the post should be specify in the frontmatter. The recommended size for OG image is **_1200 X 640_** px.

> Since AstroPaper v1.4.0, OG images will be generated automatically if not specified. Check out [the announcement](https://astro-paper.pages.dev/posts/dynamic-og-image-generation-in-astropaper-blog-posts/).
