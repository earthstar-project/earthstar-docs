# Website

This website is built using [Docusaurus 2](https://v2.docusaurus.io/), a static website generator.

## To add new pages

1. Add markdown files in the `docs` folder.
2. Update the `sidebars.js` file to include your new file.

## Formatting your markdown

Don't use h1 (`#`), start at h2 (`##`).

Your markdown file should start with a "front matter" which sets the page title in the sidebar and on the page itself.  The markdown filename is used as the URL for your page so avoid special characters.

Front matter example:
```
---
title: How To Do Things
---

## Part 1

Blah blah blah...
```

## Installation

```console
yarn install
```

## Local Development

```console
yarn start
```

This command starts a local development server and open up a browser window. Most changes are reflected live without having to restart the server.

## Build

```console
yarn build
```

This command generates static content into the `build` directory and can be served using any static contents hosting service.

## Deployment

```console
GIT_USER=<Your GitHub username> USE_SSH=true yarn deploy
```

If you are using GitHub pages for hosting, this command is a convenient way to build the website and push to the `gh-pages` branch.
