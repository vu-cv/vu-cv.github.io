# vu138.github.io
Author: Chu Van Vu
email: chuvanvu1324@gmail.com

## Step 1: Install ruby
```
cd $HOME
sudo apt-get update 
sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev

git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec $SHELL

git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
exec $SHELL

rbenv install 2.5.0
rbenv global 2.5.0
ruby -v
```

## Step 2: Clone repo
git clone https://github.com/vu138/vu138.github.io.git

## Step 3: Run jekyll
```
cd folder_blog
gem install bundler
bundle install
bundle exec jekyll serve
```

Now browse to http://localhost:4000