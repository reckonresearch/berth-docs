# Brand assets for the docs header

Two files belong here and are not in version control for this repo, because
they are owned by the site repo. Copy them before building:

    SITE=~/Desktop/"Reckon Research"/zips/reckonresearch.github.io
    cp "$SITE/assets/reckon-mark-white.png" docs/assets/
    cp "$SITE/favicon.ico"                  docs/assets/

`mkdocs.yml` references them as `theme.logo` and `theme.favicon`. If they are
absent the build still succeeds and the header falls back to Material's default
book icon, so check the header after building rather than trusting the exit
code.
