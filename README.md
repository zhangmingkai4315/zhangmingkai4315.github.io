## Mikezhang.CC

个人静态网站

linux上部署可以参考：
```

gem install jekyll bundler

echo '# Install Ruby Gems to ~/gems' >> ~/.zshrc
echo 'export GEM_HOME=$HOME/gems' >> ~/.zshrc
echo 'export PATH=$HOME/gems/bin:$PATH' >> ~/.zshrc
source ~/.zshrc

sudo apt-get install zlib1g-dev
bundle install
make build # or make server
```
