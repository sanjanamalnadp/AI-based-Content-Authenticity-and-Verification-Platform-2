# Digital Content Verification Platform

A full-stack prototype for registering and verifying digital content provenance with SHA-256 hashes, IPFS storage, and an on-chain Solidity registry.

The app lets a user upload a file, pin it to IPFS through Pinata, and store an immutable proof on a blockchain. Later, the same file can be uploaded to verify whether its SHA-256 fingerprint already exists in the registry.

## Features

- Register images, videos, audio files, and common document formats.
- Compute a SHA-256 fingerprint from the original file bytes.
- Apply a visible watermark to image uploads with Jimp.
- Compute a perceptual hash for images when image processing succeeds.
- Pin registered files to IPFS using Pinata.
- Store `sha256Hash`, `pHash`, `ipfsCid`, creator address, and timestamp in a Solidity smart contract.
- Verify uploaded files against the on-chain registry.
- Detect duplicate registrations by SHA-256.
- Expose backend health checks for RPC, contract, Pinata, CORS, and watermark configuration.

## Tech Stack

| Area             | Technology                          |
| ---------------- | ----------------------------------- |
| Frontend         | React 19, Vite, Axios               |
| Backend          | Node.js, Express, Multer, Ethers.js |
| Image Processing | Jimp                                |
| Hashing          | Node.js `crypto`, Jimp image hash   |
| Blockchain       | Solidity, Hardhat 3, Ethers.js      |
| Storage          | IPFS through Pinata                 |

## Project Structure

```text
.
+-- backend/
|   +-- index.js                  # Express API
|   +-- src/
|   |   +-- GenesisRegistry.json  # Contract ABI used by backend
|   |   +-- hashUtils.js          # SHA-256, image hash, watermark helpers
|   |   +-- watermark.py          # Older OpenCV watermark script, not used by current API
|   +-- blockchain/
|       +-- contracts/GenesisRegistry.sol
|       +-- hardhat.config.ts
|       +-- ignition/modules/DeployRegistry.ts
|       +-- scripts/deploy.js
+-- frontend/
|   +-- src/App.jsx               # Register and verify UI
+-- README.md
```

## Requirements

- Node.js and npm
- A Pinata account and API credentials
- A local Hardhat node for local development, or a funded wallet and RPC URL for testnet deployment

## Local Development

Install dependencies in each app folder:

```bash
npm install
cd backend
npm install
cd blockchain
npm install
cd ../../frontend
npm install
```

### 1. Start a Local Blockchain

```bash
cd backend/blockchain
npx hardhat node
```

Keep this terminal running. The local RPC URL is `http://127.0.0.1:8545/`.

### 2. Deploy the Contract Locally

In a second terminal:

```bash
cd backend/blockchain
npx hardhat ignition deploy ignition/modules/DeployRegistry.ts --network localhost
```

Copy the deployed contract address into `backend/.env` as `CONTRACT_ADDRESS`.

Hardhat commonly deploys the first local contract to:

```text
0x5FbDB2315678afecb367f032d93F642f64180aa3
```

Only rely on that default if your deployment output matches it.

### 3. Configure the Backend

```bash
cd backend
cp .env.example .env
```

Set the values in `backend/.env`:

```env
PORT=3001
CORS_ORIGIN=http://localhost:5173
RPC_URL=http://127.0.0.1:8545/
CONTRACT_ADDRESS=<local-deployed-contract-address>
PRIVATE_KEY=
PINATA_JWT_TOKEN=<your-pinata-jwt>
WATERMARK_STRICT=false
```

For local Hardhat, `PRIVATE_KEY` can be empty because the backend uses the first local signer. For a remote RPC, `PRIVATE_KEY` is required.

Start the backend:

```bash
npm run dev
```

The API runs at `http://localhost:3001`.

Useful health routes:

- `http://localhost:3001/api`
- `http://localhost:3001/api/health`

### 4. Configure and Run the Frontend

```bash
cd frontend
cp .env.example .env
npm run dev
```

`frontend/.env` should contain:

```env
VITE_API_BASE_URL=http://localhost:3001
```

The Vite app usually runs at `http://localhost:5173`.

## API Summary

### `POST /api/upload`

Multipart form upload with field name `file`.

Registers a file by:

1. Calculating SHA-256 from the original file.
2. Checking whether that hash already exists on-chain.
3. Watermarking images when possible.
4. Calculating image pHash when possible.
5. Pinning the final file to IPFS.
6. Creating an on-chain record.

Supported file categories include images, video, audio, PDF, CSV, TXT, Word, Excel, and PowerPoint files. Maximum upload size is 100 MB.

### `POST /api/verify`

Multipart form upload with field name `file`.

Calculates SHA-256 for the uploaded file and returns the matching on-chain record when found.

## Deployment Notes

### Frontend on Vercel

- Root directory: `frontend`
- Build command: `npm run build`
- Output directory: `dist`
- Environment variable:

```env
VITE_API_BASE_URL=https://<your-render-service>.onrender.com
```

### Backend on Render

- Root directory: `backend`
- Build command: `npm install`
- Start command: `npm start`

Required environment variables:

```env
PORT=5000
CORS_ORIGIN=https://<your-vercel-project>.vercel.app,https://*.vercel.app
RPC_URL=<your-chain-rpc-url>
CONTRACT_ADDRESS=<deployed-contract-address>
PRIVATE_KEY=<wallet-private-key-for-chain-writes>
PINATA_JWT_TOKEN=<pinata-jwt-token>
WATERMARK_STRICT=false
```

Instead of `PINATA_JWT_TOKEN`, you can set both `PINATA_API_KEY` and `PINATA_API_SECRET`.

### Deploying to Sepolia or Amoy

Create `backend/blockchain/.env` from `backend/blockchain/.env.example`, then set:

```env
RPC_URL=<sepolia-rpc-url>
SEPOLIA_RPC_URL=<optional-sepolia-rpc-url>
ALCHEMY_AMOY_URL=<optional-amoy-rpc-url>
PRIVATE_KEY=<deployer-private-key>
```

Deploy to Sepolia:

```bash
cd backend/blockchain
npm run deploy:sepolia
```

Or deploy to a configured Hardhat network directly:

```bash
npx hardhat run scripts/deploy.js --network sepolia
```

Copy the printed contract address into the backend environment as `CONTRACT_ADDRESS`.

## Important Behavior

- Verification is exact-match SHA-256 verification. If a file changes by even one byte, verification will not match the previous record.
- Image uploads are hashed before watermarking, so verification should use the original file that was registered.
- Watermarking and pHash are only used for images. Non-image files store `pHash` as `not_applicable`.
- Pinata credentials are required when the backend starts.
- Duplicate content cannot be registered twice under a different filename.

## Status

Implemented:

- React register and verify UI
- Express upload and verify API
- SHA-256 hashing
- Optional image watermarking with Jimp
- Image perceptual hash
- Pinata IPFS pinning
- Solidity registry contract
- Local Hardhat workflow
- Render/Vercel deployment configuration notes

Planned or future work:

- Stronger invisible watermarking
- More robust media similarity detection
- Video-specific provenance workflows
- Public testnet deployment hardening.
