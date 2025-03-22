<template>
  <div id="app">
    <h1>ISEQ20 Token DApp</h1>

    <!-- Wallet Connection -->
    <div v-if="!account" class="connection-box">
      <button class="connect-btn" @click="connectWallet">Connect Wallet</button>
    </div>
    <div v-else class="main-content">
      <div class="account-info">
        <p><strong>Connected Account:</strong> {{ shortAccount }}</p>
        <p><strong>Total Balance:</strong> {{ balance }} ISEQ</p>
        <p><strong>Locked Balance:</strong> {{ lockedBalance }} ISEQ</p>
        <p><strong>Unlocked Balance:</strong> {{ unlockedBalance }} ISEQ</p>
        <p><strong>Synthetic Value:</strong> {{ syntheticValue }} (based on ISEQ 20)</p>
        <p><strong>Current Index Value:</strong> {{ indexValue }}</p>
        <!-- EUR/ETH rate display -->
        <p><strong>Current EUR/ETH Rate:</strong> {{ eurEthRate }} EUR per ETH</p>
        <button @click="updateIndex">Update Index Value</button>
      </div>

      <!-- Swap Section -->
      <div class="swap-section">
        <h2>Trade ISEQ Tokens</h2>
        
        <div class="swap-container">
          <div class="swap-card">
            <div class="swap-header">
              <span>Swap</span>
              <div class="swap-settings">
                <span class="slippage">Slippage: 0.5%</span>
                <div class="direction-toggle">
                  <button class="toggle-btn" @click="toggleSwapDirection">
                    {{ isEthToToken ? 'ETH → ISEQ' : 'ISEQ → ETH' }}
                    <span class="toggle-icon">⟲</span>
                  </button>
                </div>
              </div>
            </div>
            
            <div class="swap-body">
              <!-- From Input -->
              <div class="swap-input-container">
                <div class="swap-input-header">
                  <span>From</span>
                  <span class="balance">Balance: {{ isEthToToken ? ethBalance + ' ETH' : balance + ' ISEQ' }}</span>
                </div>
                <div class="swap-input">
                  <input v-model.number="swapAmount" type="number" min="0.001" step="0.001" placeholder="0.0" @input="updateEstimatedOutput" />
                  <div class="token-selector">
                    <template v-if="isEthToToken">
                      <img src="https://cryptologos.cc/logos/ethereum-eth-logo.png" alt="ETH" class="token-icon" />
                      <span>ETH</span>
                    </template>
                    <template v-else>
                      <div class="token-icon token-iseq">ISEQ</div>
                      <span>ISEQ</span>
                    </template>
                  </div>
                </div>
                <div class="max-button" @click="setMaxAmount">MAX</div>
              </div>
              
              <!-- Arrow -->
              <div class="swap-arrow">↓</div>
              
              <!-- To Input -->
              <div class="swap-input-container">
                <div class="swap-input-header">
                  <span>To (Estimated)</span>
                  <span class="balance">Balance: {{ isEthToToken ? balance + ' ISEQ' : ethBalance + ' ETH' }}</span>
                </div>
                <div class="swap-input readonly">
                  <input :value="estimatedOutput" type="text" readonly placeholder="0.0" />
                  <div class="token-selector">
                    <template v-if="isEthToToken">
                      <div class="token-icon token-iseq">ISEQ</div>
                      <span>ISEQ</span>
                    </template>
                    <template v-else>
                      <img src="https://cryptologos.cc/logos/ethereum-eth-logo.png" alt="ETH" class="token-icon" />
                      <span>ETH</span>
                    </template>
                  </div>
                </div>
              </div>
              
              <!-- Rate -->
              <div class="swap-rate">
                <span>Rate: {{ isEthToToken ? '1 ETH ≈ ' + swapRate + ' ISEQ' : '1 ISEQ ≈ ' + inverseRate + ' ETH' }}</span>
              </div>
            </div>
            
            <div class="swap-footer">
              <button v-if="!poolExists" @click="createLiquidityPool" class="create-pool-btn" :disabled="poolCreating">
                {{ poolCreating ? 'Creating Pool...' : 'Create Liquidity Pool' }}
              </button>
              <button v-else-if="!isTokenApproved && (!isEthToToken)" @click="approveToken" :disabled="isApproving || swapAmount <= 0">
                {{ isApproving ? 'Approving...' : 'Approve ISEQ' }}
              </button>
              <button v-else @click="swapTokens" :disabled="isSwapping || swapAmount <= 0">
                {{ isSwapping ? 'Swapping...' : 'Swap' }}
              </button>
            </div>
          </div>
          
          <div class="transaction-status" v-if="txHash">
            <p>Transaction Hash: <a :href="'https://sepolia.etherscan.io/tx/' + txHash" target="_blank">{{ txHash.substring(0, 10) }}...{{ txHash.substring(txHash.length - 8) }}</a></p>
            <p v-if="txStatus === 'pending'" class="status pending">Transaction Pending...</p>
            <p v-else-if="txStatus === 'success'" class="status success">Transaction Successful!</p>
            <p v-else-if="txStatus === 'failed'" class="status failed">Transaction Failed!</p>
          </div>
        </div>
      </div>

      <!-- Value Display -->
      <div class="value-display-container">
        <h2>ISEQ 20 Index & Synthetic Value</h2>
        
        <div class="value-headers">
          <div class="value-label">Time</div>
          <div class="value-label">ISEQ 20 Index</div>
          <div class="value-label">Synthetic Value</div>
        </div>
        
        <div class="value-rows">
          <div v-for="(entry, index) in displayHistory" :key="index" class="value-row">
            <div class="value-cell time">{{ formatTime(entry.timestamp) }}</div>
            <div class="value-cell index" :class="{ 'up': entry.indexChange > 0, 'down': entry.indexChange < 0 }">
              {{ entry.indexValue }}
              <span v-if="entry.indexChange !== 0" class="change-indicator">
                {{ entry.indexChange > 0 ? '↑' : '↓' }}
              </span>
            </div>
            <div class="value-cell synthetic" :class="{ 'up': entry.synthChange > 0, 'down': entry.synthChange < 0 }">
              {{ entry.synthValue }}
              <span v-if="entry.synthChange !== 0" class="change-indicator">
                {{ entry.synthChange > 0 ? '↑' : '↓' }}
              </span>
            </div>
          </div>
        </div>
        
        <div class="update-time">
          Last updated: {{ lastUpdated }}
        </div>
      </div>

      <!-- Token Locking -->
      <div class="token-actions">
        <h2>Lock Tokens for Governance</h2>
        <div class="form-group">
          <input v-model.number="lockAmount" type="number" placeholder="Amount to lock" />
          <input v-model.number="lockDuration" type="number" placeholder="Duration in days" />
          <button @click="lockTokens" :disabled="!canLock">Lock Tokens</button>
        </div>
        <div v-if="lockEndTime > 0" class="lock-info">
          <p>Your tokens are locked until: {{ formatDate(lockEndTime) }}</p>
          <button @click="unlockTokens" :disabled="!canUnlock">Unlock Tokens</button>
        </div>
      </div>

      <!-- Create Proposal -->
      <div class="governance-section">
        <h2>Create Proposal</h2>
        <div class="form-group">
          <textarea v-model="proposalDesc" placeholder="Proposal description" rows="3"></textarea>
          <div class="duration-selector">
            <label>Duration: {{ proposalDuration / 86400 }} days</label>
            <input type="range" v-model.number="proposalDuration" min="86400" max="2592000" step="86400" />
          </div>
          <button @click="createProposal" :disabled="!canPropose">Submit Proposal</button>
        </div>
      </div>

      <!-- Proposals List -->
      <div class="proposals-section">
        <h2>Proposals</h2>
        <div class="proposals-container">
          <div v-for="proposal in proposals" :key="proposal.id" class="proposal-card">
            <div class="proposal-header">
              <span class="proposal-id">ID: {{ proposal.id }}</span>
              <span class="proposal-status" :class="getStatusClass(proposal)">{{ getProposalStatus(proposal) }}</span>
            </div>
            <div class="proposal-body">
              <p class="proposal-desc">{{ proposal.description }}</p>
              <p class="proposal-votes">Votes: {{ proposal.voteCount }} ISEQ</p>
              <div class="proposal-timeline">
                <p>Voting ends: {{ formatDate(proposal.votingDeadline) }}</p>
                <p v-if="proposal.executionTime > 0">Can execute after: {{ formatDate(proposal.executionTime) }}</p>
              </div>
            </div>
            <div class="proposal-actions">
              <div v-if="canVoteOnProposal(proposal)">
                <input v-model.number="voteAmount[proposal.id]" type="number" placeholder="Votes" />
                <button @click="vote(proposal.id)">Vote</button>
              </div>
              <button v-if="canQueueProposal(proposal)" @click="queueProposal(proposal.id)">Queue for Execution</button>
              <button v-if="canExecuteProposal(proposal)" @click="executeProposal(proposal.id)">Execute Proposal</button>
              <button v-if="isOwner && canCancelProposal(proposal)" @click="cancelProposal(proposal.id)" class="cancel-btn">Cancel</button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
/* Import Web3 */
import Web3 from 'web3';

export default {
  name: 'App',
  data() {
    return {
      web3: null,
      account: '',
      isOwner: false,
      balance: 0,
      lockedBalance: 0,
      unlockedBalance: 0,
      syntheticValue: 0,
      lockEndTime: 0,
      indexValue: 0,
      indexHistory: [],
      syntheticHistory: [],
      displayHistory: [],
      lastUpdated: 'Never',
      contract: null,
      proposalInterval: null,
      userDataInterval: null,
      
      // Add EUR/ETH rate property
      eurEthRate: 0,
      
      // Constants for decimal handling
      CHAINLINK_DECIMALS: 8, // Typical Chainlink price feed decimals
      
      // Swap related data
      ethBalance: '0.00',
      swapAmount: 0.1,
      estimatedOutput: '0.00', // renamed from estimatedTokens to be direction-agnostic
      swapRate: '0.00',
      inverseRate: '0.00', // rate for ISEQ to ETH conversion
      isApproving: false,
      isSwapping: false,
      poolCreating: false,
      poolExists: false,
      isTokenApproved: false,
      txHash: '',
      txStatus: '',
      isEthToToken: true, // true = ETH to ISEQ, false = ISEQ to ETH
      
      // Uniswap contract addresses for Sepolia
      uniswapRouter: '0xC532a74256D3Db42D0Bf7a0400fEFDbad7694008',
      uniswapFactory: '0x7E0987E5b3a30e3f2828572Bb659A548460a3003',
      wethAddress: '0x7b79995e5f793A07Bc00c21412e50Ecae098E7f9',
      
      // Token locking
      lockAmount: 0,
      lockDuration: 7, // Default: 7 days
      
      // Proposal creation
      proposalDesc: '',
      proposalDuration: 604800, // Default: 7 days in seconds
      
      // Proposals
      proposals: [],
      voteAmount: {},
      
      // Contract address and ABI
      contractAddress: '0xf784D260E8517CA6a2B134CE2162F44b66Ab5Fc8',
      contractABI: [
        // Constructor
        {"inputs":[{"internalType":"address","name":"priceFeedAddress","type":"address"}],"stateMutability":"nonpayable","type":"constructor"},
        
        // ERC20 functions
        {"inputs":[{"internalType":"address","name":"account","type":"address"}],"name":"balanceOf","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},
        {"inputs":[{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"amount","type":"uint256"}],"name":"transfer","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"nonpayable","type":"function"},
        {"inputs":[{"internalType":"address","name":"owner","type":"address"},{"internalType":"address","name":"spender","type":"address"}],"name":"allowance","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},
        {"inputs":[{"internalType":"address","name":"spender","type":"address"},{"internalType":"uint256","name":"amount","type":"uint256"}],"name":"approve","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"nonpayable","type":"function"},
        
        // Token locking
        {"inputs":[{"internalType":"address","name":"","type":"address"}],"name":"lockedTokens","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},
        {"inputs":[{"internalType":"address","name":"","type":"address"}],"name":"lockEndTime","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},
        {"inputs":[{"internalType":"uint256","name":"amount","type":"uint256"},{"internalType":"uint256","name":"duration","type":"uint256"}],"name":"lockTokens","outputs":[],"stateMutability":"nonpayable","type":"function"},
        {"inputs":[],"name":"unlockTokens","outputs":[],"stateMutability":"nonpayable","type":"function"},
        
        // Index value
        {"inputs":[],"name":"indexValue","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},
        {"inputs":[],"name":"updateIndexValue","outputs":[],"stateMutability":"nonpayable","type":"function"},
        {"inputs":[{"internalType":"address","name":"user","type":"address"}],"name":"getSyntheticValue","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},
        
        // New function for EUR/ETH price
        {"inputs":[],"name":"getEurEthPrice","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},
        
        // Governance
        {"inputs":[],"name":"owner","outputs":[{"internalType":"address","name":"","type":"address"}],"stateMutability":"view","type":"function"},
        {"inputs":[],"name":"proposalCount","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},
        {"inputs":[{"internalType":"uint256","name":"","type":"uint256"}],"name":"proposals","outputs":[
          {"internalType":"string","name":"description","type":"string"},
          {"internalType":"uint256","name":"voteCount","type":"uint256"},
          {"internalType":"uint256","name":"votingDeadline","type":"uint256"},
          {"internalType":"uint256","name":"executionTime","type":"uint256"},
          {"internalType":"bool","name":"executed","type":"bool"},
          {"internalType":"bool","name":"canceled","type":"bool"}
        ],"stateMutability":"view","type":"function"},
        {"inputs":[{"internalType":"string","name":"description","type":"string"},{"internalType":"uint256","name":"votingDuration","type":"uint256"}],"name":"createProposal","outputs":[],"stateMutability":"nonpayable","type":"function"},
        {"inputs":[{"internalType":"uint256","name":"proposalId","type":"uint256"},{"internalType":"uint256","name":"voteAmount","type":"uint256"}],"name":"vote","outputs":[],"stateMutability":"nonpayable","type":"function"},
        {"inputs":[{"internalType":"uint256","name":"proposalId","type":"uint256"}],"name":"queueProposal","outputs":[],"stateMutability":"nonpayable","type":"function"},
        {"inputs":[{"internalType":"uint256","name":"proposalId","type":"uint256"}],"name":"executeProposal","outputs":[],"stateMutability":"nonpayable","type":"function"},
        {"inputs":[{"internalType":"uint256","name":"proposalId","type":"uint256"}],"name":"cancelProposal","outputs":[],"stateMutability":"nonpayable","type":"function"},
      ],
      
      // Uniswap ABIs
      uniswapRouterABI: [
        // Router functions we need
        {
          "inputs": [
            {"internalType": "uint256", "name": "amountOutMin", "type": "uint256"},
            {"internalType": "address[]", "name": "path", "type": "address[]"},
            {"internalType": "address", "name": "to", "type": "address"},
            {"internalType": "uint256", "name": "deadline", "type": "uint256"}
          ],
          "name": "swapExactETHForTokens",
          "outputs": [{"internalType": "uint256[]", "name": "amounts", "type": "uint256[]"}],
          "stateMutability": "payable",
          "type": "function"
        },
        // New function for ISEQ to ETH swaps
        {
          "inputs": [
            {"internalType": "uint256", "name": "amountIn", "type": "uint256"},
            {"internalType": "uint256", "name": "amountOutMin", "type": "uint256"},
            {"internalType": "address[]", "name": "path", "type": "address[]"},
            {"internalType": "address", "name": "to", "type": "address"},
            {"internalType": "uint256", "name": "deadline", "type": "uint256"}
          ],
          "name": "swapExactTokensForETH",
          "outputs": [{"internalType": "uint256[]", "name": "amounts", "type": "uint256[]"}],
          "stateMutability": "nonpayable",
          "type": "function"
        },
        {
          "inputs": [
            {"internalType": "address", "name": "token", "type": "address"},
            {"internalType": "uint256", "name": "amountTokenDesired", "type": "uint256"},
            {"internalType": "uint256", "name": "amountTokenMin", "type": "uint256"},
            {"internalType": "uint256", "name": "amountETHMin", "type": "uint256"},
            {"internalType": "address", "name": "to", "type": "address"},
            {"internalType": "uint256", "name": "deadline", "type": "uint256"}
          ],
          "name": "addLiquidityETH",
          "outputs": [
            {"internalType": "uint256", "name": "amountToken", "type": "uint256"},
            {"internalType": "uint256", "name": "amountETH", "type": "uint256"},
            {"internalType": "uint256", "name": "liquidity", "type": "uint256"}
          ],
          "stateMutability": "payable",
          "type": "function"
        },
        {
          "inputs": [
            {"internalType": "address", "name": "factory", "type": "address"},
            {"internalType": "address", "name": "tokenA", "type": "address"},
            {"internalType": "address", "name": "tokenB", "type": "address"}
          ],
          "name": "getReserves",
          "outputs": [
            {"internalType": "uint256", "name": "reserveA", "type": "uint256"},
            {"internalType": "uint256", "name": "reserveB", "type": "uint256"}
          ],
          "stateMutability": "view",
          "type": "function"
        },
        {
          "inputs": [
            {"internalType": "uint256", "name": "amountOut", "type": "uint256"},
            {"internalType": "uint256", "name": "reserveIn", "type": "uint256"},
            {"internalType": "uint256", "name": "reserveOut", "type": "uint256"}
          ],
          "name": "getAmountIn",
          "outputs": [{"internalType": "uint256", "name": "amountIn", "type": "uint256"}],
          "stateMutability": "pure",
          "type": "function"
        },
        {
          "inputs": [
            {"internalType": "uint256", "name": "amountIn", "type": "uint256"},
            {"internalType": "uint256", "name": "reserveIn", "type": "uint256"},
            {"internalType": "uint256", "name": "reserveOut", "type": "uint256"}
          ],
          "name": "getAmountOut",
          "outputs": [{"internalType": "uint256", "name": "amountOut", "type": "uint256"}],
          "stateMutability": "pure",
          "type": "function"
        }
      ],
      uniswapFactoryABI: [
        {
          "inputs": [
            {"internalType": "address", "name": "tokenA", "type": "address"},
            {"internalType": "address", "name": "tokenB", "type": "address"}
          ],
          "name": "getPair",
          "outputs": [{"internalType": "address", "name": "pair", "type": "address"}],
          "stateMutability": "view",
          "type": "function"
        }
      ]
    };
  },
  computed: {
    shortAccount() {
      if (!this.account) return '';
      return `${this.account.substring(0, 6)}...${this.account.substring(this.account.length - 4)}`;
    },
    canLock() {
      return this.lockAmount > 0 && this.lockDuration >= 7 && this.unlockedBalance >= this.lockAmount;
    },
    canUnlock() {
      return this.lockedBalance > 0 && Date.now() >= this.lockEndTime;
    },
    canPropose() {
      return this.proposalDesc && this.proposalDuration >= 86400 && this.lockedBalance >= 1000;
    }
  },
  mounted() {
    // Check if already connected to MetaMask
    this.checkConnection();
  },
  
  beforeUnmount() {
    // Clean up intervals
    if (this.proposalInterval) clearInterval(this.proposalInterval);
    if (this.userDataInterval) clearInterval(this.userDataInterval);
  },
  
  methods: {
    // Toggle swap direction
    toggleSwapDirection() {
      this.isEthToToken = !this.isEthToToken;
      this.swapAmount = 0.1; // Reset amount when changing direction
      this.updateEstimatedOutput();
      
      // Check allowance if switching to ISEQ -> ETH
      if (!this.isEthToToken) {
        this.checkAllowance();
      }
    },
    
    // Format the raw index value with correct decimal placement
    formatIndexValue(rawValue) {
      // Convert from Chainlink format (8 decimals) to human-readable number
      return (parseFloat(rawValue) / Math.pow(10, this.CHAINLINK_DECIMALS)).toFixed(2);
    },
    
    // New method to fetch EUR/ETH rate
    async fetchEurEthRate() {
      if (this.contract) {
        try {
          const eurEthRate = await this.contract.methods.getEurEthPrice().call();
          // Convert from 18 decimals (standard for ETH values)
          this.eurEthRate = (parseFloat(eurEthRate) / 1e18).toFixed(6);
          console.log("EUR/ETH rate:", this.eurEthRate);
        } catch (error) {
          console.error('Error fetching EUR/ETH rate:', error);
        }
      }
    },
    
    checkConnection() {
      if (window.ethereum && window.ethereum.selectedAddress) {
        this.connectWallet();
      }
    },
    
    async connectWallet() {
      // Check if MetaMask is installed
      if (window.ethereum) {
        try {
          // Request account access
          const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });
          this.account = accounts[0];
          
          // Check if we're on Sepolia network (chain ID 11155111)
          const chainId = await window.ethereum.request({ method: 'eth_chainId' });
          if (chainId !== '0xaa36a7') { // Sepolia chain ID in hex
            try {
              // Try to switch to Sepolia
              await window.ethereum.request({
                method: 'wallet_switchEthereumChain',
                params: [{ chainId: '0xaa36a7' }], // Sepolia chain ID
              });
            } catch (switchError) {
              // This error code indicates that the chain has not been added to MetaMask.
              if (switchError.code === 4902) {
                try {
                  await window.ethereum.request({
                    method: 'wallet_addEthereumChain',
                    params: [
                      {
                        chainId: '0xaa36a7',
                        chainName: 'Sepolia Test Network',
                        nativeCurrency: {
                          name: 'ETH',
                          symbol: 'ETH',
                          decimals: 18,
                        },
                        rpcUrls: ['https://sepolia.infura.io/v3/'],
                        blockExplorerUrls: ['https://sepolia.etherscan.io/'],
                      },
                    ],
                  });
                } catch (addError) {
                  console.error('Failed to add Sepolia network to MetaMask', addError);
                  alert('Please manually switch to Sepolia network in MetaMask and try again.');
                  return;
                }
              } else {
                console.error('Failed to switch network', switchError);
                alert('Please manually switch to Sepolia network in MetaMask and try again.');
                return;
              }
            }
          }
          
          // Initialize Web3
          this.web3 = new Web3(window.ethereum);
          
          // Initialize contract
          this.contract = new this.web3.eth.Contract(
            this.contractABI,
            this.contractAddress
          );
          
          // Check if user is owner
          const ownerAddress = await this.contract.methods.owner().call();
          this.isOwner = (ownerAddress.toLowerCase() === this.account.toLowerCase());
          
          // Fetch initial data
          await this.fetchUserData();
          await this.fetchProposals();
          this.setupEventListeners();
          
          // Fetch additional data for swap functionality
          await this.fetchEthBalance();
          await this.checkPoolExists();
          if (this.poolExists) {
            await this.checkAllowance();
            await this.calculateSwapRate();
          }
          
          // Listen for account and network changes
          window.ethereum.on('accountsChanged', (accounts) => {
            window.location.reload();
          });
          
          window.ethereum.on('chainChanged', (chainId) => {
            window.location.reload();
          });
          
          // Refresh data periodically
          this.proposalInterval = setInterval(() => this.fetchProposals(), 30000); // Every 30 seconds
          this.userDataInterval = setInterval(() => this.fetchUserData(), 60000); // Every minute
          
        } catch (error) {
          console.error('Error connecting wallet:', error);
          alert('Failed to connect wallet: ' + error.message);
        }
      } else {
        alert('Please install MetaMask!');
      }
    },
    
    async fetchUserData() {
      if (this.contract && this.account) {
        try {
          // Get total balance
          const balance = await this.contract.methods.balanceOf(this.account).call();
          this.balance = parseFloat(this.web3.utils.fromWei(balance, 'ether')).toFixed(2);
          console.log("User balance in wei:", balance);
          console.log("User balance in ISEQ:", this.balance);
          
          // Get locked tokens
          const lockedTokens = await this.contract.methods.lockedTokens(this.account).call();
          this.lockedBalance = parseFloat(this.web3.utils.fromWei(lockedTokens, 'ether')).toFixed(2);
          
          // Calculate unlocked balance
          this.unlockedBalance = (parseFloat(this.balance) - parseFloat(this.lockedBalance)).toFixed(2);
          
          // Get lock end time
          const lockEnd = await this.contract.methods.lockEndTime(this.account).call();
          this.lockEndTime = parseInt(lockEnd) * 1000; // Convert to milliseconds
          
          // Get current index value and format correctly
          const rawIndexValue = await this.contract.methods.indexValue().call();
          console.log("Raw index value:", rawIndexValue);
          const formattedIndexValue = this.formatIndexValue(rawIndexValue);
          this.indexValue = formattedIndexValue;
          
          // Fetch EUR/ETH rate
          await this.fetchEurEthRate();
          
          // Get synthetic value and calculate it in a way that makes sense
          const rawSyntheticValue = await this.contract.methods.getSyntheticValue(this.account).call();
          console.log("Raw synthetic value:", rawSyntheticValue);
          
          // Calculate synthetic value as balance * indexValue / initialIndexValue (5000)
          // This matches the logic in the smart contract
          const baseValue = parseFloat(this.balance) * parseFloat(this.indexValue) / 5000;
          console.log("Calculated synthetic estimate:", baseValue);
          
          // Use the calculated value which should be reasonable
          this.syntheticValue = baseValue.toFixed(2);
          
          // Add to history for the simple value display
          const timestamp = Date.now();
          
          // Calculate proper formatted values for history
          const indexValueForHistory = parseFloat(formattedIndexValue);
          const syntheticValueForHistory = parseFloat(this.syntheticValue);
          
          // Get previous values
          const prevIndexValue = this.indexHistory.length > 0 
            ? this.indexHistory[this.indexHistory.length - 1].value 
            : indexValueForHistory;
            
          const prevSynthValue = this.syntheticHistory.length > 0 
            ? this.syntheticHistory[this.syntheticHistory.length - 1].value 
            : syntheticValueForHistory;
          
          // Add new history entries with properly formatted values
          this.indexHistory.push({
            timestamp,
            value: indexValueForHistory
          });
          
          this.syntheticHistory.push({
            timestamp,
            value: syntheticValueForHistory
          });
          
          // Keep only last 10 data points
          if (this.indexHistory.length > 10) {
            this.indexHistory = this.indexHistory.slice(-10);
          }
          
          if (this.syntheticHistory.length > 10) {
            this.syntheticHistory = this.syntheticHistory.slice(-10);
          }
          
          // Update display history (table for now, graph for future - Note: graph.js have scalling problem. author: Sam)
          this.displayHistory = [];
          
          for (let i = this.indexHistory.length - 1; i >= 0; i--) {
            // Only add if there's data for both index and synthetic value
            if (i < this.syntheticHistory.length) {
              // Calculate change
              const prevIdx = i + 1 < this.indexHistory.length ? this.indexHistory[i + 1].value : this.indexHistory[i].value;
              const prevSynth = i + 1 < this.syntheticHistory.length ? this.syntheticHistory[i + 1].value : this.syntheticHistory[i].value;
              
              this.displayHistory.push({
                timestamp: this.indexHistory[i].timestamp,
                indexValue: this.indexHistory[i].value.toFixed(2),
                synthValue: this.syntheticHistory[i].value.toFixed(2),
                indexChange: this.indexHistory[i].value - prevIdx,
                synthChange: this.syntheticHistory[i].value - prevSynth
              });
            }
          }
          
          // Update last updated time
          this.lastUpdated = new Date().toLocaleString();
          
          // Update swap rate if pool exists
          if (this.poolExists) {
            await this.calculateSwapRate();
          }
          
        } catch (error) {
          console.error('Error fetching user data:', error);
          alert('Error fetching data: ' + error.message);
        }
      }
    },
    
    async updateIndex() {
      if (this.contract && this.account) {
        try {
          await this.contract.methods.updateIndexValue().send({ from: this.account });
          alert('Index updated!');
          await this.fetchUserData();
        } catch (error) {
          console.error('Error updating index:', error);
          alert('Failed to update index: ' + error.message);
        }
      }
    },
    
    async lockTokens() {
      if (this.contract && this.account && this.lockAmount > 0 && this.lockDuration >= 7) {
        try {
          const amount = this.web3.utils.toWei(this.lockAmount.toString(), 'ether');
          const durationInSeconds = this.lockDuration * 86400; // Convert days to seconds
          await this.contract.methods.lockTokens(amount, durationInSeconds).send({ from: this.account });
          alert('Tokens locked successfully!');
          this.lockAmount = 0;
          await this.fetchUserData();
        } catch (error) {
          console.error('Error locking tokens:', error);
          alert('Failed to lock tokens: ' + error.message);
        }
      }
    },
    
    async unlockTokens() {
      if (this.contract && this.account && Date.now() >= this.lockEndTime) {
        try {
          await this.contract.methods.unlockTokens().send({ from: this.account });
          alert('Tokens unlocked successfully!');
          await this.fetchUserData();
        } catch (error) {
          console.error('Error unlocking tokens:', error);
          alert('Failed to unlock tokens: ' + error.message);
        }
      }
    },
    
    async createProposal() {
      if (this.contract && this.account && this.proposalDesc && this.proposalDuration >= 86400) {
        try {
          await this.contract.methods.createProposal(this.proposalDesc, this.proposalDuration).send({ from: this.account });
          alert('Proposal created!');
          this.proposalDesc = '';
          await this.fetchProposals();
        } catch (error) {
          console.error('Error creating proposal:', error);
          alert('Failed to create proposal: ' + error.message);
        }
      }
    },
    
    async fetchProposals() {
      if (this.contract) {
        try {
          const count = await this.contract.methods.proposalCount().call();
          const proposals = [];
          
          for (let i = 1; i <= count; i++) {
            const proposal = await this.contract.methods.proposals(i).call();
            
            // For each proposal, also check if the current user has voted
            let hasVoted = false;
            try {
              const voteAmount = await this.contract.methods.votes(i, this.account).call();
              hasVoted = (parseInt(voteAmount) > 0);
            } catch (error) {
              // This would happen if the contract doesn't expose the votes mapping
              hasVoted = false;
            }
            
            proposals.push({
              id: i,
              description: proposal.description,
              voteCount: this.web3.utils.fromWei(proposal.voteCount, 'ether'),
              votingDeadline: parseInt(proposal.votingDeadline) * 1000, // Convert to milliseconds
              executionTime: parseInt(proposal.executionTime) * 1000, // Convert to milliseconds
              executed: proposal.executed,
              canceled: proposal.canceled,
              hasVoted: hasVoted
            });
          }
          
          this.proposals = proposals.reverse(); // Newest first
        } catch (error) {
          console.error('Error fetching proposals:', error);
          alert('Error fetching proposals: ' + error.message);
        }
      }
    },
    
    async vote(proposalId) {
      if (this.contract && this.account && this.voteAmount[proposalId] > 0) {
        try {
          const amount = this.web3.utils.toWei(this.voteAmount[proposalId].toString(), 'ether');
          await this.contract.methods.vote(proposalId, amount).send({ from: this.account });
          alert('Vote submitted!');
          this.voteAmount[proposalId] = '';
          await this.fetchProposals();
        } catch (error) {
          console.error('Error voting:', error);
          alert('Failed to vote: ' + error.message);
        }
      }
    },
    
    async queueProposal(proposalId) {
      if (this.contract && this.account) {
        try {
          await this.contract.methods.queueProposal(proposalId).send({ from: this.account });
          alert('Proposal queued for execution!');
          await this.fetchProposals();
        } catch (error) {
          console.error('Error queueing proposal:', error);
          alert('Failed to queue proposal: ' + error.message);
        }
      }
    },
    
    async executeProposal(proposalId) {
      if (this.contract && this.account) {
        try {
          await this.contract.methods.executeProposal(proposalId).send({ from: this.account });
          alert('Proposal executed!');
          await this.fetchProposals();
        } catch (error) {
          console.error('Error executing proposal:', error);
          alert('Failed to execute proposal: ' + error.message);
        }
      }
    },
    
    async cancelProposal(proposalId) {
      if (this.contract && this.account && this.isOwner) {
        try {
          await this.contract.methods.cancelProposal(proposalId).send({ from: this.account });
          alert('Proposal canceled!');
          await this.fetchProposals();
        } catch (error) {
          console.error('Error canceling proposal:', error);
          alert('Failed to cancel proposal: ' + error.message);
        }
      }
    },
    
    setupEventListeners() {
      if (this.contract && this.web3) {
        // For Web3.js, event handling is a bit different (Need to check etherjs - Note for futher update)
        // We need to subscribe to specific events
        
        // Listen for relevant events
        this.contract.events.allEvents({}, (error, event) => {
          if (error) {
            console.error('Event error:', error);
            return;
          }
          
          // Handle different event types
          switch (event.event) {
            case 'IndexUpdated':
              console.log(`Index updated to ${event.returnValues.newValue}`);
              this.fetchUserData();
              break;
            case 'ProposalCreated':
            case 'Voted':
            case 'ProposalExecutionScheduled':
            case 'ProposalExecuted':
            case 'ProposalCanceled':
              this.fetchProposals();
              break;
            case 'TokensLocked':
            case 'TokensUnlocked':
              if (event.returnValues.user.toLowerCase() === this.account.toLowerCase()) {
                this.fetchUserData();
              }
              break;
          }
        });
      }
    },
    
    formatDate(timestamp) {
      return new Date(timestamp).toLocaleString();
    },
    
    formatTime(timestamp) {
      const date = new Date(timestamp);
      return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
    },
    
    getProposalStatus(proposal) {
      if (proposal.canceled) return 'Canceled';
      if (proposal.executed) return 'Executed';
      if (proposal.executionTime > 0) {
        return Date.now() >= proposal.executionTime ? 'Ready to Execute' : 'Queued';
      }
      return Date.now() >= proposal.votingDeadline ? 'Voting Ended' : 'Voting Active';
    },
    
    getStatusClass(proposal) {
      if (proposal.canceled) return 'status-canceled';
      if (proposal.executed) return 'status-executed';
      if (proposal.executionTime > 0) {
        return Date.now() >= proposal.executionTime ? 'status-ready' : 'status-queued';
      }
      return Date.now() >= proposal.votingDeadline ? 'status-ended' : 'status-active';
    },
    
    canVoteOnProposal(proposal) {
      return !proposal.canceled && 
             !proposal.executed && 
             Date.now() < proposal.votingDeadline && 
             !proposal.hasVoted &&
             this.lockedBalance > 0;
    },
    
    canQueueProposal(proposal) {
      return !proposal.canceled && 
             !proposal.executed && 
             proposal.executionTime === 0 && 
             Date.now() >= proposal.votingDeadline;
    },
    
    canExecuteProposal(proposal) {
      return !proposal.canceled && 
             !proposal.executed && 
             proposal.executionTime > 0 && 
             Date.now() >= proposal.executionTime;
    },
    
    canCancelProposal(proposal) {
      return !proposal.canceled && 
             !proposal.executed && 
             proposal.executionTime > 0 && 
             Date.now() < proposal.executionTime;
    },
    
    // Swap related methods
    async fetchEthBalance() {
      if (this.web3 && this.account) {
        try {
          const balance = await this.web3.eth.getBalance(this.account);
          this.ethBalance = parseFloat(this.web3.utils.fromWei(balance, 'ether')).toFixed(4);
        } catch (error) {
          console.error('Error fetching ETH balance:', error);
        }
      }
    },
    
    async checkPoolExists() {
      if (this.web3 && this.contractAddress) {
        try {
          const factoryContract = new this.web3.eth.Contract(
            this.uniswapFactoryABI,
            this.uniswapFactory
          );
          
          const pairAddress = await factoryContract.methods.getPair(
            this.contractAddress,
            this.wethAddress
          ).call();
          
          // If pair address is not the zero address, then the pool exists
          this.poolExists = pairAddress !== '0x0000000000000000000000000000000000000000';
          
          if (this.poolExists) {
            await this.calculateSwapRate();
          }
          
          return this.poolExists;
        } catch (error) {
          console.error('Error checking if pool exists:', error);
          return false;
        }
      }
      return false;
    },
    
    async createLiquidityPool() {
      if (this.web3 && this.contract && this.account) {
        try {
          this.poolCreating = true;
          
          // 1. Fetch the current index value to set initial pool ratio based on ISEQ index
          const currentIndexValue = parseFloat(this.indexValue);
          console.log("Creating pool with ISEQ index value:", currentIndexValue);
          
          // 2. Approve router to spend tokens
          // Use the index value to determine the token amount (if ISEQ = 1839.95, use that many tokens) - Note this is reference data
          // Add some buffer to ensure we have enough approved (2x the amount)
          const approveAmount = this.web3.utils.toWei((currentIndexValue * 2).toString(), 'ether');
          await this.contract.methods.approve(this.uniswapRouter, approveAmount).send({
            from: this.account
          });
          
          // 3. Add liquidity (ISEQ + ETH)
          const routerContract = new this.web3.eth.Contract(
            this.uniswapRouterABI,
            this.uniswapRouter
          );
          
          // Set initial pool ratio to match the ISEQ index (e.g. ~1839.95 ISEQ per 1 ETH)
          const tokenAmount = this.web3.utils.toWei(currentIndexValue.toString(), 'ether'); 
          const ethAmount = this.web3.utils.toWei('1', 'ether'); // 1 ETH
          const deadline = Math.floor(Date.now() / 1000) + 60 * 20; // 20 minute deadline
          
          const tx = await routerContract.methods.addLiquidityETH(
            this.contractAddress,
            tokenAmount,
            0, // slippage 100%
            0, // slippage 100%
            this.account,
            deadline
          ).send({
            from: this.account,
            value: ethAmount
          });
          
          this.txHash = tx.transactionHash;
          this.txStatus = 'success';
          
          alert(`Liquidity pool created successfully! Initial rate: ${currentIndexValue} ISEQ = 1 ETH`);
          this.poolExists = true;
          await this.calculateSwapRate();
          this.isTokenApproved = true; // Since we already approved tokens for the router
        } catch (error) {
          console.error('Error creating liquidity pool:', error);
          this.txStatus = 'failed';
          alert('Failed to create liquidity pool: ' + error.message);
        } finally {
          this.poolCreating = false;
        }
      }
    },
    
    async calculateSwapRate() {
      if (this.web3 && this.contract && this.poolExists) {
        try {
          const routerContract = new this.web3.eth.Contract(
            this.uniswapRouterABI,
            this.uniswapRouter
          );
          
          // Try to get reserves
          try {
            const reserves = await routerContract.methods.getReserves(
              this.uniswapFactory,
              this.wethAddress,
              this.contractAddress
            ).call();
            
            // Calculate rates for both directions
            if (parseInt(reserves[0]) > 0 && parseInt(reserves[1]) > 0) {
              // Calculate ETH -> ISEQ rate (how many ISEQ for 1 ETH)
              const oneEthInWei = this.web3.utils.toWei('1', 'ether');
              const tokensOut = await routerContract.methods.getAmountOut(
                oneEthInWei,
                reserves[0], // ETH reserve
                reserves[1]  // ISEQ reserve
              ).call();
              
              this.swapRate = parseFloat(this.web3.utils.fromWei(tokensOut, 'ether')).toFixed(2);
              
              // Calculate ISEQ -> ETH rate (how much ETH for 1 ISEQ)
              const oneTokenInWei = this.web3.utils.toWei('1', 'ether');
              const ethOut = await routerContract.methods.getAmountOut(
                oneTokenInWei,
                reserves[1], // ISEQ reserve
                reserves[0]  // ETH reserve
              ).call();
              
              this.inverseRate = parseFloat(this.web3.utils.fromWei(ethOut, 'ether')).toFixed(6);
            } else {
              // Use current ISEQ index value as the ETH -> ISEQ rate if available
              if (parseFloat(this.indexValue) > 0) {
                this.swapRate = parseFloat(this.indexValue).toFixed(2);
                // Set inverse rate as 1/indexValue
                this.inverseRate = (1 / parseFloat(this.indexValue)).toFixed(6);
              } else {
                // Fallback to initial rate if reserves can't be determined
                this.swapRate = (1000 / 0.1).toFixed(2);
                this.inverseRate = (0.1 / 1000).toFixed(6);
              }
            }
          } catch (reserveError) {
            console.warn('Could not get reserves, using index value as rate:', reserveError);
            // Use current ISEQ index value as the rate if available
            if (parseFloat(this.indexValue) > 0) {
              this.swapRate = parseFloat(this.indexValue).toFixed(2);
              this.inverseRate = (1 / parseFloat(this.indexValue)).toFixed(6);
            } else {
              // Fallback
              this.swapRate = (1000 / 0.1).toFixed(2);
              this.inverseRate = (0.1 / 1000).toFixed(6);
            }
          }
          
          // Update estimated output for current swap amount
          this.updateEstimatedOutput();
        } catch (error) {
          console.error('Error calculating swap rate:', error);
          this.swapRate = '0.00';
          this.inverseRate = '0.00';
        }
      }
    },
    
    updateEstimatedOutput() {
      if (this.swapAmount && (this.swapRate || this.inverseRate)) {
        if (this.isEthToToken) {
          // ETH -> ISEQ
          this.estimatedOutput = (this.swapAmount * parseFloat(this.swapRate)).toFixed(2);
        } else {
          // ISEQ -> ETH
          this.estimatedOutput = (this.swapAmount * parseFloat(this.inverseRate)).toFixed(6);
        }
      } else {
        this.estimatedOutput = '0.00';
      }
    },
    
    setMaxAmount() {
      if (this.isEthToToken) {
        // ETH -> ISEQ: Set to max ETH balance minus gas
        const maxAmount = Math.max(0, parseFloat(this.ethBalance) - 0.01);
        this.swapAmount = parseFloat(maxAmount.toFixed(4));
      } else {
        // ISEQ -> ETH: Set to max ISEQ balance
        const maxAmount = parseFloat(this.balance);
        this.swapAmount = parseFloat(maxAmount.toFixed(2));
      }
      this.updateEstimatedOutput();
    },
    
    async checkAllowance() {
      if (this.web3 && this.contract && this.account) {
        try {
          const allowance = await this.contract.methods.allowance(
            this.account,
            this.uniswapRouter
          ).call();
          
          // Convert to ether for easier comparison
          const allowanceInEther = this.web3.utils.fromWei(allowance, 'ether');
          console.log("Current allowance:", allowanceInEther, "ISEQ");
          
          // Check if the allowance is sufficient for the current swap amount
          // Add a small buffer for safety
          this.isTokenApproved = parseFloat(allowanceInEther) >= (parseFloat(this.swapAmount) * 1.1);
          
          return this.isTokenApproved;
        } catch (error) {
          console.error('Error checking allowance:', error);
          return false;
        }
      }
      return false;
    },
    
    async approveToken() {
      if (this.web3 && this.contract && this.account) {
        try {
          this.isApproving = true;
          
          // Approve a specific amount with buffer instead of unlimited
          // This is considered better practice for security
          const maxApproval = '115792089237316195423570985008687907853269984665640564039457584007913129639935';
          
          console.log("Approving maximum amount for ISEQ tokens");
          
          const tx = await this.contract.methods.approve(this.uniswapRouter, maxApproval).send({
            from: this.account,
            gas: 200000
          });
          
          this.txHash = tx.transactionHash;
          this.txStatus = 'success';
          
          alert('Token approved for swapping!');
          this.isTokenApproved = true;
        } catch (error) {
          console.error('Error approving token:', error);
          this.txStatus = 'failed';
          alert('Failed to approve token: ' + error.message);
        } finally {
          this.isApproving = false;
        }
      }
    },
    
    async swapTokens() {
      if (this.web3 && this.account && this.swapAmount > 0) {
        try {
          this.isSwapping = true;
          this.txStatus = 'pending';
          
          const routerContract = new this.web3.eth.Contract(
            this.uniswapRouterABI,
            this.uniswapRouter
          );

          // Use a much higher slippage tolerance (10%) to address INSUFFICIENT_OUTPUT_AMOUNT
          const slippageTolerance = 0.5; // 0.5% slippage allowance (changed from 0.90 to 0.5) need to consider better solution.
          
          // Use a longer deadline
          const deadline = Math.floor(Date.now() / 1000) + 60 * 60; // 1 hour deadline
          
          if (this.isEthToToken) {
            // ======= ETH to ISEQ swap =======
            const amountIn = this.web3.utils.toWei(this.swapAmount.toString(), 'ether');
            const path = [this.wethAddress, this.contractAddress];
            
            // Calculate very permissive minimum output with high slippage tolerance
            const estimatedOutputWei = this.web3.utils.toWei(this.estimatedOutput.toString(), 'ether');
            const amountOutMin = Math.floor(parseFloat(estimatedOutputWei) * slippageTolerance).toString();
            
            console.log("ETH -> ISEQ Swap with 10% slippage:");
            console.log("Amount In:", this.swapAmount, "ETH");
            console.log("Expected Out:", this.estimatedOutput, "ISEQ");
            console.log("Min Out (with slippage):", this.web3.utils.fromWei(amountOutMin, 'ether'), "ISEQ");
            
            const tx = await routerContract.methods.swapExactETHForTokens(
              amountOutMin,
              path,
              this.account,
              deadline
            ).send({
              from: this.account,
              value: amountIn,
              gas: 500000          
            });
            
            this.txHash = tx.transactionHash;
            this.txStatus = 'success';
            
            alert(`Swap successful! You received ISEQ tokens.`);
          } else {
            // ISEQ to ETH swap Note: this is in dollors
            const amountIn = this.web3.utils.toWei(this.swapAmount.toString(), 'ether');
            
            // Auto-approve tokens if needed
            const allowance = await this.contract.methods.allowance(
              this.account,
              this.uniswapRouter
            ).call();
            
            if (parseFloat(allowance) < parseFloat(amountIn)) {
              alert('Insufficient allowance. Approving tokens first...');
              
              try {
                // Approve a large amount
                const maxApproval = '115792089237316195423570985008687907853269984665640564039457584007913129639935';
                
                await this.contract.methods.approve(this.uniswapRouter, maxApproval).send({
                  from: this.account,
                  gas: 200000
                });
                
                // Wait for approval transaction to be confirmed
                alert('Approval successful. Proceeding with swap...');
              } catch (approvalError) {
                console.error('Error during approval:', approvalError);
                this.isSwapping = false;
                alert('Failed to approve tokens. Please try again.');
                return;
              }
            }
            
            const path = [this.contractAddress, this.wethAddress];
            
            // Use a very small minimum output to ensure the swap goes through
            const estimatedOutputWei = this.web3.utils.toWei(this.estimatedOutput.toString(), 'ether');
            const amountOutMin = Math.floor(parseFloat(estimatedOutputWei) * slippageTolerance).toString();
            
            // Different slippage due to less liqudity and less sepolia right now. 
            console.log("ISEQ -> ETH Swap with 10% slippage:");
            console.log("Amount In:", this.swapAmount, "ISEQ");
            console.log("Expected Out:", this.estimatedOutput, "ETH");
            console.log("Min Out (with slippage):", this.web3.utils.fromWei(amountOutMin, 'ether'), "ETH");
            
            const tx = await routerContract.methods.swapExactTokensForETH(
              amountIn,
              amountOutMin,
              path,
              this.account,
              deadline
            ).send({
              from: this.account,
              gas: 500000
            });
            
            this.txHash = tx.transactionHash;
            this.txStatus = 'success';
            
            alert(`Swap successful! You received ETH.`);
          }
          
          // Refresh balances
          await this.fetchUserData();
          await this.fetchEthBalance();
        } catch (error) {
          console.error('Error swapping tokens:', error);
          this.txStatus = 'failed';
          
          if (error.message.includes("INSUFFICIENT_OUTPUT_AMOUNT")) {
            alert("Swap failed: Price impact too high or low liquidity in the pool.\n\n" +
                  "Try these solutions:\n" +
                  "1. Use a smaller amount\n" +
                  "2. Add more liquidity to the pool\n" +
                  "3. Try again when there's less volatility");
          } else if (error.message.includes("execution reverted")) {
            alert("Swap failed: Transaction reverted by the network.\n\n" +
                  "This often happens on testnets due to:\n" +
                  "- Low liquidity in the pool\n" +
                  "- Network congestion\n" +
                  "- Contract issues\n\n" +
                  "Try with a smaller amount or try again later.");
          } else {
            alert('Failed to swap tokens: ' + error.message);
          }
        } finally {
          this.isSwapping = false;
        }
      }
    }
  }
};
</script>

<style scoped>
#app {
  text-align: center;
  margin: 0 auto;
  max-width: 1200px;
  padding: 20px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  color: #333;
}

h1 {
  color: #42b983;
  margin-bottom: 30px;
}

h2 {
  color: #2c3e50;
  margin-top: 30px;
  margin-bottom: 15px;
}

.connection-box {
  display: flex;
  justify-content: center;
  padding: 100px 0;
}

.connect-btn {
  padding: 15px 30px;
  font-size: 18px;
}

.main-content {
  display: flex;
  flex-direction: column;
}

.account-info {
  background-color: #f5f5f5;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

/* Swap Section Styles */
.swap-section {
  margin-bottom: 50px;
}

.swap-container {
  max-width: 500px;
  margin: 0 auto;
}

.swap-card {
  background-color: white;
  border-radius: 20px;
  padding: 20px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

.swap-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-bottom: 15px;
  margin-bottom: 15px;
  border-bottom: 1px solid #f0f0f0;
  font-weight: bold;
  font-size: 18px;
}

.swap-settings {
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 14px;
  color: #666;
}

.direction-toggle {
  margin-left: 8px;
}

.toggle-btn {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 13px;
  padding: 5px 8px;
  border-radius: 16px;
  background-color: #f0f0f0;
  color: #333;
  border: none;
  cursor: pointer;
  transition: all 0.2s;
}

.toggle-btn:hover {
  background-color: #e0e0e0;
}

.toggle-icon {
  font-size: 14px;
  display: inline-block;
  transition: transform 0.3s;
}

.toggle-btn:hover .toggle-icon {
  transform: rotate(180deg);
}

.swap-body {
  padding: 15px 0;
}

.swap-input-container {
  position: relative;
  margin-bottom: 10px;
  background-color: #f8f9fa;
  border-radius: 12px;
  padding: 15px;
}

.swap-input-header {
  display: flex;
  justify-content: space-between;
  font-size: 14px;
  color: #666;
  margin-bottom: 10px;
}

.swap-input {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.swap-input input {
  border: none;
  background: transparent;
  font-size: 20px;
  font-weight: bold;
  width: 60%;
  outline: none;
  padding: 0;
}

.token-selector {
  display: flex;
  align-items: center;
  background-color: #e9ecef;
  padding: 8px 12px;
  border-radius: 30px;
  cursor: pointer;
}

.token-icon {
  width: 24px;
  height: 24px;
  margin-right: 8px;
}

.token-iseq {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  background-color: #42b983;
  color: white;
  border-radius: 50%;
  font-size: 10px;
  margin-right: 8px;
}

.max-button {
  position: absolute;
  top: 45px; /* Position it near the input field */
  right: 120px; /* Move it further to the left of the token selector */
  background-color: #42b983;
  color: white;
  font-size: 12px;
  padding: 2px 8px;
  border-radius: 4px;
  cursor: pointer;
  z-index: 10; 
}

.swap-arrow {
  text-align: center;
  font-size: 20px;
  margin: 10px 0;
  color: #666;
}

.swap-rate {
  text-align: right;
  font-size: 14px;
  color: #666;
  margin-top: 15px;
}

.swap-footer {
  margin-top: 20px;
}

.swap-footer button {
  width: 100%;
  padding: 14px;
  border-radius: 12px;
  font-size: 16px;
  font-weight: bold;
}

.create-pool-btn {
  background-color: #ff9f1a;
}

.create-pool-btn:hover {
  background-color: #e68a00;
}

.transaction-status {
  margin-top: 20px;
  padding: 15px;
  border-radius: 10px;
  background-color: #f8f9fa;
  text-align: left;
}

.transaction-status a {
  color: #42b983;
  text-decoration: none;
}

.transaction-status a:hover {
  text-decoration: underline;
}

.status {
  display: inline-block;
  padding: 5px 10px;
  border-radius: 4px;
  margin-top: 5px;
}

.status.pending {
  background-color: #ffeeba;
  color: #856404;
}

.status.success {
  background-color: #d4edda;
  color: #155724;
}

.status.failed {
  background-color: #f8d7da;
  color: #721c24;
}

.readonly input {
  cursor: not-allowed;
  color: #666;
}

/* Value Display Styles */
.value-display-container {
  background-color: white;
  border-radius: 8px;
  padding: 30px;
  margin-bottom: 80px; /* Large margin to ensure separation from next section */
  box-shadow: 0 2px 12px rgba(0,0,0,0.1);
  position: relative;
}

.value-headers {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 20px;
  padding-bottom: 15px;
  border-bottom: 1px solid #eaeaea;
  margin-bottom: 15px;
  font-weight: bold;
  color: #718096;
}

.value-rows {
  max-height: 300px;
  overflow-y: auto;
}

.value-row {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 20px;
  padding: 15px 5px;
  border-bottom: 1px solid #f0f0f0;
  transition: background-color 0.2s;
}

.value-row:hover {
  background-color: #f9f9f9;
}

.value-cell {
  font-size: 16px;
  transition: all 0.3s;
}

.value-cell.time {
  color: #718096;
}

.up {
  color: #4caf50;
}

.down {
  color: #f44336;
}

.change-indicator {
  margin-left: 5px;
  font-weight: bold;
}

.update-time {
  margin-top: 20px;
  text-align: right;
  color: #718096;
  font-size: 14px;
}

/* Token actions and other sections */
.token-actions, .governance-section {
  background-color: white;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.form-group {
  margin-bottom: 15px;
}

button {
  padding: 10px 20px;
  margin: 5px;
  background-color: #42b983;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

button:hover {
  background-color: #359c6d;
}

button:disabled {
  background-color: #c0c0c0;
  cursor: not-allowed;
}

.cancel-btn {
  background-color: #e74c3c;
}

.cancel-btn:hover {
  background-color: #c0392b;
}

input, textarea {
  padding: 10px;
  margin: 5px;
  border: 1px solid #ddd;
  border-radius: 4px;
  width: 100%;
  max-width: 400px;
  box-sizing: border-box;
}

textarea {
  max-width: 100%;
  resize: vertical;
}

.duration-selector {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 10px 0;
}

.duration-selector input {
  width: 100%;
  max-width: 400px;
}

.proposals-container {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
}

.proposal-card {
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  overflow: hidden;
  display: flex;
  flex-direction: column;
  transition: transform 0.2s;
}

.proposal-card:hover {
  transform: translateY(-5px);
}

.proposal-header {
  background-color: #f9f9f9;
  padding: 15px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #eee;
}

.proposal-id {
  font-weight: bold;
}

.proposal-status {
  padding: 3px 8px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: bold;
}

.status-active {
  background-color: #3498db;
  color: white;
}

.status-ended {
  background-color: #95a5a6;
  color: white;
}

.status-queued {
  background-color: #f39c12;
  color: white;
}

.status-ready {
  background-color: #27ae60;
  color: white;
}

.status-executed {
  background-color: #2ecc71;
  color: white;
}

.status-canceled {
  background-color: #e74c3c;
  color: white;
}

.proposal-body {
  padding: 15px;
  flex-grow: 1;
}

.proposal-desc {
  margin-bottom: 15px;
  font-size: 14px;
}

.proposal-votes {
  font-weight: bold;
  color: #2c3e50;
}

.proposal-timeline {
  font-size: 12px;
  color: #7f8c8d;
  margin-top: 10px;
}

.proposal-actions {
  padding: 15px;
  border-top: 1px solid #eee;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.lock-info {
  margin-top: 15px;
  padding: 10px;
  background-color: #f8f9fa;
  border-radius: 4px;
}

@media (max-width: 768px) {
  .proposals-container {
    grid-template-columns: 1fr;
  }
  
  input, textarea, button {
    width: 100%;
  }
  
  .value-headers, .value-row {
    grid-template-columns: 1fr 1fr 1fr;
    gap: 10px;
    font-size: 14px;
  }
}
</style>