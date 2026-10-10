https://raw.githubusercontent.com/AngelGonePro/file-network-share/refs/heads/main/network-share.zip

```
rm -rf ~/network-share && \
mkdir -p ~/network-share && \
wget -q -O /tmp/network-share.zip https://raw.githubusercontent.com/AngelGonePro/file-network-share/refs/heads/main/network-share.zip && \
unzip -q /tmp/network-share.zip -d ~ && \
rm /tmp/network-share.zip && \
cd ~/network-share && \
ls -la
```
