#!/usr/bin/env bash
set -euo pipefail

MIRROR_ROOT="/mnt/volume-arch-mirror"
RSYNC_SOURCE="rsync://rsync.archlinux.org/archlinux/" # Switch to a specific Tier-1 later if you like
RSYNC_BWLIMIT="${RSYNC_BWLIMIT:-0}"                   # KB/s (0 = unlimited)
RSYNC_TIMEOUT=600
RSYNC_CONNECT_TIMEOUT=60
LOG_DIR="/var/log/archmirror"

require_root() {
  [[ $EUID -eq 0 ]] || {
    echo "Run as root."
    exit 1
  }
}

detect_pkg_manager() {
  if command -v apt-get >/dev/null 2>&1; then
    PKG_MGR="apt"
  elif command -v dnf >/dev/null 2>&1; then
    PKG_MGR="dnf"
  elif command -v yum >/dev/null 2>&1; then
    PKG_MGR="yum"
  elif command -v pacman >/dev/null 2>&1; then
    PKG_MGR="pacman"
  else
    echo "Need apt, dnf, yum, or pacman."
    exit 1
  fi
}

install_packages() {
  case "$PKG_MGR" in
  apt)
    apt-get update -y
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends rsync nginx ca-certificates
    ;;
  dnf)
    dnf install -y rsync nginx
    systemctl enable --now nginx || true
    ;;
  yum)
    yum install -y rsync nginx
    systemctl enable --now nginx || true
    ;;
  pacman) pacman -Sy --noconfirm rsync nginx ;;
  esac
}

prep_dirs() {
  mkdir -p "$MIRROR_ROOT" "$LOG_DIR" /var/lock
  chown -R root:root "$MIRROR_ROOT" "$LOG_DIR"
  chmod 755 "$MIRROR_ROOT"
}

install_rsync_script() {
  cat >/usr/local/bin/archmirror-rsync.sh <<'RSYNC_SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

MIRROR_ROOT="$MIRROR_ROOT"
RSYNC_SOURCE="$RSYNC_SOURCE"
LOG_FILE="/var/log/archmirror/rsync.log"
LOCK_FILE="/var/lock/archmirror-rsync.lock"
RSYNC_BWLIMIT="$RSYNC_BWLIMIT"  # KB/s
RSYNC_TIMEOUT="$RSYNC_TIMEOUT"
RSYNC_CONNECT_TIMEOUT="$RSYNC_CONNECT_TIMEOUT"

EXCLUDES=( "--exclude=.*~tmp~" "--exclude=/iso" )   # packages-only

mkdir -p "$(dirname "$LOG_FILE")"; : > "$LOG_FILE"

exec 9>"$LOCK_FILE"
flock -n 9 || { echo "$(date -Is) Another sync is running, exiting." | tee -a "$LOG_FILE"; exit 0; }

echo "$(date -Is) Starting rsync from $RSYNC_SOURCE" | tee -a "$LOG_FILE"

rsync \
  -rtlH --safe-links --delete-after --delay-updates \
  --timeout="$RSYNC_TIMEOUT" --contimeout="$RSYNC_CONNECT_TIMEOUT" \
  --no-motd \
  --partial --partial-dir=.rsync-partial \
  --bwlimit="$RSYNC_BWLIMIT" \
  "${EXCLUDES[@]}" \
  "$RSYNC_SOURCE" "$MIRROR_ROOT" | tee -a "$LOG_FILE"

echo "$(date -Is) Sync finished" | tee -a "$LOG_FILE"
RSYNC_SCRIPT
  chmod +x /usr/local/bin/archmirror-rsync.sh

  cat >/etc/default/archmirror-rsync <<EOF
MIRROR_ROOT="$MIRROR_ROOT"
RSYNC_SOURCE="$RSYNC_SOURCE"
LOG_FILE="$LOG_DIR/rsync.log"
LOCK_FILE="/var/lock/archmirror-rsync.lock"
RSYNC_BWLIMIT="$RSYNC_BWLIMIT"
RSYNC_TIMEOUT="$RSYNC_TIMEOUT"
RSYNC_CONNECT_TIMEOUT="$RSYNC_CONNECT_TIMEOUT"
EOF
}

install_systemd_units() {
  cat >/etc/systemd/system/archmirror-sync.service <<'UNIT'
[Unit]
Description=Sync Arch Linux mirror (packages only) via rsync
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
EnvironmentFile=/etc/default/archmirror-rsync
ExecStart=/usr/local/bin/archmirror-rsync.sh
Nice=10
IOSchedulingClass=best-effort
IOSchedulingPriority=7
ProtectSystem=full
ProtectHome=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
UNIT

  cat >/etc/systemd/system/archmirror-sync.timer <<'TIMER'
[Unit]
Description=Run Arch mirror sync every 15 minutes

[Timer]
OnBootSec=2min
OnUnitActiveSec=15min
RandomizedDelaySec=1min
Persistent=true

[Install]
WantedBy=timers.target
TIMER

  systemctl daemon-reload
  systemctl enable --now archmirror-sync.timer
}

install_nginx_config() {
  local conf_path
  if [[ -d /etc/nginx/sites-available ]]; then
    conf_path="/etc/nginx/sites-available/archmirror.conf"
  else
    conf_path="/etc/nginx/conf.d/archmirror.conf"
  fi

  cat >"$conf_path" <<EOF
server {
    listen 80;

    location /archlinux/ {
        root ${MIRROR_ROOT%/}/..;   # so /archlinux maps to $MIRROR_ROOT
        autoindex on;
        sendfile on;
        tcp_nopush on;
        aio on;
        expires 1h;
        etag on;
        gzip off;
        types {
            application/octet-stream pkg;
            application/zstd zst;
        }
    }

    location = / { return 302 /archlinux/; }

    access_log /var/log/nginx/archmirror_access.log;
    error_log  /var/log/nginx/archmirror_error.log;
}
EOF

  if [[ -d /etc/nginx/sites-available ]]; then
    ln -sf "$conf_path" /etc/nginx/sites-enabled/archmirror.conf
    [[ -f /etc/nginx/sites-enabled/default ]] && rm -f /etc/nginx/sites-enabled/default || true
  fi

  nginx -t
  systemctl enable --now nginx
  systemctl restart nginx
}

initial_sync() {
  echo "Starting initial sync (packages only)…"
  /usr/local/bin/archmirror-rsync.sh
}

summary() {
  local host
  host=$(hostname -f 2>/dev/null || hostname)
  cat <<EOF

✅ Arch packages-only mirror is ready.

Mirror root: $MIRROR_ROOT
Upstream:    $RSYNC_SOURCE
Logs:        $LOG_DIR/rsync.log
HTTP:        http://$host/archlinux/

Use this in /etc/pacman.d/mirrorlist:
  Server = http://$host/archlinux/\$repo/os/\$arch

Check timer:
  systemctl list-timers --all | grep archmirror-sync.timer || true
EOF
}

main() {
  require_root
  detect_pkg_manager
  install_packages
  prep_dirs
  install_rsync_script
  install_systemd_units
  install_nginx_config
  initial_sync
  summary
}

main "$@"
