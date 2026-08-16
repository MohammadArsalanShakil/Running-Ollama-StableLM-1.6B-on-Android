# Running Ollama + StableLM 2 1.6B on Android

Run a local LLM directly on an Android phone with **Ollama**, **Termux**, and a lightweight **StableLM 2 1.6B** model.

The setup uses a Linux userspace environment inside Termux because Termux itself runs against **Android's Bionic libc**, while Ollama's Linux binaries expect a **glibc-based environment**.

The resulting stack looks like:

```text
📱 Android Phone
        │
        ▼
🐧 Termux
        │
        ▼
🐧 proot-distro Linux environment
        │
        ▼
🦙 Ollama
        │
        │ localhost:11434
        ▼
🧠 StableLM 2 1.6B
        │
        ▼
💬 Local HTTP inference
```

The model used here is **StableLM 2 1.6B**, quantized to roughly **1 GB**. It has a **4K token context window**, making it better suited to short exchanges and lightweight tasks than long multi-turn conversations.

---

## What You Will Build

By the end of this guide, your Android phone will be able to:

- Run Ollama locally
- Run StableLM 2 1.6B
- Generate responses without a cloud API
- Expose the Ollama HTTP API locally on port `11434`
- Serve as a potential AI node for automation
- Be reachable from other trusted devices when combined with a private network such as Tailscale

The important distinction is that this is not just about running an LLM.

The interesting part is turning the phone into a **node that other services can call**:

```text
n8n ──────────────┐
                  │
IoT Controller ────┤
                  ▼
              Android
                  │
               Ollama
                  │
                  ▼
          StableLM 2 1.6B
```

---

# 1. Prerequisites

You will need:

- An Android phone
- Termux
- Internet access for the initial setup and model download
- Several GB of free storage
- At least 4 GB RAM recommended
- Basic Linux command-line knowledge
- A device with an ARM64 (`aarch64`) architecture is recommended

More RAM and a newer CPU will generally provide a better experience.

> **Important:** Performance varies significantly between Android devices. CPU-only inference can be slow and may produce noticeable heat and battery drain.

---

# 2. Install Termux

Install Termux from a current trusted source such as **F-Droid** or the official Termux project releases.

Avoid relying on an old Termux build because package compatibility can become a problem.

Open Termux and update the package repositories:

```bash
pkg update && pkg upgrade -y
```

Install the basic tools:

```bash
pkg install -y git curl wget proot-distro
```

You may also want common development dependencies:

```bash
pkg install -y python clang make pkg-config libffi openssl
```

---

# 3. Give Termux Storage Access

Run:

```bash
termux-setup-storage
```

Android will ask for storage permission.

Allow it.

Shared Android storage should then be available through:

```text
~/storage/shared
```

You can verify it with:

```bash
ls ~/storage/shared
```

---

# 4. Check Your Architecture

Before installing the Linux environment, check the phone's architecture:

```bash
uname -m
```

A modern 64-bit Android phone will normally return something similar to:

```text
aarch64
```

You can also check:

```bash
dpkg --print-architecture
```

An ARM64 phone is the recommended target for this guide.

---

# 5. Install a Linux Environment with proot-distro

Termux itself uses Android's **Bionic libc**, rather than the standard **glibc** found on most desktop/server Linux distributions.

This matters because Linux binaries such as Ollama's standard distribution expect a glibc-based userspace.

`proot-distro` provides a Linux userspace environment inside Termux.

Install it if you haven't already:

```bash
pkg install -y proot-distro
```

List the available distributions:

```bash
proot-distro list
```

Install Debian:

```bash
proot-distro install debian
```

Enter the Debian environment:

```bash
proot-distro login debian
```

You should now be inside the Linux environment.

Verify:

```bash
cat /etc/os-release
```

You should see Debian information.

---

# 6. Update Debian

Inside the Debian environment:

```bash
apt update && apt upgrade -y
```

Install the basic dependencies:

```bash
apt install -y curl wget git ca-certificates procps
```

You can also install useful utilities:

```bash
apt install -y nano htop lsof net-tools
```

---

# 7. Install Ollama

Still inside the Debian environment, try the standard Ollama installation script:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Verify the installation:

```bash
ollama --version
```

If the command works, Ollama is installed.

You can also check its location:

```bash
which ollama
```

---

# 8. Start Ollama

Start the Ollama server:

```bash
ollama serve
```

By default, Ollama serves its API on:

```text
http://127.0.0.1:11434
```

Keep this process running.

Open another Termux session and enter the Debian environment again:

```bash
proot-distro login debian
```

You can now use Ollama from the second session.

---

# 9. Verify the Ollama API

Run:

```bash
curl http://127.0.0.1:11434/api/tags
```

If Ollama is running, you should receive a JSON response describing the installed models.

For example:

```json
{
  "models": []
}
```

An empty model list is fine at this stage.

It means the Ollama API is working and you simply haven't downloaded a model yet.

---

# 10. Download StableLM 2 1.6B

Pull the StableLM model available through your Ollama installation:

```bash
ollama pull stablelm
```

Then check the installed models:

```bash
ollama list
```

You should see the StableLM model listed.

> **Model naming note:** Ollama model tags can change over time. Verify the exact StableLM 2 1.6B tag available in the current Ollama model library rather than assuming that every installation uses the same tag.

The model used for this setup is **StableLM 2 1.6B**, with a quantized size of approximately **1 GB**.

---

# 11. Run the Model

Start an interactive session:

```bash
ollama run stablelm
```

Try a simple prompt:

```text
>>> Explain machine learning in simple terms.
```

You should receive a response directly from the model running on the Android phone.

At this point:

```text
No cloud API
No external server
No laptop
```

The inference is happening locally on the phone.

---

# 12. Test Local Inference Through HTTP

Ollama provides an HTTP API.

The default local endpoint is:

```text
http://127.0.0.1:11434
```

You can send a generation request with:

```bash
curl http://127.0.0.1:11434/api/generate \
  -d '{
    "model": "stablelm",
    "prompt": "Explain artificial intelligence in one paragraph.",
    "stream": false
  }'
```

The response will contain the generated text and inference metadata.

This HTTP interface is what makes the setup useful beyond an interactive terminal.

Instead of manually typing prompts, another application can call the model.

---

# 13. The Important Part: Turn It Into a Node

Running a local LLM is interesting.

Making that LLM available to other services is where it becomes much more useful.

The architecture can become:

```text
             ┌──────────────┐
             │     n8n      │
             └──────┬───────┘
                    │
                    │ HTTP
                    ▼
            ┌───────────────┐
            │ Android Phone │
            │               │
            │    Ollama     │
            │       │       │
            │       ▼       │
            │ StableLM 2    │
            │    1.6B       │
            └───────────────┘
```

Other possible callers include:

- IoT controllers
- Local applications
- Python services
- Discord bots
- Automation workflows
- Development tools
- Other devices on a private network

---

# 14. Expose Ollama Beyond Localhost

By default, Ollama listens locally.

If you want another device to connect to the phone, configure Ollama to listen on the desired interface.

For example:

```bash
export OLLAMA_HOST=0.0.0.0:11434
ollama serve
```

Now Ollama can accept connections beyond `127.0.0.1`.

However:

> **Do not expose port `11434` directly to the public internet.**

Ollama should be placed behind a trusted/private network or an appropriate authentication and security layer.

---

# 15. Using Tailscale

A private overlay network such as Tailscale is a good option when the phone needs to communicate with your other devices without exposing Ollama publicly.

A possible architecture is:

```text
             Private Tailscale Network
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
     Laptop           n8n          IoT Device
       │               │
       └───────────────┼────────────────┘
                       │
                       ▼
                Android Phone
                       │
                    Ollama
                       │
                       ▼
                StableLM 2 1.6B
```

The phone becomes a small private AI endpoint available to the rest of your infrastructure.

---

# 16. Connecting n8n

Once Ollama is reachable from the machine running n8n, you can use an **HTTP Request** node.

Send a request to:

```text
http://<android-phone-ip>:11434/api/generate
```

with a JSON body similar to:

```json
{
  "model": "stablelm",
  "prompt": "Analyze this event and determine whether it requires attention.",
  "stream": false
}
```

The workflow can then process the returned response.

For example:

```text
Event
  ↓
n8n
  ↓
HTTP Request
  ↓
Android Ollama
  ↓
StableLM 2 1.6B
  ↓
AI Response
  ↓
n8n continues workflow
```

This is the point where the phone stops being merely a demonstration and starts becoming an **automation node**.

---

# 17. Possible Automation Use Cases

Once another service can call the model, there are many possibilities.

## 🔐 Private Local Assistants

Process short prompts locally without sending them to a cloud AI provider.

## 🛠️ Personal Automation Agents

Use the model to classify events or make lightweight decisions inside automation workflows.

## 📡 AI-Powered IoT

An IoT controller could send sensor information to the phone and receive a lightweight AI-generated decision.

```text
Sensor
  ↓
IoT Controller
  ↓
Ollama API
  ↓
StableLM
  ↓
Decision
  ↓
IoT Controller
```

## 🤖 n8n Agents

Use n8n as the orchestration layer while Android provides local inference.

```text
Trigger
  ↓
n8n
  ↓
Local LLM
  ↓
Decision
  ↓
Action
```

## 💻 Lightweight Coding Assistance

The model can be used for simple coding explanations, text transformations, and lightweight development tasks.

Do not expect the same capability as large modern coding models.

## 🌐 Private Services

With Tailscale or another private networking solution, the phone can act as a small AI service accessible to trusted devices.

---

# 18. StableLM 2 1.6B Limitations

The model is intentionally small.

The **4K token context window** means it is better suited to:

- Short prompts
- Classification
- Simple text generation
- Lightweight automation
- Short conversations
- Small structured tasks
- Experimentation

It is less suitable for:

- Long documents
- Large codebases
- Long multi-turn conversations
- Complex reasoning
- Large-scale autonomous agents

The goal here is not to compete with large cloud models.

The goal is to have a **small, local, inexpensive inference node** that can run on hardware you already carry.

---

# 19. Performance and Thermal Considerations

CPU-only inference on a phone has trade-offs.

Performance depends heavily on:

- CPU architecture
- CPU performance
- Number of cores
- RAM
- Thermal throttling
- Quantization
- Android background process management
- Available storage

The phone can become noticeably warm during sustained inference.

Battery consumption can also become significant if Ollama is processing requests continuously.

For unattended deployments, monitor:

```text
CPU usage
RAM usage
Temperature
Battery level
Inference latency
```

A phone running short, occasional requests is a very different workload from one performing continuous inference.

---

# 20. Keeping Ollama Alive

One of the practical challenges is keeping the server running in the background.

Android can terminate background processes to save battery.

Things to consider:

- Disable battery optimization for Termux
- Keep Termux excluded from aggressive battery management
- Avoid unnecessary background applications
- Use persistent terminal sessions where appropriate
- Monitor whether the Ollama process is still running
- Keep the phone connected to power for sustained workloads

Check whether Ollama is running:

```bash
ps aux | grep ollama
```

You can also check the API:

```bash
curl http://127.0.0.1:11434/api/tags
```

If the API responds, Ollama is alive.

---

# 21. Troubleshooting

## `ollama: command not found`

Check:

```bash
which ollama
```

If nothing is returned, verify that you are inside the Debian `proot-distro` environment:

```bash
cat /etc/os-release
```

Then check whether Ollama was installed successfully.

---

## Ollama Installation Script Fails

Remember that Ollama is being run inside the Linux userspace, not directly against Termux's Android/Bionic environment.

Check:

```bash
uname -m
```

and:

```bash
ldd --version
```

The latter should show the glibc implementation available inside the Linux environment.

---

## Port 11434 Is Already in Use

Check:

```bash
ss -ltnp | grep 11434
```

If Ollama is already running, don't start a second instance.

---

## Model Download Fails

Check available storage:

```bash
df -h
```

Then retry:

```bash
ollama pull stablelm
```

Also verify your internet connection.

---

## Android Kills Ollama

Check Android's battery optimization settings and exclude Termux from battery optimization.

Some manufacturers apply additional background process restrictions, so you may need to adjust their device-specific power-management settings as well.

---

## The Phone Gets Too Hot

CPU-only inference can generate substantial heat.

Try:

- Reducing request frequency
- Avoiding continuous inference
- Running shorter prompts
- Using the phone while connected to power only when appropriate
- Allowing the device to cool between sustained workloads
- Monitoring temperature during testing

Do not treat a phone as a continuously loaded server without considering thermal and battery limitations.

---

# 22. Recommended Architecture

For an automation-oriented deployment:

```text
                         ┌──────────────┐
                         │   Discord    │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │     n8n      │
                         └──────┬───────┘
                                │
                         HTTP / Tailscale
                                │
                                ▼
                     ┌───────────────────┐
                     │   Android Phone   │
                     │                   │
                     │     Termux        │
                     │        │          │
                     │  proot-distro     │
                     │        │          │
                     │     Ollama        │
                     │        │          │
                     │        ▼          │
                     │ StableLM 2 1.6B   │
                     └───────────────────┘
```

This gives you:

**Automation → Network → Local inference → Response → Automation**

The phone becomes a lightweight AI worker rather than just a device running a chatbot.

---

# 23. Security Considerations

If Ollama is reachable from other devices, treat it as a network service.

Avoid:

```text
Internet
   │
   ▼
Ollama:11434
```

Prefer:

```text
Internet
   │
   ▼
Authenticated / Private Network
   │
   ▼
Tailscale
   │
   ▼
Android Phone
   │
   ▼
Ollama
```

Do not expose Ollama's port directly to the public internet without an appropriate authentication, authorization, and network-security layer.

Also avoid sending sensitive information to the model unless you understand how your surrounding automation handles that data.

---

# 24. Useful Verification Commands

### Check Android/Termux architecture

```bash
uname -m
```

### Enter Debian

```bash
proot-distro login debian
```

### Check Linux distribution

```bash
cat /etc/os-release
```

### Check glibc

```bash
ldd --version
```

### Check Ollama

```bash
ollama --version
```

### List models

```bash
ollama list
```

### Check Ollama API

```bash
curl http://127.0.0.1:11434/api/tags
```

### Check Ollama process

```bash
ps aux | grep ollama
```

### Check port

```bash
ss -ltnp | grep 11434
```

### Check storage

```bash
df -h
```

---

# 25. Final Architecture

The complete setup looks like:

```text
📱 Android Phone
       │
       ▼
🐧 Termux
       │
       ▼
🐧 Debian via proot-distro
       │
       ▼
🦙 Ollama
       │
       │ localhost:11434
       ▼
🧠 StableLM 2 1.6B
       │
       ▼
💬 Local inference
```

And once connected to an automation environment:

```text
Automation / IoT / Services
            │
            ▼
           n8n
            │
            ▼
     Tailscale / Private Network
            │
            ▼
      Android + Ollama
            │
            ▼
     StableLM 2 1.6B
            │
            ▼
       AI Response
            │
            ▼
      Automation Action
```

---

# 26. What Makes This Interesting?

The interesting part isn't simply:

> "I managed to run an LLM on my phone."

The more useful idea is:

> **"My phone can now be a node in my automation infrastructure."**

Once something can call the model and get a response back, the possibilities expand.

For example:

```text
IoT Device
    ↓
n8n
    ↓
Android Ollama
    ↓
StableLM
    ↓
Decision
    ↓
n8n
    ↓
Action
```

That could eventually mean:

- Local AI-powered IoT
- Private assistants
- Lightweight agents
- n8n automation
- Local event classification
- Device monitoring
- Tailscale-accessible AI services

The next step is turning the model into part of a **loop**, rather than keeping it as a one-way demo.

---

# 27. Known Limitations

This setup has several practical limitations:

1. **Android background management** can kill Termux/Ollama.
2. **CPU-only inference** can be slow.
3. Sustained inference can cause **thermal throttling**.
4. Continuous inference can consume significant **battery power**.
5. StableLM 2 1.6B has a relatively small **4K context window**.
6. A 1.6B model is not comparable to large modern cloud models for complex reasoning or coding.
7. Ollama's Android/Termux compatibility can change as versions are updated.
8. Exact model tags available through Ollama may change.

This setup is best viewed as an **experimental, lightweight local AI node**, rather than a replacement for a dedicated GPU server.

---

# 28. Next Steps

Once the basic setup works, possible next experiments include:

- Connect Ollama to n8n
- Connect the phone to Tailscale
- Build a local Discord AI bot
- Connect IoT sensors to the model
- Build a lightweight local agent
- Add persistent automation workflows
- Measure inference speed on different phones
- Monitor battery and thermal performance
- Experiment with other small local models

The real goal is to move from:

```text
"I have an LLM running on Android."
```

to:

```text
"I have an Android device acting as a local AI node
inside my automation infrastructure."
```

---

## License

Feel free to adapt this guide for your own projects.

If you improve the setup or discover a more reliable way to run Ollama and StableLM 2 1.6B on Android, consider contributing the findings back to the project.
