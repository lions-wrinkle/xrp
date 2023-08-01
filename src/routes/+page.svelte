<script>
  import { onDestroy, onMount } from "svelte";
  import { Client } from "xrpl";
  import { XummPkce } from "xumm-oauth2-pkce";

  /** @type {Client}*/
  let client;
  let xumm;
  let state;

  /** @type {String} */
  let address;

  /** @type {import("xrpl").AccountNFToken[]} */
  let nftsRaw = [];

  let nftsLoaded = [];

  $: address && loadNFTs(address);

  onMount(async () => {
    console.log("mount")
    connectAll();
  });

  onDestroy(() => {});

  async function connectAll(){
    await connectXRP();
    await connectXumm();
  }

  async function connectXRP() {
    console.log("connect XRP...")
    const PUBLIC_SERVER = "wss://xrplcluster.com/";
    client = new Client(PUBLIC_SERVER);
    await client.connect();
  }

  async function connectXumm() {
    console.log("connect xumm...")
    xumm = new XummPkce("1eb84150-a89d-4874-a139-ce0053747639", {
      implicit: true, // Implicit: allows to e.g. move from social browser to stock browser
      redirectUrl: document.location.href + "?custom_state=test",
    });

    xumm.on("error", (error) => {
      console.log("error", error);
    });

    xumm.on("success", async () => {
      state = await xumm.state(); // state.sdk = instance of https://www.npmjs.com/package/xumm-sdk
      address = state?.me?.account;
      console.log("sucesss");
      console.log(state);
    });

    xumm.on("retrieved", async () => {
      console.log("Retrieved: from localStorage or mobile browser redirect");
      state = await xumm.state(); // state.sdk = instance of https://www.npmjs.com/package/xumm-sdk
      address = state?.me?.account;
      console.log("retrieved");
      console.log(state);
    });
  }

  function disconnectXRP() {
    client.disconnect();
  }

  function login() {
    xumm.authorize().catch((e) => console.log("e", e));
  }

  function logout() {
    console.log("logout");
    xumm.logout();
    state = undefined
  }

  /**
   *
   * @param {String} address
   */
  async function loadNFTs(address) {

    /*if (!client){
      return
    }*/

    const response = await client.request({
      command: "account_nfts",
      account: address,
      ledger_index: "validated",
    });

    console.log(response);

    nftsRaw = response.result.account_nfts.filter((nft) => nft.URI);
    nftsLoaded = Array(nftsRaw.length).fill(undefined);

    for (const nftRaw of nftsRaw) {
      await loadNFT(nftRaw);
    }
  }

  /**
   *
   * @param {String} hex
   */
  function hex2a(hex) {
    var str = "";
    for (var i = 0; i < hex.length; i += 2) {
      var v = parseInt(hex.substr(i, 2), 16);
      if (v) str += String.fromCharCode(v);
    }
    return str;
  }

  /**
   *
   * @param {String} uri
   */
  function parseUri(uri) {
    if (!uri) {
      return;
    }
    const gateawayUrl = "https://ipfs.io/ipfs/";
    if (uri.startsWith("ipfs://")) {
      return gateawayUrl + uri.slice(7, uri.length);
    } else if (uri.startsWith("http://") || uri.startsWith("https://")) {
      return uri;
    } else {
      //guess that it's an ipfs cid
      return gateawayUrl + uri;
    }
  }

  /**
   *
   * @param {import("xrpl").AccountNFToken} nft
   */
  async function loadNFT(nft) {
    if (!nft?.URI) {
      return;
    }

    const uri = hex2a(nft.URI);

    let nftInfo = null;

    let jsonUrl = parseUri(uri);

    if (jsonUrl) {
      //load content
      const res = await fetch(jsonUrl);

      if (res.status === 200) {
        const contentType = res.headers.get("Content-Type");

        if (contentType) {
          nftInfo = {};
          if (contentType.includes("application/json")) {
            nftInfo.json = await res.json();
            nftInfo.image = parseUri(nftInfo.json.image);
          } else if (contentType.includes("image")) {
            nftInfo.image = jsonUrl;
          }
        }
      }
    }

    console.log(nftInfo);

    //find nft index
    const index = nftsRaw.findIndex((n) => n.NFTokenID === nft.NFTokenID);
    nftsLoaded[index] = nftInfo;
  }
</script>

<h1>XRP Test</h1>
<div>
  {#if state?.me}
    <button on:click={logout}>Logout</button>
  {:else}
    <button on:click={login}>Login</button>
  {/if}
</div>
<p>
  {#if client}
    Connected to XRPL
  {:else}
    Connecting...
  {/if}
  {#if state?.me}
    {address}

    <div>
      {nftsLoaded.length} NFTs
    </div>

    <ul>
      {#each nftsLoaded as nft}
        <li>
          {#if nft === undefined}
            Loading...
          {:else if nft === null}
            Failed
          {:else}
            {nft.json ? nft.json.name : "- no name -"}
            <br />
            <img
              src={nft.image}
              style="max-width: 16rem; aspect-ratio: 1 / 1;"
              alt={nft.json ? nft.json.name : "- no name -"}
              loading="lazy"
            /><br />
            {#if nft.json?.description}
              <small> {nft.json.description}</small>
            {/if}
            <br /><br />
          {/if}
        </li>
      {/each}
    </ul>
  {/if}
</p>
