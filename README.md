bash << 'EOF'
set -e
echo "===== ДО ====="
ls -ld /lib64/libc.so.6 /usr/lib64/libc.so.6 || true
ls -ld /lib/x86_64-linux-gnu /usr/lib/x86_64-linux-gnu 2>/dev/null || true
rpm -q glibc
rpm -q happ || true

if [ ! -e /lib64/libc.so.6 ]; then
  echo "СТОП: нет /lib64/libc.so.6 — это не этот откат. Нужна переустановка МОС/live."
  exit 1
fi

echo "===== 1) Ссылки установщика ====="
for p in /lib/x86_64-linux-gnu /usr/lib/x86_64-linux-gnu; do
  if [ -L "$p" ]; then
    echo "СНОШУ ССЫЛКУ $p -> $(readlink "$p")"
    sudo rm -f "$p"
  else
    echo "не трогаю $p"
  fi
done

echo "===== 2) Кэш линкера ====="
sudo ldconfig

echo "===== 3) Happ ====="
killall Happ Happ.orig happd 2>/dev/null || true
sudo systemctl stop happd.service 2>/dev/null || true
sudo rpm -e --noscripts happ 2>/dev/null || true
sudo dnf -y remove happ 2>/dev/null || true
sudo rm -rf /opt/happ
sudo ldconfig

echo "===== 4) tun / сеть (хвосты Happ) ====="
sudo chmod 660 /dev/net/tun 2>/dev/null || true
sudo chown root:root /dev/net/tun 2>/dev/null || true
for i in $(ip -o link show | awk -F': ' '{print $2}' | grep -E '^(tun|happ|xray)'); do
  echo "снимаю интерфейс $i"
  sudo ip link delete "$i" 2>/dev/null || true
done
sudo systemctl restart NetworkManager 2>/dev/null || true

echo "===== 5) Домашние хвосты ====="
rm -rf "$HOME/happ-rosa" "$HOME/happ.sh"
rm -rf "$HOME/.config/happ" "$HOME/.config/Happ" "$HOME/.local/share/happ" "$HOME/.local/share/Happ"

echo "===== ПОСЛЕ ====="
ls -ld /lib64/libc.so.6
ls -ld /lib/x86_64-linux-gnu /usr/lib/x86_64-linux-gnu 2>/dev/null || echo "debian-путей нет (так и надо на РОСА)"
echo "ping:"
ping -c 1 -W 3 8.8.8.8 || ping -c 1 -W 3 1.1.1.1 || echo "сеть ещё мёртвая"
echo "ГОТОВО. Перелогиньтесь или перезагрузитесь."
EOF



sudo ldconfig
killall Happ Happ.orig happd
sudo systemctl stop happd.service
sudo rpm -e --noscripts happ
sudo dnf -y remove happ
sudo rm -rf /opt/happ
sudo ldconfig
sudo chmod 660 /dev/net/tun
sudo chown root:root /dev/net/tun
rm -rf "$HOME/happ-rosa" "$HOME/happ.sh"
rm -rf "$HOME/.config/happ" "$HOME/.config/Happ"
rm -rf "$HOME/.local/share/happ" "$HOME/.local/share/Happ"


ls /lib64/libc.so.6
dnf --version
ping -c 2 8.8.8.8
rpm -q happ


mkdir -p "$HOME/Downloads/happ-mos12"
cd "$HOME/Downloads/happ-mos12"
curl -L --retry 5 -o smart-dnf-install https://raw.githubusercontent.com/andert133/mos12-happ-install/main/smart-dnf-install
curl -L --retry 5 -o Happ.linux.x64.rpm https://github.com/Happ-proxy/happ-desktop/releases/latest/download/Happ.linux.x64.rpm
chmod +x smart-dnf-install
sudo ./smart-dnf-install --auto Happ.linux.x64.rpm



sudo modprobe tun
sudo mkdir -p /dev/net
sudo test -e /dev/net/tun || sudo mknod /dev/net/tun c 10 200
sudo chmod 666 /dev/net/tun
sudo setcap cap_net_admin,cap_net_raw,cap_net_bind_service+eip /opt/happ/bin/Happ
sudo setcap cap_net_admin,cap_net_raw,cap_net_bind_service+eip /opt/happ/bin/Happ.orig
sudo setcap cap_net_admin,cap_net_raw,cap_net_bind_service+eip /opt/happ/bin/core/xray
sudo setcap cap_net_admin,cap_net_raw,cap_net_bind_service+eip /opt/happ/bin/tun/sing-box
sudo setcap cap_net_admin,cap_net_raw,cap_net_bind_service+eip /opt/happ/bin/tun2/tun2proxy-bin


sudo dnf -y install libcap
sudo setcap cap_net_admin,cap_net_raw,cap_net_bind_service+eip /opt/happ/bin/Happ
sudo setcap cap_net_admin,cap_net_raw,cap_net_bind_service+eip /opt/happ/bin/tun2/tun2proxy-bin
sudo setcap cap_net_admin,cap_net_raw,cap_net_bind_service+eip /opt/happ/bin/happd

sudo dnf -y install /usr/sbin/setcap


killall Happ Happ.orig
sudo dnf -y install /usr/sbin/setcap
sudo find /opt/happ -type f \( -name '*.orig' -o -name 'xray' -o -name 'sing-box' \) -exec setcap cap_net_admin,cap_net_raw,cap_net_bind_service+eip {} \; -print
