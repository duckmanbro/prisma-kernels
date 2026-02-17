sudo mount -t tmpfs -o uid=0,gid=0,mode=0755 cgroup /sys/fs/cgroup

(after mounting) ...

sudo lxc-start -n ubuntu -F -l DEBUG -o lxc_debug.log
sudo lxc-start -n arch -F -l DEBUG

# DOCKER

sudo dockerd --iptables=false (daemon start)

# LXC 

path "$PREFIX/var/lib/lxc/ubuntu/config"

# Required Configuration
lxc.net.0.type = none
lxc.hook.version = 1
lxc.cgroup.devices.allow = a
lxc.mount.auto = cgroup:mixed sys:mixed proc:mixed

# A container that is doing nothing uses as little as some few MB of RAM.
# But when you run huge, really huge memory intensive programs or compilations, it will obviously use more RAM.
# Very, very intensive program/task == too much RAM == the LMK will free up the RAM by killing the containers.
# We don't want that.
# So we set the maximum RAM that the container is allowed to use.
# It will never go beyond this limit, so we have no more worries.
# Here, 2G = 2GB limit (can use M for MB, etc)
lxc.cgroup.memory.limit_in_bytes = 2G

# LXC does not set a default password for us, so we have to set it ourselves.
# We usually need to chroot into the container and manually set the password.
# It's boring to do this for every new container, so we will automate it.
# This one-time hook will set a temporary password called 'password' for the 'root' user and the default user (eg:- 'ubuntu'). 
# This is useful for newbies and you can change it later from inside the container.
# It'll run ONLY ONCE at the very first run of the container, so it won't interfere if the password is changed by the user later on.
# Temporary password for 'root' is 'password' (no quotes).
# Remember to change your password later using command 'passwd'
lxc.hook.pre-start = bash -c "echo 'Set Temporary Password'; LD_PRELOAD= chroot '${LXC_ROOTFS_PATH}' usr/bin/bash -c \"/usr/bin/echo password | /usr/bin/sed 's/.*/\0\n\0/' | /usr/bin/passwd root; /usr/bin/echo password | /usr/bin/sed 's/.*/\0\n\0/' | /usr/bin/passwd ubuntu\"; sed -i -E \"s/(.*echo 'Set Temporary Password'.*)/# \1/g\" '${LXC_CONFIG_FILE}'; true;"

# Brings Termux colors to the containers' console
lxc.environment = TERM="xterm-256color"

# This will do a bunch of important things -
# 1) Mount the required cgroups
# 2) Sets correct DNS resolver to fix connectivity
# 3) Makes non-funtional udevadm always return true, or else some packages and snaps gives errors when trying to install
# 4) Sets temporary suid for the rootfs using bind mounts, otherwise normal users inside the container won't be able to use sudo commands
lxc.hook.pre-start = bash -c "if ! mountpoint -q /sys/fs/cgroup &>/dev/null; then mkdir -p /sys/fs/cgroup; mount -t tmpfs -o rw,nosuid,nodev,noexec,relatime cgroup_root /sys/fs/cgroup; fi; for cg in blkio cpu cpuacct cpuset devices freezer memory pids; do if ! mountpoint -q /sys/fs/cgroup/\${cg} &>/dev/null; then mkdir -p /sys/fs/cgroup/\${cg}; mount -t cgroup -o rw,nosuid,nodev,noexec,relatime,\${cg} \${cg} /sys/fs/cgroup/\${cg} &>/dev/null; fi; done; mkdir -p /sys/fs/cgroup/systemd; mount -t cgroup -o none,name=systemd systemd /sys/fs/cgroup/systemd; umount -Rl /sys/fs/cgroup/cg2_bpf; umount -Rl /sys/fs/cgroup/schedtune; umount -Rl '${LXC_ROOTFS_PATH}'; sed -i -E 's/^( *# *DNS=.*|DNS=.*)/DNS=1.1.1.1/g' '${LXC_ROOTFS_PATH}/etc/systemd/resolved.conf'; mount -B '${LXC_ROOTFS_PATH}' '${LXC_ROOTFS_PATH}'; mount -i -o remount,suid '${LXC_ROOTFS_PATH}'; if [ ! -e '${LXC_ROOTFS_PATH}/usr/bin/udevadm.' ]; then mv -f '${LXC_ROOTFS_PATH}/usr/bin/udevadm' '${LXC_ROOTFS_PATH}/usr/bin/udevadm.'; fi; echo -e '#!/usr/bin/bash\n/usr/bin/udevadm. \"\$@\" || true' > '${LXC_ROOTFS_PATH}/usr/bin/udevadm'; chmod +x '${LXC_ROOTFS_PATH}/usr/bin/udevadm'; true;"

# Necessary lxc container configuration that properly sets up the containers internals. Sets up required character files, correct cgroups, etc.
lxc.hook.pre-start = bash -c 'mkdir -p '"${LXC_ROOTFS_PATH}/etc/tmpfiles.d"'; echo -e "#Type Path       Mode User Group Age Argument\nc!     /dev/cuse  0666 root root  -   10:203\nc!     /dev/fuse  0666 root root  -   10:229\nc!     /dev/ashmem  0666 root root  -   10:58\nc!     /dev/loop-control  0600 root root  -   10:237" > '"${LXC_ROOTFS_PATH}/etc/tmpfiles.d/lxc-required-setup.conf"'; for i in $(seq -s " " 0 255); do echo "b!     /dev/loop${i}  0600 root root  -   7:$((${i} * 8))" >> '"${LXC_ROOTFS_PATH}/etc/tmpfiles.d/lxc-required-setup.conf"'; done; for i in binder hwbinder vndbinder; do echo "L!     /dev/${i}  - - -  -   /dev/binderfs/anbox-${i}" >> '"${LXC_ROOTFS_PATH}/etc/tmpfiles.d/lxc-required-setup.conf"'; done; echo -e "#!/usr/bin/bash\n\nsetup_lxc_configuration(){\n\nmount -o remount,rw /sys/fs/cgroup\numount -Rl /sys/fs/cgroup/{schedtune,cpu,cpuacct,'cpu,cpuacct'} &>/dev/null\nrm -rf /sys/fs/cgroup/{schedtune,cpu,cpuacct,'cpu,cpuacct'}\nmkdir -p /sys/fs/cgroup/{cpu,cpuacct}\nfor cg in cpu cpuacct; do\n  mount -t cgroup -o rw,nosuid,nodev,noexec,relatime,\${cg} \${cg} /sys/fs/cgroup/\${cg}\ndone\nmount -o remount,ro /sys/fs/cgroup\n\numount -Rl /dev/binderfs\n\nrm -rf /dev/binderfs\nmkdir -p /dev/binderfs\nmount -t binder binder /dev/binderfs\n\n}\n\nsetup_lxc_configuration &>/dev/null || true\n" > '"${LXC_ROOTFS_PATH}/etc/rc.local"'; chmod +x '"${LXC_ROOTFS_PATH}/etc/rc.local"'; true;'

# If container stopped then umount the bind mounted rootfs and restore it's nosuid if it was set
lxc.hook.post-stop = bash -c "umount -Rl '${LXC_ROOTFS_PATH}'; true;"
lxc.hook.destroy = bash -c "umount -Rl '${LXC_ROOTFS_PATH}'; true;"