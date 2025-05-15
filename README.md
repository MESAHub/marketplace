# MESA Marketplace

The MESA Marketplace is a collection of
educational resources, summer school links, inlists, and add-ons for the
[Modules in Stellar Astrophysics (MESA)](https://mesastar.org) code.

The MESA Marketplace was originally hosted on Frank Timmes' website https://cococubed.com
and was migrated to GitHub Pages in 2025.

We now have a
[MESA Zenodo Community](https://zenodo.org/communities/mesa/records?q=&l=list&p=1&s=10)
as the prefered way of sharing MESA inlists and projects.
Check it out! The inlists and add-ons shared shared on the Marketplace are generally from before 2022.
But this website still serves as a useful collection of
pre-Zenodo and pre-GitHub migration information, and has many useful links to
educational videos, blog posts, and MESA summer schools.

## Building the website

To build the website locally, make sure you have installed `hugo` (e.g. `brew install hugo`),
and then do:

```console
hugo server
```

and open `localhost:1313` in your browser.

This site will automatically deploy to GitHub Pages
with each update, through a GitHub action.

## Adding new content

To add new content, just do, e.g.

```console
hugo new content content/guides/my-first-guide.md
```

to create a new Markdown page. Then edit the file with your content. Make sure to change `draft = false` in the file's header for it to get published.
