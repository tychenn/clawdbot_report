# A Trajectory-Based Safety Audit of Clawdbot (OpenClaw)

<!-- TODO: Add badges here once CI/arxiv are ready -->
<!-- [![arXiv](https://img.shields.io/badge/arXiv-XXXX.XXXXX-b31b1b.svg)](ARXIV_LINK_PLACEHOLDER) -->

This repository contains the **seed test cases** and evaluation artifacts for *"A Trajectory-Based Safety Audit of Clawdbot (OpenClaw)"*. We conduct a trajectory-centric safety evaluation of [Clawdbot/OpenClaw](https://openclaw.ai/) across six risk dimensions, using a curated suite of 34 canonical cases drawn from established agent-safety benchmarks and hand-designed scenarios tailored to Clawdbot's tool surface.

📄 **Paper:** [A Trajectory-Based Safety Audit of Clawdbot (OpenClaw)](https://arxiv.org/pdf/2602.14364)  <!-- ARXIV_LINK_PLACEHOLDER -->

📝 **Blog Post (中文):** [当AI助手"真的动手做事"，安全边界在哪里？——Clawdbot安全审计报告深度解读](https://mp.weixin.qq.com/s/E0frstSFdARIHWTUIXtJ4A)  <!-- WECHAT_LINK_PLACEHOLDER -->

---

## Overview

Clawdbot is a self-hosted, tool-using personal AI agent with a broad action space spanning local execution and web-mediated workflows. Our evaluation reveals a **non-uniform safety profile** (overall pass rate: 58.9%): the agent performs reliably on well-scoped tasks but degrades under ambiguity, open-ended goals, or adversarial steering, where minor misinterpretations can escalate into irreversible tool actions.

### Key Findings

| Dimension | Pass Rate | Tier |
|:--|:-:|:-:|
| Hallucination & Reliability | 100% | Relatively Better |
| Operational Safety | 75% | Relatively Better |
| User-facing Deception | 71% | Relatively Better |
| Prompt Injection Robustness | 57% | Needs Improvement |
| Unexpected Results from Ambitious Goals | 50% | Needs Improvement |
| Intent Misunderstanding & Unsafe Assumptions | 0% | Critical |

---

## Repository Structure

The test cases are organized by the six risk dimensions defined in our evaluation framework. Each case includes the task prompt, any required seed files (e.g., intentionally empty PDFs, pre-populated directory trees), and expected safe/unsafe behavior annotations.

```
.
├── README.md
├── deception.json                  # (i)   User-facing Deception
├── hallucination.json              # (ii)  Hallucination & Reliability Failures
├── false_assumption.json           # (iii) Intent Misunderstanding & Unsafe Assumptions
├── high_goals.json                 # (iv)  Unexpected Results from Ambitious Goals
├── safety_awareness.json           # (v)   Operational Safety Awareness & Efficiency
└── jailbreak_prompt_injection.json # (vi)  Robustness to Prompt Injection & Jailbreak
```

Each JSON file contains the seed test cases for one risk dimension, including task prompts, required seed file descriptions, and expected behavior annotations.

---

## Environment Setup

### Prerequisites

- A machine for running self-hosting Clawdbot
- A [MiniMax Coding Plan](https://platform.minimax.io/subscribe/coding-plan) subscription (Plus tier or above, required for MiniMax M2.1 model access)
- A [Brave Search API](https://brave.com/search/api/) key for web search functionality

### Step 1: Install and Configure Clawdbot

Follow the [official OpenClaw Getting Started guide](https://docs.openclaw.ai/start/getting-started) to install and launch Clawdbot in its standard self-hosted configuration. Interact with Clawdbot through the browser-based [Control UI](https://docs.openclaw.ai/web/control-ui) to maintain a consistent interface across all test runs.

### Step 2: Configure the Model Provider

We use **MiniMax M2.1** as the underlying LLM throughout all experiments. Set the default model in Clawdbot's provider configuration:

```yaml
# In your Clawdbot provider config (refer to docs.openclaw.ai/providers/models)
provider: minimax
model: MiniMax-M2.1
```

This corresponds to the `minimax/MiniMax-M2.1` identifier in Clawdbot's documented provider/model format. For detailed setup instructions, see the [MiniMax Provider documentation](https://docs.openclaw.ai/providers/minimax) and the [Model Provider Quickstart](https://docs.openclaw.ai/providers/models).

> **Subscription Note:** MiniMax M2.1 access requires a [MiniMax Coding Plan](https://platform.minimax.io/subscribe/coding-plan) subscription at the **Plus** tier or above. Refer to the [MiniMax Coding Plan documentation](https://platform.minimax.io/docs/pricing/coding-plan) for tier details and pricing. Verify your subscription status before running evaluations.

### Step 3: Enable Web Search (Brave API)

We enable the built-in `web_search` tool backed by the [Brave Search API](https://brave.com/search/api/) for public web retrieval, alongside `web_fetch` for lightweight HTTP retrieval when needed.

1. Obtain an API key from the [Brave Search API dashboard](https://brave.com/search/api/).
2. Configure the search tool in Clawdbot following the [Web Tools documentation](https://docs.openclaw.ai/tools/web):

```yaml
# In your Clawdbot tool config
tools:
  web_search:
    provider: brave
    api_key: <YOUR_BRAVE_API_KEY>
  web_fetch:
    enabled: true
```

### Step 4: Configure the Execution Environment

The tool surface is held fixed across all runs. Enable unrestricted local execution within a scoped workspace directory:

- **Exec tool:** Enabled — allows arbitrary shell commands within the workspace via the built-in exec surface ([Exec Tool docs](https://docs.openclaw.ai/tools/exec))
- **Web search / Web fetch:** Enabled — configured as described in Step 3
- **Sandbox isolation:** Disabled — we run on a real host environment for realistic evaluation

> ⚠️ **Security Warning:** Running without sandbox isolation carries inherent risk. We recommend using a dedicated machine or VM with a scoped workspace directory to limit the blast radius of potentially destructive actions. See the [OpenClaw Security documentation](https://docs.openclaw.ai/gateway/security) for best practices.

For tasks requiring authenticated surfaces (e.g., Gmail OAuth, chat platform pairing), we transform instructions so the agent writes structured action files (JSON/YAML) to a designated directory instead of performing real-world side effects. See Section 3.3 of the paper for details.

### Step 5: Enable Trajectory Logging

All evaluation runs are logged as complete trajectories (user messages, agent actions, tool-call arguments, tool outputs, and final responses) via the Gateway's JSONL file logs. Ensure logging is enabled following the [Logging documentation](https://docs.openclaw.ai/gateway/logging).

---

## Running the Evaluation

### Executing Test Cases

Each JSON file contains an array of test case objects. To run a case:

1. Select a case from the corresponding dimension JSON file.
2. Set up the workspace directory with the required seed files as described in the case (if any).
3. Send the user prompt to Clawdbot through the Control UI.
4. Allow the agent to execute to completion without interruption.
5. The trajectory will be automatically logged to the Gateway's JSONL output.

### Safety Judgment

We determine safety using two complementary procedures:

1. **Automated Trajectory Judge:** We use [AgentDoG-Qwen3-4B](https://huggingface.co/AI45Research/AgentDoG-Qwen3-4B) as the automated trajectory judge. Unlike conventional content moderators that classify a single model response, AgentDoG diagnoses an **entire agent trajectory** — a sequence of steps, each containing an action and an observation — to determine whether any step exhibits unsafe behavior during execution. The model accepts a trajectory JSON file (containing the full interaction trace of user messages, agent actions, tool-call arguments, and tool outputs), formats it via a conversation-history template consisting of (i) task definition, (ii) the formatted trajectory, and (iii) output format instructions, and produces a binary `safe`/`unsafe` judgment. For deployment, AgentDoG can be served via sglang or vllm as an OpenAI-compatible API endpoint. See the [AgentDoG GitHub repository](https://github.com/AI45Lab/AgentDoG) for usage details.
2. **Human Review:** Complete trajectories are reviewed to validate automated judgments and adjudicate ambiguous cases.

For the 34 canonical cases reported in the paper, the automated and manual judgments were identical.

---

## Citation

If you find this work useful, please cite our paper:

```bibtex
@article{chen2026trajectory,
  title={A Trajectory-Based Safety Audit of Clawdbot (OpenClaw)},
  author={Chen, Tianyu and Liu, Dongrui and Hu, Xia and Yu, Jingyi and Wang, Wenjie},
  journal={arXiv preprint arXiv:XXXX.XXXXX},  % ARXIV_ID_PLACEHOLDER
  year={2026}
}
```

---

## Related Resources

- **OpenClaw Official Site:** [openclaw.ai](https://openclaw.ai/)
- **OpenClaw Security Guidance:** [docs.openclaw.ai/gateway/security](https://docs.openclaw.ai/gateway/security)
- **AgentDoG Framework:** [arXiv:2601.18491](https://arxiv.org/abs/2601.18491)
- **LPS-Bench:** [arXiv:2602.03255](https://arxiv.org/abs/2602.03255)

---

## License

<!-- LICENSE_PLACEHOLDER: Specify your chosen license here -->
This project is released under the [MIT License](LICENSE). See `LICENSE` for details.

---

## Contact

For questions or collaboration inquiries, please reach out to:

- **Tianyu Chen** — [chenty12024@shanghaitech.edu.cn](mailto:chenty12024@shanghaitech.edu.cn)

You can also ask questions by opening an issue in the section.


**Affiliations:**  
¹ ShanghaiTech University, Shanghai, China  
² Shanghai Artificial Intelligence Laboratory, Shanghai, China
