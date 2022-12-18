# pleonex's website

Personal portfolio and blog.

Created using [Hugo](https://gohugo.io/) and
[Toha](https://github.com/hugo-toha/toha) template.

## Building

1. Install Hugo. Check the [workflow](.github/workflows/hugo.yml) for the
   supported version.
2. Build:
   - Run `hugo` to build. Output would be in the `public/` directory.
   - Run `hugo server --watch` to test locally with hot-reload support.

## Deploying

GitHub Actions provides a workflow to deploy _Hugo_ based websites to GitHub
pages. A custom [DNS entry](./CNAME) redirects to the GitHub Pages site.
