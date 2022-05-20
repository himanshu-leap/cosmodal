# Cosmodal

Connecting web applications to the Cosmos ecosystem.

## Preview

You can test the library on https://cosmodal.vercel.app/

![preview](./preview.png)

## Running example locally

```
yarn && yarn start

# OR

npm install && npm run start
```

## Usage

1. Install the Cosmodal package in your React project

```
yarn add cosmodal

# OR

npm install --save cosmodal
```

2. Import `WalletManagerProvider` and wrap it around your whole app. Only include it once as an ancestor of all components that need to access the wallet. Likely you'll want this in your root App component.

```tsx
import { FunctionComponent } from "react"
import { getKeplrFromWindow } from "@keplr-wallet/stores"
import { ChainInfo } from "@keplr-wallet/types"
import WalletConnect from "@walletconnect/client"
import {
  KeplrWalletConnectV1,
  Wallet,
  WalletClient,
  WalletManagerProvider,
} from "cosmodal"
import type { AppProps } from "next/app"
import Head from "next/head"
import { EmbedChainInfos } from "../config"

const CHAIN_ID = "juno-1"
const CHAIN_INFO: ChainInfo = "..."

const enableWallet = async (wallet: Wallet, client: WalletClient) => {
  // WalletConnect does not support suggesting chains.
  if (!(client instanceof KeplrWalletConnectV1)) {
    await client.experimentalSuggestChain(CHAIN_INFO)
  }
  await client.enable(CHAIN_ID)
}

const wallets: Wallet[] = [
  {
    id: "keplr-wallet-extension",
    name: "Keplr Wallet",
    description: "Keplr Browser Extension",
    logoImgUrl: "/keplr-wallet-extension.png",
    getWallet: () => getKeplrFromWindow(),
  },
  {
    id: "walletconnect-keplr",
    name: "WalletConnect",
    description: "Keplr Mobile",
    logoImgUrl: "/walletconnect-keplr.png",
    getWallet: (walletConnect?: WalletConnect) =>
      Promise.resolve(
        walletConnect
          ? new KeplrWalletConnectV1(walletConnect, EmbedChainInfos)
          : undefined
      ),
  },
]

const MyApp: FunctionComponent<AppProps> = ({ Component, pageProps }) => (
  <WalletManagerProvider
    enableKeplr={enableKeplr}
    clientMeta={{
      name: "CosmodalExampleDAPP",
      description: "A dapp using the cosmodal library.",
      url: "https://cosmodal.example.app",
      icons: ["https://cosmodal.example.app/walletconnect.png"],
    }}
    wallets={wallets}
  >
    <Component {...pageProps} />
  </WalletManagerProvider>
)

export default MyApp
```

3. Manage the wallet by using the `useWalletManager` hook in your components. You can use the hook in as many components as you want since the same objects are always returned (as long as there is only one WalletManagerProvider ancestor).

```tsx
import { Keplr } from "@keplr-wallet/types"
import { KeplrWalletConnectV1, useWalletManager, WalletClient } from "cosmodal"
import type { NextPage } from "next"
import { useEffect, useState } from "react"
import { CosmWasmClient } from "@cosmjs/cosmwasm-stargate"
import { GasPrice } from "@cosmjs/stargate"

const CHAIN_ID = "juno-1"
const CHAIN_RPC_ENDPOINT = "..."
const CW20_CONTRACT_ADDRESS = "..."
const RECIPIENT_ADDDRESS = "..."

const Home: NextPage = () => {
  const { connect, disconnect, connectedWallet, connectionError } =
    useWalletManager()

  const [name, setName] = useState("")
  const [address, setAddress] = useState("")
  useEffect(() => {
    if (!connectedWallet) return

    const { client } = connectedWallet

    // Get the name of the connected wallet.
    client.getKey(CHAIN_ID).then((key) => {
      setName(key.name)
    })
    // Get the address of the connected wallet.
    client.getAccounts().then((accounts) => {
      setAddress(accounts[0].address)
    })

    // Execute a contract to transfer some tokens.
    transfer(client)
  }, [connectedWallet])

  return connectedWallet ? (
    <div>
      <p>
        Name: <b>{name}</b>
      </p>
      <p>
        Address: <b>{address}</b>
      </p>
      <button onClick={disconnect}>Disconnect</button>
    </div>
  ) : (
    <div>
      <button onClick={connect}>Connect</button>
      {connectionError && <p>{connectionError.message}</p>}
    </div>
  )
}

export default Home

const transfer = async (client: WalletClient) => {
  const offlineSigner =
    // WalletConnect only supports the Amino signer.
    client instanceof KeplrWalletConnectV1
      ? await client.getOfflineSignerOnlyAmino(CHAIN_ID)
      : await client.getOfflineSignerAuto(CHAIN_ID)

  // Connect to the chain via an RPC node.
  const cwClient = await SigningCosmWasmClient.connectWithSigner(
    CHAIN_RPC_ENDPOINT,
    offlineSigner,
    {
      gasPrice: GasPrice.fromString("0.0025ujuno"),
    }
  )

  // Get the address of the connected wallet.
  const walletAddress = (await client.getAccounts())[0].address

  // Transfer 10 tokens from the wallet to the recipient address.
  const result = await cwClient.execute(
    walletAddress,
    CW20_CONTRACT_ADDRESS,
    {
      transfer: {
        amount: "10",
        recipient: RECIPIENT_ADDDRESS,
      },
    },
    "auto"
  )

  alert("Transferred 10 tokens in TX " + result.transactionHash)
}
```

## API

### Relevant types

```tsx
interface Wallet {
  // A unique identifier among all wallets.
  id: string
  // The name of the wallet.
  name: string
  // A description of the wallet.
  description: string
  // The URL of the wallet logo.
  imageUrl: string
  // If this wallet client uses WalletConnect.
  isWalletConnect: boolean
  // A function that returns an instantiated wallet client, with walletConnect passed if `isWalletConnect` is true.
  getClient: (
    walletConnect?: WalletConnect
  ) => Promise<WalletClient | undefined>
  // A function whose response is awaited right after the wallet is picked. If this throws an error, the selection process is interrupted, `connectionError` is set to the thrown error, and all modals are closed.
  onSelect?: () => Promise<void>
}

type EnableWalletFunction = (
  wallet: Wallet,
  client: WalletClient
) => Promise<void> | void

interface ModalClassNames {
  modalContent?: string
  modalOverlay?: string
  modalHeader?: string
  modalSubheader?: string
  modalCloseButton?: string
  walletList?: string
  wallet?: string
  walletImage?: string
  walletInfo?: string
  walletName?: string
  walletDescription?: string
  textContent?: string
}

interface IClientMeta {
  description: string
  url: string
  icons: string[]
  name: string
}

interface ConnectedWallet {
  wallet: Wallet
  client: WalletClient
}

interface WalletManagerContextInfo {
  connect: () => void
  disconnect: () => Promise<void>
  connectedWallet?: ConnectedWallet
  connectionError?: unknown
  isMobileWeb: boolean
}
```

### WalletManagerProvider

This component takes the following properties:

| Property                  | Type                   | Required | Description                                                                                                                                                                                                                                |
| ------------------------- | ---------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `wallets`                 | `Wallet[]`             | &#x2611; | Wallets available for connection.                                                                                                                                                                                                          |
| `enableWallet`            | `EnableWalletFunction` | &#x2611; | Function that enables the wallet once one is selected.                                                                                                                                                                                     |
| `classNames`              | `ModalClassNames`      |          | Class names applied to various components for custom theming.                                                                                                                                                                              |
| `closeIcon`               | `ReactNode`            |          | Custom close icon.                                                                                                                                                                                                                         |
| `preselectedWalletId`     | `string \| undefined`  |          | When set, the connect function will skip the selection modal and attempt to connect to this wallet immediately.                                                                                                                            |
| `walletConnectClientMeta` | `IClientMeta`          |          | Descriptive info about the React app which gets displayed when enabling a WalletConnect wallet (e.g. name, image, etc.).                                                                                                                   |
| `attemptAutoConnect`      | `boolean`              |          | If set to true on initial mount, the connect function will be called as soon as possible. If `preselectedWalletId` is also set and a wallet has previously connected and enabled, this can be used to seamlessly reconnect a past session. |
| `renderLoader`            | `() => ReactNode`      |          | A custom loader to display in a few modals, such as when enabling the wallet.                                                                                                                                                              |

### useWalletManager

This hook returns the following properties in an object (`WalletManagerContextInfo`):

| Property          | Type                           | Description                                                                                                                                                                 |
| ----------------- | ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `connect`         | `() => void`                   | Function to begin the connection process. This will either display the wallet picker modal or immediately attempt to connect to a wallet when `preselectedWalletId` is set. |
| `disconnect`      | `() => Promise<void>`          | Function that disconnects from the connected wallet.                                                                                                                        |
| `connectedWallet` | `ConnectedWallet \| undefined` | Connected wallet information and client.                                                                                                                                    |
| `connectionError` | `unknown`                      | Error encountered during the connection process. Can be anything since the `enableWallet` function can throw anything.                                                      |
| `isMobileWeb`     | `boolean`                      | If this app is running inside the Keplr Mobile web interface.                                                                                                               |

## Learn More

To learn more about Cosmodal API, please check [our example code](https://github.com/chainapsis/cosmodal/tree/main/example)

To learn more about how to use Keplr-specific API, please check the following resources:

- [Keplr Example](https://github.com/chainapsis/keplr-example)
- [Keplr Documentation](https://docs.keplr.app)

To learn more about JavaScript client library for Cosmos check out [CosmJS](https://github.com/chainapsis/keplr-example)
