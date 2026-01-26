

Here is the concise plan. Since Claude Code connects to APIs, we need to turn your downloaded GGUF files into an Ollama "model" first, then tell Claude Code to talk to Ollama.

**The Workflow:**
1. Move files from phone to PC.
2. Import them into Ollama using a simple "Modelfile".
3. Run Claude Code pointing to your local Ollama.

---

### Part 1: Windows PC (RTX 3060 12GB)
*Start here. The GPT-OSS 20B model fits your VRAM perfectly.*

**Step 1: Move Files**
Create a folder `C:\Models` and move your downloads there:
- `C:\Models\GLM-4.7-Flash-Q4_K_M.gguf`
- `C:\Models\gpt-oss-20b-Q4_K_M.gguf`

**Step 2: Import Models into Ollama**
Open **PowerShell** or **Command Prompt**.

*2a. Import GPT-OSS 20B (Recommended for Windows)*
Create a file named `Modelfile` in `C:\Models\gpt-oss` with this content:
```dockerfile
FROM ./gpt-oss-20b-Q4_K_M.gguf
```
Run:
```powershell
cd C:\Models\gpt-oss
ollama create gpt-oss-local -f Modelfile
```

*2b. Import GLM 4.7 Flash (Optional, will use System RAM)*
Create a file named `Modelfile` in `C:\Models\glm47` with this content:
```dockerfile
FROM ./GLM-4.7-Flash-Q4_K_M.gguf
```
Run:
```powershell
cd C:\Models\glm47
ollama create glm47-local -f Modelfile
```

**Step 3: Run Claude Code**
Set the environment variables to tell Claude Code to use Ollama, then start it.

```powershell
# 1. Set environment variables for this session
$env:ANTHROPIC_AUTH_TOKEN="ollama"
$env:ANTHROPIC_BASE_URL="http://localhost:11434"

# 2. Start Claude Code using the model that fits your VRAM best
claude --model gpt-oss-local
```
*(Note: You can switch to `glm47-local` if you want to test the Flash model, but it will be slower on the 12GB GPU).*

---

### Part 2: Mac Mini (M1 16GB)
*The M1 unified memory handles the larger GLM model better.*

**Step 1: Move Files**
Create a folder `~/Models` and move your downloads there:
- `~/Models/GLM-4.7-Flash-Q4_K_M.gguf`
- `~/Models/gpt-oss-20b-Q4_K_M.gguf`

**Step 2: Import Models into Ollama**
Open **Terminal**.

*2a. Import GLM 4.7 Flash (Recommended for Mac)*
```bash
mkdir -p ~/Models/glm47
cd ~/Models/glm47
echo "FROM ./GLM-4.7-Flash-Q4_K_M.gguf" > Modelfile
ollama create glm47-local -f Modelfile
```

*2b. Import GPT-OSS 20B*
```bash
mkdir -p ~/Models/gpt-oss
cd ~/Models/gpt-oss
echo "FROM ./gpt-oss-20b-Q4_K_M.gguf" > Modelfile
ollama create gpt-oss-local -f Modelfile
```

**Step 3: Run Claude Code**
```bash
# 1. Set environment variables
export ANTHROPIC_AUTH_TOKEN="ollama"
export ANTHROPIC_BASE_URL="http://localhost:11434"

# 2. Start Claude Code
claude --model glm47-local
```

---

### Summary: Which model to use on which machine?

| Machine | Best Model | Why? |
| :--- | :--- | :--- |
| **Windows (RTX 3060)** | **gpt-oss-local** | The 11.7GB file fits mostly in your 12GB VRAM for fast speeds. |
| **Mac (M1 16GB)** | **glm47-local** | The 18.1GB MoE model runs well on unified memory; active parameters are fast. |

**Pro Tip:** If Ollama is running slowly on Windows with the GLM model, you can force it to use the CPU or limit VRAM usage by setting this variable before running `ollama serve`:
```powershell
$env:OLLAMA_GPU_OVERHEAD="0"
```
