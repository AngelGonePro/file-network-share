```
rm -rf ~/cloud-frps && \
mkdir -p ~/cloud-frps && \
wget -q -O /tmp/cloud-frps.zip https://raw.githubusercontent.com/AngelGonePro/frp-docker/refs/heads/main/cloud-frps.zip && \
unzip -q /tmp/cloud-frps.zip -d ~ && \
rm /tmp/cloud-frps.zip && \
cd ~/cloud-frps && \
ls -la
```
