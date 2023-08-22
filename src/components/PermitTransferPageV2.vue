<template>
  <div class="PermitTransferPage">
    <div v-if="isConnected">
      <p>Connected Address: {{ address }}</p>
      <div>
        <input type="text" v-model="spender" placeholder="Enter Spender Address"/>
        <input type="text" v-model="erc20Address" placeholder="Enter ERC20 Address"/>
        <input type="date" v-model="selectedDate" placeholder="Select Date"/>
        <input type="time" v-model="selectedTime" placeholder="Select Time"/>
        <button @click="handlePermit">Submit</button>
      </div>
    </div>
    <button v-else @click="connectWallet">Connect Wallet</button>
  </div>
</template>

<script>
import Web3 from 'web3';
import ABI from "../contents"

export default {
  name: 'PermitTransferPage',
  data() {
    return {
      isConnected: false,
      address: "",
      web3: null,
      spender: "",
      deadline: Math.floor(Date.now() / 1000) + 3600, // Default: 1 hour from now
      selectedDate: "",
      selectedTime: "",
      erc20Address: "",
      contractABI: ABI,
      contractAddress: "0x51Edef037EAD278171f9262389d0A19B2951330b"
    };
  },
  mounted() {
    this.connectWallet();
  },
  methods: {
    async connectWallet() {
      if (window.ethereum) {
        const web3Instance = new Web3(window.ethereum);
        try {
          const addresses = await window.ethereum.request({method: 'eth_requestAccounts'});
          const address = addresses[0];
          this.isConnected = true;
          this.address = address;
          this.web3 = web3Instance;
        } catch (error) {
          console.error("Error connecting to wallet:", error);
        }
      } else {
        console.error("Ethereum provider not detected");
      }
    },
    async handlePermit() {
      if (this.selectedDate && this.selectedTime) {
        const combinedDateTime = new Date(`${this.selectedDate}T${this.selectedTime}:00`); // assuming UTC timezone
        this.deadline = Math.floor(combinedDateTime.getTime() / 1000);
      }

      if (this.web3 && this.address) {
        const contract = new this.web3.eth.Contract(this.contractABI, this.contractAddress);

        // 1. Get the nonce for the user
        const nonce = await contract.methods.getNonce(this.address).call();

        // 2. Prepare EIP-712 signature data
        const domain = {
          name: "Permit2TransferAll",
          version: "1",
          chainId: await this.web3.eth.net.getId(),
          verifyingContract: this.contractAddress
        };

        const message = {
          token: this.erc20Address,
          amount: "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          nonce: nonce,
          deadline: this.deadline
        };

        const data = {
          types: {
            EIP712Domain: [
              {name: "name", type: "string"},
              {name: "version", type: "string"},
              {name: "chainId", type: "uint256"},
              {name: "verifyingContract", type: "address"}
            ],
            TokenPermissions: [
              {name: "token", type: "address"},
              {name: "amount", type: "uint256"},
              {name: "nonce", type: "uint256"},
              {name: "deadline", type: "uint256"}
            ]
          },
          domain: domain,
          primaryType: "TokenPermissions",
          message: message
        };


        // 3. Use MetaMask to sign data
        // const serializedData = this.convertBigInts(data);
        const serializedData = JSON.stringify(data, (_, v) => typeof v === 'bigint' ? v.toString() : v);
        const signature = await window.ethereum.request({
          method: 'eth_signTypedData_v4',
          params: [this.address, serializedData]
        });
        console.log("Signature:", signature)
        // 4. Call permitTransferFrom with the signature
        const tokenPermissions = {
          token: this.erc20Address,
          amount: "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          nonce: nonce,
          deadline: this.deadline
        };

        const transferDetails = {
          to: this.spender,
          requestedAmount: "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"
        };
        console.log("执行函数")
        try {
          await contract.methods.permitTransferFrom(tokenPermissions, transferDetails, this.address, signature).send({from: this.address});
          console.log("Success!")
        } catch (e) {
          console.error("Error calling permitTransferFrom:", e);
        }
        console.log("执行完函数")

      }
    }
  }
};
</script>

<style scoped src="../css/PermitTransferPage.css"></style>
