---
title: Using Pandoc Markdown with Hugo on Vercel
---

In a not-for-monetary-profit project of mine, I use [Pandoc](https://pandoc.org) for converting Google Docs documents to markdown. I then publish these on a website using [Hugo](https://gohugo.io) and [Vercel](https://vercel.com). 

Pandoc and Hugo both support several markdown flavours, but they only have one in common, you guessed it: Pandoc's. That is all good, we just tell Hugo to use the Pandoc markdown engine, right? However, Hugo doesn't have a Pandoc markdown parser of its own, it relies on that the Pandoc binaries are installed. Which they aren't on Vercel. (They are if you use [Netlify](https://www.netlify.com), so if you do you can skip reading now. 😀)

It took me a little experimentation to get it to work so writing this down her for anyone else to save some times. I started with contacting Vercel's wonderful support. Despite me being on the free tier they give me of their time, in a very timely manner. They told me that I can use anything for building my site and adviced me to configure the build command (for which I had left the default `npm run build`) to `sh build.sh` or something and then install dependencies before running `npm run build`. They also told me that the build servers are using Centos.

From what I could tell there is no reasonably updated Pandoc package for Centos. The [Pandoc install instructions](https://pandoc.org/installing.html#linux) say to download the tar ball on the distros they don't update themselves. The tarball has `amd64` in the file name, and I wondered if that would work on Vercel. It did.

Then it was a question about how to get pandoc on the path. I tried to symlink it to `/usr/bin`, which seemed to be allowed, but it didn't work. I then tried with appending the local pandoc bin directory to $PATH, which worked.

So, this is my `vercel-build.sh` now, with a bit of the diagnostics output left in:

```sh
echo Rohan responds. Gondor shall not fall.
echo ===
echo installing pandoc
TGZ=pandoc-2.10.1-linux-amd64.tar.gz
PDV=pandoc-2.10.1
curl -sOL "https://github.com/jgm/pandoc/releases/download/2.10.1/$TGZ"
tar xvzf "$TGZ"
export PATH="$PATH:$PDV/bin"
pandoc -v
echo ===
echo running npm run build
npm run build
```

To instruct Hugo to use this binary I have this in my site `config.toml`:

```toml
[markup]
defaultMarkdownHandler = "pandoc"
[markup.pandoc.renderer]
unsafe = true
```

(I dare using unsafe html settings, because I control all text being published on the site, but thanks for your concerns! 😍)

There is one more litte bit of advice I might give you. From what I understand there is no way to instruct Vercel to use different build commands for different branch deploys. This makes it a bit scary to change it, while experimenting with deploying build scripts and see if they work on a remote machine out there. What I did was to first change the build command in Vercel to `sh vercel-build.sh`. Then I deployed a `vercel-build.sh` containing only:

```sh
npm run build
```

And deployed that on `master`. Then I could experiment on a development branch, and merge back once things where working. (Which took me a while, as I've already said, but that hopefully now will take you less time. ♥️)

