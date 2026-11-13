## CI Regression Testing (GitHub Actions)

Every change to the chatbot code or prompt automatically reruns a reduced version of the attack suite, so a weakened defence is caught before it goes unnoticed.

### How it works

```mermaid
flowchart LR
    Dev["Edit app/ and git push"] --> GH["GitHub detects the push"]
    GH --> WF["Workflow: red-team-regression"]
    WF --> VM["Fresh Ubuntu runner"]
    VM --> Setup["Install Python 3.12, Ollama, llama3.2:1b"]
    Setup --> Run["Run attack_runner.py"]
    Run --> Check{"Bypass rate over limit?"}
    Check -->|No| Pass["Run passes"]
    Check -->|Yes| Fail["Run fails"]
    Run --> Art["Upload results as artifact"]
```

<img width="1095" height="887" alt="image" src="https://github.com/user-attachments/assets/ed808ae2-8b68-4748-acf0-92800b80a5cd" />


1. **Trigger:** the workflow starts on any push that changes a file under `app/`, or manually from the Actions tab (`workflow_dispatch`).
2. **Environment:** GitHub creates a clean Ubuntu runner. It installs Python 3.12, installs Ollama and pulls `llama3.2:1b`. No external LLM API is used; the model runs on the runner itself.
3. **Test run:** `attack_runner.py` runs the attack set and the benign baseline against the chatbot with these settings:

| Setting | Value | Purpose |
|---|---|---|
| `DEFENCE` | `filtered` | Hardened prompt and output filter on, with the fake secret kept in the prompt as a canary |
| `MODEL` | `llama3.2:1b` | Small model, because the runner has no GPU |
| `RUNS` | `2` | Repeats per prompt, kept low to limit run time |
| `MAX_BYPASS` | `30` | Maximum acceptable attack bypass rate (%) |

4. **Gate:** if the attack bypass rate is above `MAX_BYPASS`, the script exits with an error and the run is marked failed.
5. **Evidence:** the `results/` folder is uploaded as the `redteam-results` artifact on every run, pass or fail, and can be downloaded from the run page.
6. **Cleanup:** the runner is deleted after the run. CI result files are not committed back to the repo.

### What the pipeline protects

If someone later weakens the system prompt, removes the output filter, or changes the model, the planted fake secret leaks again, the bypass rate rises above the limit, and the run turns red.

### Latest CI run

- Status: [passed / failed]
- Attack bypass rate: [X]%  |  Benign refusal rate: [Y]%
- Screenshot: `docs/ci-run.png` (add yours)

### Limitations

- CI uses a 1B model with 2 repeats per prompt, so its numbers are a **regression signal**, not comparable with the 3B-model results measured locally.
- The refusal check for benign prompts is a phrase-matching heuristic and can miss or over-count refusals.
- The bypass limit is a project-specific threshold and not an industry standard.
