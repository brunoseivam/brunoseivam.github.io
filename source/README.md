# Install

Look for latest version https://github.com/gohugoio/hugo/releases

    wget https://github.com/gohugoio/hugo/releases/download/v0.71.0/hugo_0.71.0_Linux-64bit.deb
    sudo dpkg -i hugo*.deb

# Creating a new site

    hugo new site source

# Add a theme

Browse themes on [themes.gohugo.io](themes.gohugo.io).
Install a theme:

    cd source
    git init
    git submodule add https://github.com/lxndrblz/anatole themes/anatole
    echo 'theme = "anatole" >> config.toml

# Add content

    hugo new posts/my-first-post.md

Then edit the file: `content/posts/my-first-post.md`

# Start the local hugo server

    hugo server -D

Go to http://localhost:1313 to see the website.

# Build static pages

    hugo -D --destination ../




