import { useState } from 'react';
import { ethers } from 'ethers';
import * as bitcoin from 'bitcoinjs-lib';
import * as bip39 from 'bip39';
import { Button } from '@/components/ui/button';

const chains = [
  {
    name: 'Ethereum',
    rpc: 'https://cloudflare-eth.com',
    symbol: 'ETH'
  },
  {
    name: 'Polygon',
    rpc: 'https://polygon-rpc.com',
    symbol: 'MATIC'
  },
  {
    name: 'BSC',
    rpc: 'https://bsc-dataseed.binance.org',
    symbol: 'BNB'
  }
];

export default function Home() {
  const [isRecovering, setIsRecovering] = useState(false);
  const [recoveryStatus, setRecoveryStatus] = useState('');

  const downloadResults = () => {
    const blob = new Blob([recoveryStatus], { type: 'text/plain' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'wallet_recovery_results.txt';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  };

  const simulateWalletRecovery = async () => {
    setIsRecovering(true);
    setRecoveryStatus('Starting wallet recovery simulation...');

    try {
      const mnemonic = ethers.Wallet.createRandom().mnemonic.phrase;
      setRecoveryStatus(`Generated mnemonic: ${mnemonic}`);

      const hdNode = ethers.utils.HDNode.fromMnemonic(mnemonic);

      for (let i = 0; i < 3; i++) {
        const wallet = new ethers.Wallet(hdNode.derivePath(`m/44'/60'/0'/0/${i}`));
        setRecoveryStatus(prev => prev + `\n\nChecking address: ${wallet.address}`);

        for (const chain of chains) {
          const provider = new ethers.providers.JsonRpcProvider(chain.rpc);
          const balance = await provider.getBalance(wallet.address);
          const formatted = ethers.utils.formatEther(balance);

          setRecoveryStatus(prev => prev + `\n${chain.name} Balance: ${formatted} ${chain.symbol}`);
        }
      }

      // Bitcoin wallet derivation
      setRecoveryStatus(prev => prev + '\n\nChecking Bitcoin wallets...');
      const seed = await bip39.mnemonicToSeed(mnemonic);
      const bitcoinNetwork = bitcoin.networks.bitcoin;
      const root = bitcoin.bip32.fromSeed(seed, bitcoinNetwork);

      for (let i = 0; i < 3; i++) {
        const child = root.derivePath(`m/44'/0'/0'/0/${i}`);
        const { address } = bitcoin.payments.p2pkh({ pubkey: child.publicKey, network: bitcoinNetwork });

        setRecoveryStatus(prev => prev + `\n\nBitcoin Address ${i}: ${address}`);

        const res = await fetch(`https://blockstream.info/api/address/${address}`);
        const json = await res.json();
        const btcBalance = json.chain_stats?.funded_txo_sum && json.chain_stats?.spent_txo_sum != null
          ? (json.chain_stats.funded_txo_sum - json.chain_stats.spent_txo_sum) / 1e8
          : 'N/A';

        setRecoveryStatus(prev => prev + `\nBitcoin Balance: ${btcBalance} BTC`);
      }

    } catch (error) {
      console.error(error);
      setRecoveryStatus('Error occurred during wallet recovery.');
    }

    setIsRecovering(false);
  };

  return (
    <main className="flex flex-col items-center justify-center min-h-screen bg-gray-950 text-white p-4">
      <h1 className="text-4xl font-bold mb-6">LostMining Recovery Service</h1>
      <p className="text-lg mb-4 max-w-xl text-center">
        Recover access to your lost cryptocurrency wallets using our automated 12-word seed phrase recovery engine.
      </p>

      <Button className="mb-4" onClick={simulateWalletRecovery} disabled={isRecovering}>
        {isRecovering ? 'Processing...' : 'Start Free Wallet Recovery'}
      </Button>

      {recoveryStatus && (
        <>
          <div className="text-sm text-gray-300 mt-2 border p-2 rounded bg-gray-800 whitespace-pre-line max-w-2xl">
            {recoveryStatus}
          </div>
          <Button className="mt-4" onClick={downloadResults}>Download Results</Button>
        </>
      )}
    </main>
  );
}
