# Celo Names - Cast (Foundry) Examples

All examples use Foundry's `cast` CLI. Set these environment variables first:

```bash
export CELO_RPC=https://forno.celo.org
export PRIVATE_KEY=0x...  # your wallet private key
export REGISTRAR=0x9Eb22700eFa1558eb2e0E522eB1DECC8025C3127
export REGISTRY=0x4d7912779679AFdC592CBd4674b32Fcb189395F7
export REVERSE_REGISTRAR=0xa58E81fe9b61B5c3fE2AFD33CF304c454AbFc7Cb
```

---

## Read Operations

### Check if a name is available

```bash
cast call $REGISTRAR "available(string)(bool)" "alice" --rpc-url $CELO_RPC
```

### Get price in native CELO (2-arg version)

```bash
cast call $REGISTRAR "rentPrice(string,uint64)(uint256)" "alice" 1 --rpc-url $CELO_RPC
```

Returns price in wei. Convert to CELO:

```bash
PRICE=$(cast call $REGISTRAR "rentPrice(string,uint64)(uint256)" "alice" 1 --rpc-url $CELO_RPC)
echo "Price: $(cast from-wei $PRICE) CELO"
```

### Get price in a specific ERC-20 token

```bash
# USDC price
cast call $REGISTRAR \
  "rentPrice(string,uint64,address)(uint256)" \
  "alice" 1 0xcebA9300f2b948710d2653dD7B07f33A8B32118C \
  --rpc-url $CELO_RPC
```

### Check name expiry

Requires computing the namehash first:

```bash
NODE=$(cast namehash "alice.celo.eth")
cast call $REGISTRY "expiries(bytes32)(uint256)" $NODE --rpc-url $CELO_RPC
```

### Check name owner

```bash
NODE=$(cast namehash "alice.celo.eth")
TOKEN_ID=$(cast to-uint256 $NODE)
cast call $REGISTRY "ownerOf(uint256)(address)" $TOKEN_ID --rpc-url $CELO_RPC
```

### Read text record

```bash
NODE=$(cast namehash "alice.celo.eth")
cast call $REGISTRY "text(bytes32,string)(string)" $NODE "avatar" --rpc-url $CELO_RPC
```

### Read address record

```bash
NODE=$(cast namehash "alice.celo.eth")
cast call $REGISTRY "addr(bytes32)(address)" $NODE --rpc-url $CELO_RPC
```

### Total registered names

```bash
cast call $REGISTRY "totalSupply()(uint256)" --rpc-url $CELO_RPC
```

---

## Write Operations

### Register a name with native CELO

```bash
LABEL="myname"
OWNER=$(cast wallet address --private-key $PRIVATE_KEY)
YEARS=1

# 1. Get price
PRICE=$(cast call $REGISTRAR "rentPrice(string,uint64)(uint256)" $LABEL $YEARS --rpc-url $CELO_RPC)

# 2. Register
cast send $REGISTRAR \
  "register(string,uint64,address,bytes[])" \
  $LABEL $YEARS $OWNER "[]" \
  --value $PRICE \
  --private-key $PRIVATE_KEY \
  --rpc-url $CELO_RPC
```

### Renew a name with native CELO

```bash
LABEL="myname"
YEARS=1

PRICE=$(cast call $REGISTRAR "rentPrice(string,uint64)(uint256)" $LABEL $YEARS --rpc-url $CELO_RPC)

cast send $REGISTRAR \
  "renew(string,uint64)" \
  $LABEL $YEARS \
  --value $PRICE \
  --private-key $PRIVATE_KEY \
  --rpc-url $CELO_RPC
```

### Set primary name (reverse record)

```bash
cast send $REVERSE_REGISTRAR \
  "setName(string)" \
  "myname.celo.eth" \
  --private-key $PRIVATE_KEY \
  --rpc-url $CELO_RPC
```

### Set a text record

```bash
NODE=$(cast namehash "myname.celo.eth")

cast send $REGISTRY \
  "setText(bytes32,string,string)" \
  $NODE "avatar" "https://example.com/avatar.png" \
  --private-key $PRIVATE_KEY \
  --rpc-url $CELO_RPC
```

### Set address record

```bash
NODE=$(cast namehash "myname.celo.eth")

cast send $REGISTRY \
  "setAddr(bytes32,address)" \
  $NODE 0xYourAddress \
  --private-key $PRIVATE_KEY \
  --rpc-url $CELO_RPC
```

### Transfer a name

```bash
NODE=$(cast namehash "myname.celo.eth")
TOKEN_ID=$(cast to-uint256 $NODE)
FROM=$(cast wallet address --private-key $PRIVATE_KEY)
TO=0xRecipientAddress

cast send $REGISTRY \
  "safeTransferFrom(address,address,uint256)" \
  $FROM $TO $TOKEN_ID \
  --private-key $PRIVATE_KEY \
  --rpc-url $CELO_RPC
```

### Edit records in one transaction (multicall)

This batches **address + multiple text updates** for the same node into a single transaction.

```bash
NODE=$(cast namehash "myname.celo.eth")

# Build each inner call's calldata (each element is a bytes-encoded function call).
CALL_SETADDR=$(cast calldata "setAddr(bytes32,address)" $NODE 0xYourAddress)
CALL_SETAVATAR=$(cast calldata "setText(bytes32,string,string)" $NODE "avatar" "https://example.com/avatar.png")
CALL_SETEMAIL=$(cast calldata "setText(bytes32,string,string)" $NODE "email" "me@domain.com")

# multicall expects: data[] where each item is one of the above calldata blobs.
DATA="[${CALL_SETADDR},${CALL_SETAVATAR},${CALL_SETEMAIL}]"

cast send $REGISTRY \
  "multicall(bytes[])" \
  "$DATA" \
  --private-key $PRIVATE_KEY \
  --rpc-url $CELO_RPC
```

---

## Scripts

### Full registration script

```bash
#!/bin/bash
set -euo pipefail

CELO_RPC=https://forno.celo.org
REGISTRAR=0x9Eb22700eFa1558eb2e0E522eB1DECC8025C3127
REVERSE_REGISTRAR=0xa58E81fe9b61B5c3fE2AFD33CF304c454AbFc7Cb

LABEL="${1:?Usage: ./register.sh <label> [years]}"
YEARS="${2:-1}"
OWNER=$(cast wallet address --private-key $PRIVATE_KEY)

echo "Registering ${LABEL}.celo.eth for ${YEARS} year(s)..."

# Check availability
AVAILABLE=$(cast call $REGISTRAR "available(string)(bool)" "$LABEL" --rpc-url $CELO_RPC)
if [ "$AVAILABLE" != "true" ]; then
  echo "Error: ${LABEL}.celo.eth is not available"
  exit 1
fi

# Get price
PRICE=$(cast call $REGISTRAR "rentPrice(string,uint64)(uint256)" "$LABEL" "$YEARS" --rpc-url $CELO_RPC)
echo "Price: $(cast from-wei $PRICE) CELO"

# Register
TX=$(cast send $REGISTRAR \
  "register(string,uint64,address,bytes[])" \
  "$LABEL" "$YEARS" "$OWNER" "[]" \
  --value "$PRICE" \
  --private-key $PRIVATE_KEY \
  --rpc-url $CELO_RPC \
  --json | jq -r '.transactionHash')

echo "Registered! TX: $TX"

# Set as primary name
echo "Setting ${LABEL}.celo.eth as primary name..."
cast send $REVERSE_REGISTRAR \
  "setName(string)" \
  "${LABEL}.celo.eth" \
  --private-key $PRIVATE_KEY \
  --rpc-url $CELO_RPC

echo "Done! ${LABEL}.celo.eth is registered and set as primary name."
```

### Batch check availability

```bash
#!/bin/bash
CELO_RPC=https://forno.celo.org
REGISTRAR=0x9Eb22700eFa1558eb2e0E522eB1DECC8025C3127

for name in alice bob charlie dave; do
  AVAILABLE=$(cast call $REGISTRAR "available(string)(bool)" "$name" --rpc-url $CELO_RPC)
  PRICE=$(cast call $REGISTRAR "rentPrice(string,uint64)(uint256)" "$name" 1 --rpc-url $CELO_RPC)
  echo "${name}.celo.eth - available: ${AVAILABLE}, price: $(cast from-wei $PRICE) CELO/year"
done
```

### Look up a name's info

```bash
#!/bin/bash
CELO_RPC=https://forno.celo.org
REGISTRY=0x4d7912779679AFdC592CBd4674b32Fcb189395F7

NAME="${1:?Usage: ./lookup.sh <name>}"
FULL_NAME="${NAME}.celo.eth"
NODE=$(cast namehash "$FULL_NAME")
TOKEN_ID=$(cast to-uint256 $NODE)

echo "=== ${FULL_NAME} ==="
echo "Node: $NODE"

OWNER=$(cast call $REGISTRY "ownerOf(uint256)(address)" $TOKEN_ID --rpc-url $CELO_RPC 2>/dev/null || echo "not registered")
echo "Owner: $OWNER"

EXPIRY=$(cast call $REGISTRY "expiries(bytes32)(uint256)" $NODE --rpc-url $CELO_RPC 2>/dev/null || echo "0")
if [ "$EXPIRY" != "0" ]; then
  echo "Expires: $(date -r $EXPIRY 2>/dev/null || date -d @$EXPIRY 2>/dev/null || echo $EXPIRY)"
fi

ADDR=$(cast call $REGISTRY "addr(bytes32)(address)" $NODE --rpc-url $CELO_RPC 2>/dev/null || echo "not set")
echo "ETH Address: $ADDR"

AVATAR=$(cast call $REGISTRY "text(bytes32,string)(string)" $NODE "avatar" --rpc-url $CELO_RPC 2>/dev/null || echo "not set")
echo "Avatar: $AVATAR"
```
