## Personal Blog

This repository contains the source for my personal blog.

### Development

This blog is built with `jekyll`.

**Install Ruby**

From the `rbenv` documentation (with modifications for `zsh`):

```bash
sudo apt-get update 
sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libxml2-dev libxslt1-dev libcurl4-openssl-dev libffi-dev

git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(rbenv init -)"' >> ~/.zshrc
exec $SHELL

git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.zshrc
exec $SHELL

rbenv install 3.1.2
rbenv global 3.1.2
ruby -v
```

**Check Prerequisites**

Ensure that we now have all prerequisites installed:

```bash
# Ruby
ruby -v
# RubyGems
gem -v
# GCC and Make
gcc -v
g++ -v
make -v
```

**Run Jekyll Locally**

From the [Jekyll documentation](https://jekyllrb.com/docs/):

```bash
# Install jekyll and bundler gems
gem install jekyll bundler
# Ensure rbenv can see new gems
rbenv rehash

# Start the development server
bundle exec jekyll serve
```

I ran into some issues with this step initially because I had not previously run `jekyll new ...`. I fixed this issue by running these commands in a separate directory and copying the `Gemfile` and `Gemfile.lock` that were generated.

### Acknowledgements

Original layout / boilerplate for this site cloned from the [jekyll-now](https://github.com/barryclark/jekyll-now) project.