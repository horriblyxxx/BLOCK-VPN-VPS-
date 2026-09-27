# BLOCK-VPN-VPS-
block VPN VPs and any other 
COMMANDS 

sudo bash <<'END'
set -e
apt-get update
apt-get -y install curl nftables
systemctl enable --now nftables

nft flush ruleset
nft -f - <<'NFT'
table inet filter {
  set vpnblock {
    type ipv4_addr
    flags interval
    auto-merge
  }
  chain input {
    type filter hook input priority 0; policy drop;
    ct state established,related accept
    iif "lo" accept
    ip protocol icmp accept
    ip saddr @vpnblock drop
    tcp dport 22 accept
  }
  chain forward { type filter hook forward priority 0; policy drop; }
  chain output { type filter hook output priority 0; policy accept; }
}
NFT

tmp=$(mktemp)
curl -fsSL https://raw.githubusercontent.com/X4BNet/lists_vpn/main/output/vpn/ipv4.txt | awk '/^[0-9]/ {print $1}' >"$tmp" || true
curl -fsSL https://check.torproject.org/torbulkexitlist | grep -E '^[0-9.]+$' >>"$tmp" || true
sort -u "$tmp" | split -l 400 - /tmp/vpnc_
for f in /tmp/vpnc_*; do
  [ -s "$f" ] || continue
  els=$(paste -sd, "$f")
  nft add element inet filter vpnblock { $els } || true
done
rm -f "$tmp" /tmp/vpnc_*
nft list ruleset >/etc/nftables.conf
echo DONE
END
