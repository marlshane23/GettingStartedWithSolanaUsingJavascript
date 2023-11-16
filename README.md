# GettingStartedWithSolanaUsingJavascript
This project provides a simple and straightforward guide for developers looking to get started with Solana using Javascript. It covers the creation of a new Solana account, connecting to a Phantom Wallet, and transferring SOL between accounts.

## Description
Solana is a high-performance blockchain that supports smart contracts and decentralized applications. It uses a unique proof-of-history consensus mechanism that makes it one of the fastest and most secure blockchains in the world. This project aims to help developers understand how to interact with the Solana blockchain using Javascript. It covers everything from setting up the development environment to sending transactions and interacting with smart contracts.

## Getting Started
1. Create the functionality for the button titled “Create a new Solana account” that generates a new KeyPair at the backend and airdrops 2 SOL to the newly created Keypair.
2. Once this step is completed, create the functionality for the next button that says - “Connect to Phantom Wallet”, which should connect to the Phantom Wallet if it exists.
3. Once this step is completed, create the final functionality for a button called “Transfer SOL to New Wallet”. This button should trigger a transfer of 1 SOL (airdropped into the account you generated in step 1) to the account connected in Step 2.

### Executing program

run npm dev
```
// import functionalities
import './App.css';
import {
  Connection,
  Keypair,
  LAMPORTS_PER_SOL,
  PublicKey,
  SystemProgram,
  Transaction,
  clusterApiUrl,
  sendAndConfirmTransaction,
} from "@solana/web3.js";
import { useEffect, useState } from "react";
import './App.css'

// import to fix polyfill issue with buffer with webpack
import * as buffer from "buffer";
window.Buffer = buffer.Buffer;


// create types
type DisplayEncoding = "utf8" | "hex";

type PhantomEvent = "disconnect" | "connect" | "accountChanged";
type PhantomRequestMethod =
  | "connect"
  | "disconnect"
  | "signTransaction"
  | "signAllTransactions"
  | "signMessage";

interface ConnectOpts {
  onlyIfTrusted: boolean;
}

// create a provider interface (hint: think of this as an object) to store the Phantom Provider
interface PhantomProvider {
  publicKey: PublicKey | null;
  isConnected: boolean | null;
  signTransaction: (transaction: Transaction) => Promise<Transaction>;
  signAllTransactions: (transactions: Transaction[]) => Promise<Transaction[]>;
  signMessage: (
    message: Uint8Array | string,
    display?: DisplayEncoding
  ) => Promise<any>;
  connect: (opts?: Partial<ConnectOpts>) => Promise<{ publicKey: PublicKey }>;
  disconnect: () => Promise<void>;
  on: (event: PhantomEvent, handler: (args: any) => void) => void;
  request: (method: PhantomRequestMethod, params: any) => Promise<unknown>;
}

/**
* @description gets Phantom provider, if it exists
*/
const getProvider = (): PhantomProvider | undefined => {
  if ("solana" in window) {
    // @ts-ignore
    const provider = window.solana as any;
    if (provider.isPhantom) return provider as PhantomProvider;
  }
};

export default function App() {
  // create state variable for the provider
  const [provider, setProvider] = useState<PhantomProvider | undefined>(
    undefined
  );

  // create state variable for the phantom wallet key
  const [receiverPublicKey, setReceiverPublicKey] = useState<PublicKey | undefined>(
    undefined
  );

  // create state variable for the sender wallet key
  const [senderKeypair, setSenderKeypair] = useState<Keypair | undefined>(
    undefined
  );

  // create a state variable for our connection
  const connection = new Connection(clusterApiUrl("devnet"), "confirmed");
  
  // connection to use with local solana test validator
  // const connection = new Connection("http://127.0.0.1:8899", "confirmed");

  // this is the function that runs whenever the component updates (e.g. render, refresh)
  useEffect(() => {
    const provider = getProvider();

    // if the phantom provider exists, set this as the provider
    if (provider) setProvider(provider);
    else setProvider(undefined);
  }, []);

  /**
   * @description creates a new KeyPair and airdrops 2 SOL into it.
   * This function is called when the Create a New Solana Account button is clicked
   */
  const createSender = async () => {
      try   {   // create a new Keypair
      const newSenderKeypair = Keypair.generate();

      console.log('Sender account: ', newSenderKeypair.publicKey.toString());
      console.log('Airdropping 2 SOL to Sender Wallet');

      // request airdrop into this new account
      const airdropSignature = await connection.requestAirdrop(
        newSenderKeypair.publicKey,
        LAMPORTS_PER_SOL * 2
      );

      // wait for airdrop confirmation
      await connection.confirmTransaction(airdropSignature);

      // update the state with the new keypair
      setSenderKeypair(newSenderKeypair);

      console.log('Airdrop successful');
      console.log('Wallet Balance: ' + (await connection.getBalance(newSenderKeypair.publicKey)) / LAMPORTS_PER_SOL);
    } catch (err) {
      console.error('Error creating account and airdropping SOL:', err);
    }
  }

  /**
   * @description prompts user to connect wallet if it exists.
   * This function is called when the Connect to Phantom Wallet button is clicked
   */
  const connectWallet = async () => {
    // @ts-ignore
    const { solana } = window;

    // checks if phantom wallet exists
    if (solana) {
      try {
        // connect to phantom wallet and return response which includes the wallet public key
        const connectedWallet = await solana.connect();

        // save the public key of the phantom wallet to the state variable
        setReceiverPublicKey(new PublicKey(connectedWallet.publicKey));

        console.log('Connected to Phantom Wallet');
      } catch (err) {
        console.error('Error connecting to Phantom Wallet:', err);
      }
    }
  };

  /**
   * @description disconnects wallet if it exists.
   * This function is called when the disconnect wallet button is clicked
   */
  const disconnectWallet = async () => {
    // @ts-ignore
    const { solana } = window;

    // checks if phantom wallet exists
    if (solana) {
      try {
        solana.disconnect();
        setReceiverPublicKey(undefined);
        console.log("wallet disconnected")
      } catch (err) {
        console.log(err);
      }
    }
  };

  /**
   * @description transfer SOL from sender wallet to connected wallet.
   * This function is called when the Transfer SOL to Phantom Wallet button is clicked
   */
  const transferSol = async () => {
    try {
      if (!senderKeypair || !receiverPublicKey) {
        console.error('Sender keypair or receiver public key is undefined');
        return;
      }

      // create a new transaction for the transfer
      const transaction = new Transaction().add(
        SystemProgram.transfer({
          fromPubkey: senderKeypair.publicKey,
          toPubkey: receiverPublicKey,
          lamports: LAMPORTS_PER_SOL, // Transfer 1 SOL
        })
      );

      // send and confirm the transaction
    const signature = await sendAndConfirmTransaction(
        connection,
        transaction,
        [senderKeypair]
      );

      console.log('Transaction sent and confirmed:', signature);
      console.log('Sender Balance: ' + (await connection.getBalance(senderKeypair.publicKey)) / LAMPORTS_PER_SOL);
      console.log('Receiver Balance: ' + await connection.getBalance(receiverPublicKey) / LAMPORTS_PER_SOL);
    } catch (err) {
      console.error('Error transferring SOL:', err);
    }
  };
```

## Authors
Marl Shane G. Esteron
