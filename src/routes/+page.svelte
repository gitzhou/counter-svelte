<script lang="ts">
  import { onMount } from "svelte";
  import { Counter } from "../contracts/counter";
  import artifact from "../../artifacts/counter.json";

  import {
    Scrypt,
    bsv,
    ContractCalledEvent,
    SensiletSigner,
    ScryptProvider,
  } from "scrypt-ts";

  const contractId = {
    txId: "fdc5218259b0d8b2127873537339d58b7c8c1e19f0859b6bc3d3549d48d64a3c",
    outputIndex: 0,
  };

  let counterText = "N/A";
  let instance: Counter | null = null;
  let signer: SensiletSigner | null = null;

  onMount(() => {
    initScrypt();
    updateInstance();
    Scrypt.contractApi.subscribe(
      {
        clazz: Counter,
        id: contractId,
      },
      (event: ContractCalledEvent<Counter>) => {
        const txId = event.tx.id;
        console.log(`Counter increment: ${txId}`);
        updateInstance(event.nexts[0]);
      }
    );
  });

  function initScrypt() {
    Counter.loadArtifact(artifact);
    Scrypt.init({
      apiKey: "testnet_3OJHoUTWnhTtVGck0T6ZpV2Cx3lcLw0UchOfl4aPtfA8D10Kf",
      network: bsv.Networks.testnet,
    });

    signer = new SensiletSigner(new ScryptProvider());
  }

  async function updateInstance(newInstance: Counter | null = null) {
    if (!newInstance) {
      try {
        newInstance = await Scrypt.contractApi.getLatestInstance(
          Counter,
          contractId
        );
      } catch (e: any) {
        console.log(`Fetch instance error: ${e}`);
      }
    }
    instance = newInstance;
    counterText = instance!.count.toString();
  }

  const increment = async () => {
    if (instance && signer) {
      const { isAuthenticated, error } = await signer.requestAuth();
      if (!isAuthenticated) {
        throw new Error(error);
      }

      const currentInstance = instance;
      await currentInstance.connect(signer);

      const nextInstance = currentInstance.next();
      nextInstance.count++;

      currentInstance.methods
        .increment({
          next: {
            instance: nextInstance,
            balance: currentInstance.balance,
          },
        })
        .catch((e) => {
          console.log(`Counter call error: ${e}`);
          updateInstance();
        });
    }
  };
</script>

<h1>Counter now is {counterText}</h1>
<button on:click={increment}> Increment </button>
