# Setup: publishing this site to GitHub Pages

This is a Hugo site using the PaperMod theme. Follow these steps once.

## 0. Install Hugo locally (to preview before publishing)
- macOS:  `brew install hugo`
- Linux:  `sudo apt install hugo`  (or download the "extended" build from gohugo.io)
- Windows: `choco install hugo-extended`

Check it worked: `hugo version`

## 1. Make this a git repo and add the theme
From inside this folder:

    git init
    git branch -M main
    git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod

(The theme is added as a submodule — the deploy workflow pulls it automatically.)

## 2. Add your images
Put these three files into `static/images/` (names must match exactly):
- single_threaded_architecture.png   <- your first Excalidraw diagram (FIX the "Coonnections" typo first!)
- multithreaded_architecture.png     <- the shard diagram (already copied if it was in uploads)
- shard_scaling.png                  <- the chart (already copied if it was in uploads)

## 3. Preview locally
    hugo server

Open http://localhost:1313 — you'll see the home page, the Posts list (your sidebar of
clickable posts), and the post itself. Toggle your OS dark mode to confirm auto night mode works.

## 4. Create the GitHub repo
Create a repo named EXACTLY:  shashank2602.github.io
(A repo with this name publishes at https://shashank2602.github.io/ — that's the shareable base URL.)

Then push:

    git remote add origin https://github.com/shashank2602/shashank2602.github.io.git
    git add .
    git commit -m "initial site"
    git push -u origin main

## 5. Turn on Pages
On GitHub: repo -> Settings -> Pages -> Build and deployment -> Source: "GitHub Actions".
The included workflow (.github/workflows/deploy.yml) builds and deploys on every push.
Wait ~1 minute; your site is live at https://shashank2602.github.io/

## 6. Publishing future posts
Drop a new .md file in content/posts/ with front matter (copy the top of vortexkv.md),
then: git add . && git commit -m "new post" && git push
It's live in ~30 seconds. Each post gets its own shareable URL automatically:
https://shashank2602.github.io/posts/<filename>/

## Optional: custom domain later
If you buy shashankjoshi.dev, add a file named `static/CNAME` containing just `shashankjoshi.dev`,
set DNS at your registrar, and update baseURL in hugo.toml. Not required — the github.io URL works fine.
