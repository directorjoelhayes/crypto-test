<script setup>
import { onMounted, reactive, ref } from "vue";
import { ulid } from "ulid";

const dropBox = reactive({
  items: new Map(),
  nonces: new Map(),
  receipts: new Map(),
  public: null,
  private: null,

  addItem(key, item, nonce) {
    this.items.set(key, item);
    this.nonces.set(key, nonce);
  },
  removeItem(key) {
    this.items.delete(key);
    this.nonces.delete(key);
  },
  async addReceipt(receipt) {
    //decode the receipt
    const decryptedReceipt = await SEA.decrypt(
      receipt,
      await SEA.secret(dropBox.private.a, dropBox.private.b)
    );

    const itemNonce = this.nonces.get(decryptedReceipt.itemKey);

    if (itemNonce !== decryptedReceipt.nonce) {
      console.log("nonce mismatch");
      return;
    }

    const receiptId = ulid();

    this.receipts.set(receiptId, decryptedReceipt);
  },
  async initialize() {
    const boxPair = await SEA.pair();
    this.public = {
      a: boxPair.epub,
      b: boxPair,
    };
    this.private = {
      a: boxPair.epub,
      b: boxPair,
    };
  },
});

async function makePair() {
  const key1 = await SEA.pair();
  const key2 = await SEA.pair();
  return {
    public: {
      a: key1.epub,
      b: key2,
    },
    private: {
      a: key2.epub,
      b: key1,
    },
  };
}

async function makeNode(onFinalCallback) {
  const node = await makePair();
  const id = ulid();
  return {
    public: node.public,
    private: node.private,
    id,
    scanInterval: null,
    scan() {
      this.scanInterval = setInterval(async () => {
        const items = dropBox.items;
        for (const [key, item] of items) {
          const decryptedKey = await SEA.decrypt(
            key,
            await SEA.secret(node.private.a, node.private.b)
          );
          if (!decryptedKey) continue;

          if (decryptedKey.startsWith("address:")) {
            this.stopScan();
            // dropBox.addReceipt(pickupReceipt);

            const itemCopy = item;
            const dataAggregator = [];

            const decryptedItem = await SEA.decrypt(
              itemCopy,
              await SEA.secret(node.private.a, node.private.b)
            );

            const pickupNonce = decryptedItem.nonce;
            const publicKey = decryptedItem.payloadSecret;

            const pickupReceipt = await SEA.encrypt(
              JSON.stringify({
                publicKey: await SEA.encrypt(
                  "pickup at: " + new Date().toISOString(),
                  await SEA.secret(publicKey.a, publicKey.b)
                ),
                action: "pickup",
                itemKey: key,
                nonce: pickupNonce,
                timestamp: Date.now(),
              }),
              await SEA.secret(dropBox.public.a, dropBox.public.b)
            );

            dropBox.addReceipt(pickupReceipt);

            let foundAddress = false;

            for (const address of decryptedItem.addresses) {
              const decryptedAddress = await SEA.decrypt(
                address,
                await SEA.secret(node.private.a, node.private.b)
              );
              if (!decryptedAddress) continue;

              if (decryptedAddress.startsWith("address:")) {
                foundAddress = true;
                await passOnPayload(decryptedAddress, decryptedItem);
              }
            }

            if (!foundAddress) {
              console.log("Last node reached:", this.id);
              this.onFinal(decryptedItem.data, node);
            }

            async function passOnPayload(decryptedAddress, decryptedItem) {
              const publicKey = JSON.parse(
                decryptedAddress.split("address:")[1]
              );
              const address = await SEA.encrypt(
                "address:" + ulid(),
                await SEA.secret(publicKey.a, publicKey.b)
              );

              const addedData = id;

              const payloadSecret = decryptedItem.payloadSecret;

              const encryptedAddedData = await SEA.encrypt(
                addedData,
                await SEA.secret(payloadSecret.a, payloadSecret.b)
              );

              const data = [...decryptedItem.data, encryptedAddedData];

              console.log(data, "data");

              const newNonce = ulid();

              const encryptedPayload = await SEA.encrypt(
                JSON.stringify({
                  ...decryptedItem,
                  data: data,
                  nonce: newNonce,
                }),
                await SEA.secret(publicKey.a, publicKey.b)
              );
              dropBox.addItem(address, encryptedPayload, newNonce);
            }
          }
        }
      }, 1000);
    },
    stopScan() {
      clearInterval(this.scanInterval);
    },
    async send(address, payload) {
      const encrypted = await SEA.encrypt(
        payload,
        await SEA.secret(this.private.a, this.private.b)
      );
      dropBox.addItem(address, encrypted);
    },
    onFinal(data, node) {
      onFinalCallback(data, node);
    },
  };
}

const finalData = ref("");

onMounted(async () => {
  await dropBox.initialize();

  let starterNode = "";
  
  starterNode = await makeNode(async (data, node) => {
    console.log(node, starterNode, "node");
    
    // Use Promise.all to wait for all decryption operations
    const decryptedItems = await Promise.all(data.map(async (item) => {
      return await SEA.decrypt(
        item,
        await SEA.secret(node.private.a, node.private.b)
      );
    }));

    console.log(decryptedItems, "decrypted items");
    finalData.value = decryptedItems.join("");
  });

  const nodes = [
    await makeNode(),
    await makeNode(),
    await makeNode(),
    await makeNode(),
    await makeNode(),
    await makeNode(),
    await makeNode(),
    await makeNode(),
    starterNode,
  ];

  const encryptedAddresses = await Promise.all(
    nodes.map(async (node, index) => {
      if (index < nodes.length - 1) {
        return await SEA.encrypt(
          "address:" + JSON.stringify(nodes[index + 1].public),
          await SEA.secret(node.public.a, node.public.b)
        );
      }
      return null;
    })
  );

  const validAddresses = encryptedAddresses.filter(
    (address) => address !== null
  );
  const nonce = ulid();

  const payload = {
    addresses: validAddresses,
    payloadSecret: starterNode.public,
    data: [],
    nonce,
  };

  console.log(payload, "payload!!");

  const encryptedPayload = await SEA.encrypt(
    JSON.stringify(payload),
    await SEA.secret(nodes[0].public.a, nodes[0].public.b)
  );

  const encryptedAddress = await SEA.encrypt(
    "address:" + ulid(),
    await SEA.secret(nodes[0].public.a, nodes[0].public.b)
  );

  dropBox.addItem(encryptedAddress, encryptedPayload, nonce);

  nodes.map((node) => {
    node.scan();
  });
});
</script>

<template>
  <div>
    <h1>Hello World</h1>
    <div class="receipts">
      <div v-for="receipt in dropBox.receipts" :key="receipt">
        {{ receipt[0] }} : {{ receipt[1].nonce }}
      </div>
    </div>
    <div class="final-data">
      {{ finalData }}
    </div>
  </div>
</template>

<style scoped>
.receipts {
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.final-data {
    position: relative;
    overflow: hidden; 
    width: 500px;
    overflow-wrap: break-word;
    background-color: #af3248;
    border-radius: 20px;
    padding: 20px;
}

</style>
