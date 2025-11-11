# cluster-node-
this script  is multiple device node run 

## node_bootstrap.sh (run on each Node Mac once)

```
#!/usr/bin/env bash
# Usage: sudo bash node_bootstrap.sh <NODE_ID> <NODE_IP> [KEYDIR] [RPC_URL]
# Example: sudo bash node_bootstrap.sh 1000005432 192.168.1.45
set -euo pipefail

NODE_ID="${1:?NODE_ID missing}"
NODE_IP="${2:?NODE_IP missing}"
KEYDIR="${3:-$HOME/arx}"
RPC_URL="${4:-https://api.devnet.solana.com}"

mkdir -p "$KEYDIR"
cd "$KEYDIR"

echo "[*] Using KEYDIR=$KEYDIR, RPC_URL=$RPC_URL"
```

## 0) Solana endpoint

```
solana config set --url "$RPC_URL" >/dev/null
```


##  1) Keys (idempotent)

```
[[ -f node-kp.json     ]] || solana-keygen new --outfile node-kp.json --no-bip39-passphrase <<< y >/dev/null
[[ -f callback-kp.json ]] || solana-keygen new --outfile callback-kp.json --no-bip39-passphrase <<< y >/dev/null
[[ -f identity.pem     ]] || openssl genpkey -algorithm Ed25519 -out identity.pem >/dev/null 2>&1

NODE_ADDR="$(solana address -k node-kp.json)"
CALLBACK_ADDR="$(solana address -k callback-kp.json)"
```


## 2) Airdrop helper

```
airdrop_until() {
  local kp="$1" need="${2:-1}" tries=0
  while :; do
    bal="$(solana balance -k "$kp" 2>/dev/null | awk '{print $1}' || echo 0)"
    awk "BEGIN{exit !($bal >= $need)}" && break
    (( tries++ ))
    (( tries>6 )) && { echo "[!] Airdrop failed for $kp"; exit 1; }
    solana airdrop 2 "$(solana address -k "$kp")" >/dev/null || true
    sleep 3
  done
}

echo "[*] Funding signers (devnet faucet)…"
airdrop_until node-kp.json 1
airdrop_until callback-kp.json 1
```


## 3) Register node on-chain (idempotent safe to re-run)

```
echo "[*] Initializing on-chain accounts for NODE_ID=$NODE_ID IP=$NODE_IP"
arcium init-arx-accs \
  --keypair-path node-kp.json \
  --callback-keypair-path callback-kp.json \
  --peer-keypair-path identity.pem \
  --node-offset "$NODE_ID" \
  --ip-address "$NODE_IP" \
  --rpc-url "$RPC_URL"
```


## 4) Persist summary

```
cat > "$KEYDIR/node_info.txt" <<EOF
NODE_ID=$NODE_ID
NODE_IP=$NODE_IP
NODE_ADDR=$NODE_ADDR
CALLBACK_ADDR=$CALLBACK_ADDR
KEYDIR=$KEYDIR
EOF

echo "[✓] Node initialized. Save this NODE_ID for invite: $NODE_ID"
```

## 2) cluster_invite_from_csv.sh (run on Cluster Controller Mac)

```
#!/usr/bin/env bash
# Usage: bash cluster_invite_from_csv.sh <CLUSTER_OWNER_KEYPAIR.json> <CLUSTER_OFFSET> <NODES_CSV> [RPC_URL]
# CSV format: node_id (one per line) OR node_id,comment
set -euo pipefail

OWNER_KP="${1:?owner keypair missing}"
CLUSTER_OFFSET="${2:?cluster offset missing}"
CSV="${3:?csv missing}"
RPC_URL="${4:-https://api.devnet.solana.com}"

invite() {
  local node_id="$1"
  echo "[*] Inviting node_id=$node_id"
  arcium propose-join-cluster \
    --keypair-path "$OWNER_KP" \
    --cluster-offset "$CLUSTER_OFFSET" \
    --node-offset "$node_id" \
    --rpc-url "$RPC_URL"
}

while IFS=, read -r node_id _; do
  [[ -z "${node_id// }" ]] && continue
  [[ "$node_id" =~ ^# ]] && continue
  invite "$node_id"
done < "$CSV"

echo "[✓] All invites sent from $CSV"
```


##  Create a CSV file (example nodes.csv)

```
1000005432
1000005433
1000005434
```

##  … add all node_ids here (one per line)


## Run 

```
chmod +x cluster_invite_from_csv.sh
./cluster_invite_from_csv.sh cluster-owner-keypair.json 10021010 nodes.csv
```

## 3) node_accept.sh (run on each Node Mac after it’s invited)

```#!/usr/bin/env bash
# Usage: bash node_accept.sh <NODE_ID> <CLUSTER_OFFSET> [KEYDIR] [RPC_URL]
# Example: bash node_accept.sh 1000005432 10021010
set -euo pipefail

NODE_ID="${1:?NODE_ID missing}"
CLUSTER_OFFSET="${2:?CLUSTER_OFFSET missing}"
KEYDIR="${3:-$HOME/arx}"
RPC_URL="${4:-https://api.devnet.solana.com}"

cd "$KEYDIR"

echo "[*] Accepting invite for NODE_ID=$NODE_ID into CLUSTER_OFFSET=$CLUSTER_OFFSET"
arcium join-cluster true \
  --keypair-path node-kp.json \
  --node-offset "$NODE_ID" \
  --cluster-offset "$CLUSTER_OFFSET" \
  --rpc-url "$RPC_URL"

echo "[✓] Node joined cluster."
```


## Quick run

On every Node Mac 

```
chmod +x node_bootstrap.sh node_accept.sh
sudo ./node_bootstrap.sh <NODE_ID> <NODE_IP>
# Wait for invite, then:
./node_accept.sh <NODE_ID> 10021010
```

# On Cluster Mac

```
chmod +x cluster_invite_from_csv.sh
./cluster_invite_from_csv.sh cluster-owner-keypair.json 10021010 nodes.csv
```

## THIS COMPLETELY ERROR FREE NODE SCRIPT
