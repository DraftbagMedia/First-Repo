# First-Repo
draftbag-ai-milestone-agent
import { ThirdwebSDK } from "@thirdweb-dev/sdk";
import { ethers } from "ethers";
import axios from "axios";
import { config } from "dotenv";
config();

async function init() {
  const provider = new ethers.providers.JsonRpcProvider(process.env.BASE_RPC);
  const sdk = new ThirdwebSDK(provider, { privateKey: process.env.PRIVATE_KEY });

  const contract = await sdk.getContract(process.env.CONTRACT_ADDRESS);
  contract.events.addEventListener("MilestoneCompleted", async (event) => {
    const { projectId, milestoneId, ipfsUri } = event.data;
    console.log(`MilestoneCompleted received: Project ${projectId}, Milestone ${milestoneId}`);

    const { data: content } = await axios.get(`https://ipfs.io/ipfs/${ipfsUri}`);
    const aiResponse = await axios.post("https://api.openai.com/v1/chat/completions", {
      model: "gpt-4",
      messages: [
        { role: "system", content: "You are an AI milestone verifier for Draftbag Media." },
        { role: "user", content: `Please evaluate the following achievement content: ${JSON.stringify(content).slice(0,1000)}` }
      ]
    },{
      headers: { Authorization: `Bearer ${process.env.OPENAI_KEY}` }
    });

    const verdict = aiResponse.data.choices[0].message.content;
    console.log(`AI verdict: ${verdict}`);

    if (verdict.toLowerCase().includes("approved")) {
      await contract.call("approveMilestone", [projectId, milestoneId]);
      console.log("Milestone approved on-chain.");
    } else {
      console.log("Milestone not approved — requires review.");
    }

    if (process.env.NOTION_API_KEY) {
      await axios.post("https://api.notion.com/v1/pages", {
        parent: { database_id: /* your-notion-db-id */ },
        properties: {
          Name: {
            title: [{ text: { content: `Project ${projectId} — Milestone ${milestoneId}` } }]
          },
          Status: { select: { name: verdict.toLowerCase().includes("approved") ? "Approved" : "Flagged" } }
        }
      }, {
        headers: {
          "Authorization": `Bearer ${process.env.NOTION_API_KEY}`,
          "Notion-Version": "2022-06-28"
        }
      });
      console.log("Notion updated with milestone status.");
    }
  });
}

init().catch(console.error);
3. Deploy your draftbag smart contract (with MilestoneCompleted event & approveMilestone call).
4. Start the agent:


## 🧠 How It Works
- Listens for `MilestoneCompleted(event)`
- Fetches IPFS content
- Uses GPT-4 to approve or flag
- Calls `approveMilestone()` if approved
- Updates Notion database (optional)

## 📦 Deploying to Replit
1. Import from GitHub
2. Fill in Secrets (OPENAI_KEY, BASE_RPC, ...)
3. Run the code – server will stay live
