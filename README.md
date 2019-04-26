I tried to setup a development environment with docker for the following 2 reasons. First, docker images are portable, a development environment is not easy to set up and should be reusable. Second, using redsocks to redirect all traffic within docker can avoid many problems which affect the whole os.

I first tried to create a new bridge network for docker, which for some unfathomable reason did not work. Two containers still can not communicate with each other. Thus the shadowsocks instance is unusable in another container. I tried host mode network for docker. This did work. But iptables messed up with my network outside docker. In the spirit of making it work first, I tried to bundle shadowsocks into ubuntu 18.04. Here is my dockerfile.

```
FROM ubuntu:18.04

# Set the working directory to /app
WORKDIR /app
# Copy the current directory contents into the container at /app
ADD . /app

ENV PROXY_SERVER=localhost
ENV PROXY_PORT=1089

RUN apt-get update
RUN apt-get upgrade -qy
RUN apt-get install shadowsocks-libev git zsh iptables redsocks curl lynx neovim -qy
COPY redsocks.conf /etc/redsocks.conf

ENTRYPOINT /bin/bash /app/run.sh
```

where `/app/run.sh` setup the whole redsocks environment, `redsocks.conf` is a redsocks configuration file.

```
#!/bin/bash
set -xe
env
sed -i "s/vPROXY-SERVER/$PROXY_SERVER/g" /etc/redsocks.conf
sed -i "s/vPROXY-PORT/$LOCAL_PORT/g" /etc/redsocks.conf

nohup ss-local -s "$SERVER_ADDR" -p "$SERVER_PORT" -b "$LOCAL_ADDR" -l "$LOCAL_PORT" -m "$METHOD" -k "$PASSWORD" -u --no-delay --fast-open &

/etc/init.d/redsocks restart

iptables -t nat -A OUTPUT -p tcp -d "$SERVER_ADDR" --dport "$SERVER_PORT" -j ACCEPT
iptables -t nat -A OUTPUT -p tcp -d "$LOCAL_ADDR" --dport "$LOCAL_PORT" -j ACCEPT
iptables -t nat -A OUTPUT -p tcp -d "127.0.0.1" --dport "12345" -j ACCEPT
iptables -t nat -A OUTPUT -p tcp -d "$LOCAL_ADDR" --dport "12345" -j ACCEPT
iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-port 12345

# Run app
curl http://ipinfo.io/json

exec zsh "$@"
```

running `docker run -it --privileged --name=crb --env-file ~/docker.env crb` gives me a gfw-agnostic docker container.

Fortunate for me, this attempt also failed, as I can not run `lvmetad` in the docker container laterly anyway, and this is required for `cros_sdk` setup. Only then I found my root partition is not large enough. I will have to bind mount `/var/lib/docker` if I was to use docker to build chromium os, or maybe I may have to use docker volume. But things could become nasty.

I tried to copy all my files downloaded into the docker container with `docker cp`. It failed with not enough space. I destroyed all the docker container to make way for a new development environment outside docker.

I wrote a script `proxy.sh` to start and stop my transparent proxy environment.

```
#!/bin/bash
set -ex
if [[ $EUID -ne 0 ]]; then
    sudo "$0" "$@"
    exit "$?"
fi

do_restart() {
    do_stop
    do_start
}

do_start() {
    pkill ss-local || true
    nohup ss-local -s "$SERVER_ADDR" -p "$SERVER_PORT" -b "$LOCAL_ADDR" -l "$LOCAL_PORT" -m "$METHOD" -k "$PASSWORD" -u --no-delay --fast-open &
    sed -i "s/vPROXY-SERVER/$PROXY_SERVER/g" /etc/redsocks.conf
    sed -i "s/vPROXY-PORT/$LOCAL_PORT/g" /etc/redsocks.conf
    /etc/init.d/redsocks restart
    iptables -t nat -N REDSOCKS || iptables -t nat -F REDSOCKS
    iptables -t nat -A OUTPUT -p tcp -j REDSOCKS
    iptables -t nat -A REDSOCKS -p tcp -d "$SERVER_ADDR" --dport "$SERVER_PORT" -j RETURN
    iptables -t nat -A REDSOCKS -d 0.0.0.0/8 -j RETURN
    iptables -t nat -A REDSOCKS -d 10.0.0.0/8 -j RETURN
    iptables -t nat -A REDSOCKS -d 127.0.0.0/8 -j RETURN
    iptables -t nat -A REDSOCKS -d 169.254.0.0/16 -j RETURN
    iptables -t nat -A REDSOCKS -d 172.16.0.0/12 -j RETURN
    iptables -t nat -A REDSOCKS -d 192.168.0.0/16 -j RETURN
    iptables -t nat -A REDSOCKS -d 224.0.0.0/4 -j RETURN
    iptables -t nat -A REDSOCKS -d 240.0.0.0/4 -j RETURN
    iptables -t nat -A REDSOCKS -p tcp -j REDIRECT --to-port 12345
}

do_stop() {
    pkill ss-local
    /etc/init.d/redsocks stop
    iptables -t nat -F REDSOCKS
    iptables -t nat -X REDSOCKS
    iptables -t nat -D OUTPUT -p tcp -j REDSOCKS
    iptables -t nat -D REDSOCKS -p tcp -d "$SERVER_ADDR" --dport "$SERVER_PORT" -j RETURN
    iptables -t nat -D REDSOCKS -d 0.0.0.0/8 -j RETURN
    iptables -t nat -D REDSOCKS -d 10.0.0.0/8 -j RETURN
    iptables -t nat -D REDSOCKS -d 127.0.0.0/8 -j RETURN
    iptables -t nat -D REDSOCKS -d 169.254.0.0/16 -j RETURN
    iptables -t nat -D REDSOCKS -d 172.16.0.0/12 -j RETURN
    iptables -t nat -D REDSOCKS -d 192.168.0.0/16 -j RETURN
    iptables -t nat -D REDSOCKS -d 224.0.0.0/4 -j RETURN
    iptables -t nat -D REDSOCKS -d 240.0.0.0/4 -j RETURN
    iptables -t nat -D REDSOCKS -p tcp -j REDIRECT --to-port 12345
}

do_status() {
    pgrep -f -a ss-local || true
    /etc/init.d/redsocks status
    iptables -L -t nat --line-numbers
}

# Run app
# curl http://ipinfo.io/json
#
conf=""
if [[ "${#@}" -ne 2 ]]; then
    echo "Usage: $0 conf { start | stop | restart | status }"
    exit 1
else
    conf="$1"
fi

. "$conf"

env
shift
RETVAL=0
case "$1" in
start | stop | restart | status)
    do_$1
    ;;
*)
    echo "Usage: $0 { start | stop | restart | status }"
    RETVAL=1
    ;;
esac

exit "$RETVAL"
```

After running `proxy.sh proxy.conf start`, the `repo sync` is fast enough. But unfortunately, shortly my vps has been taken care of by gfw. I encountered connection reset error every a few dozens of seconds. This became intolerable for me, as `emerge` has no break point download. I setup a new vps in digital ocean, and installed shadowsocks. Now no more connection reset. But the packet loss rate was so high that tcp transmission speed almost never reached 10KB/s.

Bliss did not come even after I tried yet another vps. The speed of downloading packages from some domains like `commonstoreage.googleapi.com` was unbearable. I tried ping this domain from my vps and my localhost. Got different results, thought it could be DNS poisoning. I stopped `systemd-resolved` and setup dnscrypt-proxy on the VM, and configured it to use only tcp for any dns query, only in this way I don't have to worry about DNS server itself was blocked. TCP traffic was forwarded to my remote shadowsocks server. This time everything seemed to be working fine.

This is what I did in the first day. Yeah, not much, I knew. This is my night job mostly. Before I went to asleep, I tried to make the vm busy with some supposed-to-be idempotent operations.

```
for i in `seq 1 20`; do cros_sdk; cros_sdk -- ./build_packages --board=${BOARD}; cros_sdk -- ./build_image --board=${BOARD}; done
```

The thing is that `cros_sdk` is not pure in functional programming jargon. It depends on my current environment. When I wake up, the build did not finish the way I had expected. It stopped at chroot. So I still had to wait for the compilation of images patiently.

The long wait was over. And then I found I was building for a wrong target amd64-generic. I did set `export BOARD=x86-generic` in my `~/.bashrc`. It could be unset while `chroot`ing.

New problem emerged while `emerge`ing

```
!!! The ebuild selected to satisfy "chromeos-base/chromeos-chrome" for /build/x86-generic/ has unmet requirements.
- chromeos-base/chromeos-chrome-76.0.3775.0_rc-r1::chromiumos USE="accessibility autotest build_tests buildcheck cfi chrome_debug chrome_remoting cups debug_fission evdev_gestures fonts highdpi libcxx lld nacl opengles reorder_text_sections runhooks thinlto v4l2_codec vaapi xkbcommon -afdo_use -app_shell -asan (-authpolicy) -chrome_debug_tests -chrome_internal -chrome_media -clang
-clang_tidy -component_build -gold -goma -internal_gles_conform -jumbo -mojo (-neon) -new_tcmalloc -oobe_config -opengl -strict_toolchain_checks -v4lplugin -verbose -vtable_verify" OZONE_PLATFORM="default_gbm gbm -caca -cast -default_caca -default_cast -default_egltest -default_test -egltest -test" OZONE_PLATFORM_DEFAULT="gbm -caca -cast -egltest -test"

  The following REQUIRED_USE flag constraints are unsatisfied:
    libcxx? ( clang ) thinlto? ( clang )
```

In fairly plain English, it told me that I should add USE flag clang to `chromeos-base/chromeos-chrome-76.0.3775.0_rc-r1`. Did that, and found it still was not working. The USE flag for chromeos-chrome was still -clang. Finally, I found out I should just `sudo cp /etc/portage/package.use/chromeos-chrome /build/x86-generic/etc/portage/package.use`. Eureka.

The pain of another long ongoing wait was eased by `tail -f /build/x86-generic/tmp/portage/logs/*log`.

I started the build process at 10 am and it is still building at 11 pm. So, this is concludes the story.

