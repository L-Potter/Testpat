apt-get install gnome-keyring
apt-get install libsecret-1-0
apt-get install libsecret-1-dev
ps aux | grep gnome
apt install libsecret-tools
secret-tool --version
secret-tool store --label="TestKey" service test username testuser
# Init DEfault keyring password for gui encoding
secret-tool lookup service test username testuser
sudo make --directory=/usr/share/doc/git/contrib/credential/libsecret/
git config --global credential.helper /usr/share/doc/git/contrib/credential/libsecret/git-credential-libsecret
git config --global user.name "danny"
git config --global user.email "danny@..."

git add
git commit -m 
git push origin main
#gui
sudo apt install seahorse
secret-tool search server github.com
