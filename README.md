# Qwen3-TTS Voice Studio — Build It with One Prompt

**Create your own local AI voice studio with Codex or Claude Code — using a single implementation prompt.**

This repository contains an **implementation specification (`prompt.md`)**, not a prebuilt application. Give the prompt to an AI coding agent, and it will guide the agent through building a Windows-based web application powered by [Qwen3-TTS](https://github.com/QwenLM/Qwen3-TTS).

Designed for game developers, content creators, and anyone who wants to experiment with character voices and dialogue generation on their own computer.

> **Unofficial project:** This is an independent community project. It is not affiliated with, endorsed by, or sponsored by the Qwen team or Alibaba Cloud.

## What the prompt asks the AI agent to build

The specification describes a local application with:

- **Character profiles** — create characters and save their voice settings for future sessions.
- **Three voice workflows** — built-in Qwen voices, AI-designed voice candidates, and cloning from a reference recording.
- **Dialogue Studio** — enter dialogue, select available voice settings, and generate multiple takes.
- **Audio Library** — preview, rename, organize, and export generated recordings.
- **Model Manager** — review model information before choosing which model to download.
- **Local web interface** — a dark-themed UI served from a Python backend.
- **REST API and CLI** — use the studio from scripts or other local tools.
- **Windows setup, tests, and documentation** — included as requirements for the AI-generated project.

These are **requested features**, not a guarantee that every AI agent will implement and verify them successfully on every machine. In particular, voice identity, expressiveness, and supported languages depend on the model and its version.

## Requirements

To **build** the application:

- **Windows 11** (the target platform in the prompt).
- An AI coding agent capable of creating and testing project files, such as **OpenAI Codex** or **Claude Code**.
- Access to the agent and any account, subscription, or usage allowance it requires.
- Internet access for installing dependencies and downloading model files.
- Enough disk space and memory for the chosen Qwen3-TTS model.

The prompt asks the agent to use **Python 3.12 (64-bit)** for the generated application and to detect the computer's CPU, RAM, GPU/VRAM, and free disk space. **A compatible NVIDIA GPU is optional**; the specification includes a CPU inference path, which can be significantly slower.

## Quick start

### 1. Get the prompt

Download [`prompt.md`](./prompt.md) from this repository and put it in a **new, empty project folder** on your computer.

The prompt is the starting point; you are not downloading an already-built voice studio.

### 2. Open the folder in your AI coding agent

Open that project folder in **Codex** or **Claude Code** and give it this instruction:

```text
Read prompt.md and build the application according to the entire specification.
Start implementing and continue through testing and documentation.
```

Allow the agent to create the project files and review its requests before approving commands or installations. Read the agent's final report for anything not implemented or tested.

### 3. Set up the generated application

If the implementation follows the specification, it will provide Windows setup and launch files. From **PowerShell in the generated project folder**, the expected commands are:

```powershell
.\setup.ps1
.\start.bat
```

If PowerShell blocks local scripts, follow your system's security policy rather than disabling protections indiscriminately. Refer to the **README generated inside your new application** for its actual setup requirements.

### 4. Download a voice model and generate audio

The intended application runs locally at:

```text
http://127.0.0.1:8765
```

After it starts, use the Model Manager to **review and explicitly download** a compatible Qwen3-TTS model. Then create a character, select its voice, and generate a line of dialogue. Model weights are **not included** in this repository and can require substantial storage.

## Intended voice workflows

| Workflow | Purpose | Notes |
| --- | --- | --- |
| Preset Voice | Create a character with a built-in Qwen speaker | Direct delivery control depends on the selected model. |
| Voice Design | Generate candidate voices and save one as a reference | A supported Base model is used for subsequent dialogue cloning. |
| Import Reference | Use an audio recording you have permission to use | Requires an accurate transcript and rights confirmation. |

The prompt requires the generated application to show limitations clearly instead of pretending unsupported emotion controls work. Voice cloning is not guaranteed to reproduce a person's voice perfectly.

## Privacy and responsible use

- The intended application performs **inference locally**, without sending voice-generation requests to a cloud service.
- Initial installation and model downloads **do require network access**.
- Use only voice recordings you own or have **permission** to use. Do not impersonate others deceptively.
- Review commands and downloaded dependencies before running AI-generated software.
- Generated audio may be subject to applicable laws, rights of third parties, and the licenses of the models or assets you use.

## License and third-party software

The original **prompt and documentation in this repository** are shared under the [MIT License](./LICENSE), if that license file is included in the repository.

**Qwen3-TTS is a separate upstream project.** This repository does not contain Qwen model weights or the Qwen source code. Qwen3-TTS and other third-party dependencies remain subject to their **own** applicable licenses and notices. The prompt instructs the generated application to preserve relevant third-party license information and to use Apache License 2.0 for its newly generated application code.

Official project and model sources:

- [Qwen3-TTS on GitHub](https://github.com/QwenLM/Qwen3-TTS)
- [Qwen3-TTS CustomVoice 1.7B](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice)
- [Qwen3-TTS VoiceDesign 1.7B](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-VoiceDesign)
- [Qwen3-TTS Base 0.6B](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-0.6B-Base)

Always review the current license and usage terms for the specific model, package, and assets you download.

## Video tutorial

A walkthrough showing the app and how to use the prompt will be linked here.

<!-- Replace the sentence above with the link to your published YouTube video. -->

## Important notes

- **This repository provides a prompt, not executable application files.**
- Results may vary across coding agents, versions, operating systems, and hardware.
- Building a complete application may require multiple agent sessions and debugging.
- The app's **local operation** does not imply the AI coding agent is free to use.
- If something fails, check the generated project's logs, tests, and documentation before reporting a bug against Qwen itself.

---

Created by **N3W GameDev**. If you build something with this prompt, feel free to share your experience in the comments on the tutorial video.
