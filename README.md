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



bash << 'EOF'
say() {
  local n="$1" p found=0
  echo "-- $n"
  for p in \
    "/lib64/$n" \
    "/usr/lib64/$n" \
    "/opt/happ/lib/$n" \
    "/opt/happ/.smart-runtime/lib/x86_64-linux-gnu/$n" \
    "/opt/happ/.smart-runtime/usr/lib/x86_64-linux-gnu/$n"
  do
    if [[ -L "$p" && ! -e "$p" ]]; then
      echo "BROKEN $p -> $(readlink "$p")"
      found=1
    elif [[ -e "$p" ]]; then
      ls -l "$p"
      found=1
    fi
  done
  [[ $found == 1 ]] || echo "NO $n"
}
file() {
  echo "-- $1"
  if [[ -L "$1" && ! -e "$1" ]]; then echo "BROKEN $1 -> $(readlink "$1")"
  elif [[ -e "$1" ]]; then ls -l "$1"
  else echo "NO $1"; fi
}
echo "=== СИСТЕМНЫЕ ССЫЛКИ СКРИПТА ==="
file /lib/x86_64-linux-gnu
file /usr/lib/x86_64-linux-gnu
file /opt/happ/.ld64
echo "=== БИБЛИОТЕКИ РАНТАЙМА ==="
for n in \
  ld-linux-x86-64.so.2 libc.so.6 libm.so.6 libmvec.so.1 \
  libpthread.so.0 libdl.so.2 librt.so.1 libresolv.so.2 \
  libutil.so.1 libanl.so.1 libnsl.so.1 libBrokenLocale.so.1 \
  libthread_db.so.1 libc_malloc_debug.so.0 \
  libnss_files.so.2 libnss_dns.so.2 libnss_compat.so.2 libnss_hesiod.so.2 \
  libgcc_s.so.1 libstdc++.so.6 libssl.so.3 libcrypto.so.3 \
  libssl.so.1.1 libcrypto.so.1.1
do say "$n"; done
echo "=== GUI, ИХ СКРИПТ НЕ СТАВИТ ==="
for n in libGL.so.1 libEGL.so.1 libX11.so.6 libxcb.so.1 \
  libxkbcommon.so.0 libfontconfig.so.1 libfreetype.so.6 \
  libnss3.so libglib-2.0.so.0 libdbus-1.so.3
do
  echo "-- $n"
  ldconfig -p 2>/dev/null | grep "$n" || echo "NO $n"
done
echo "=== ФАЙЛЫ HAPP ==="
for f in \
  /opt/happ/bin/Happ /opt/happ/bin/Happ.orig \
  /opt/happ/bin/happd /opt/happ/bin/happd.orig \
  /opt/happ/bin/happ-diag /opt/happ/bin/happ-tcping \
  /opt/happ/bin/core/xray /opt/happ/bin/core/xray.orig \
  /opt/happ/bin/tun/sing-box /opt/happ/bin/tun/sing-box.orig \
  /opt/happ/bin/tun2/tun2proxy-bin /opt/happ/bin/tun2/tun2proxy-bin.orig \
  /opt/happ/bin/tun2/udpgw-server \
  /opt/happ/bin/core/geoip.dat /opt/happ/bin/core/geosite.dat \
  /dev/net/tun /usr/sbin/setcap /usr/bin/setcap
do file "$f"; done
echo "=== DONE ==="
EOF