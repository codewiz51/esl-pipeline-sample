# Setup

The pipeline needs Python 3.7 or later and `requests`. It has been tested
on Python 3.10, 3.13 and 3.14. To generate real lessons it also needs
Ollama and a model.

**Reference setup** (the versions the pipeline was last tested with):
Mac Studio M4 Max with 36 GB, Ollama 0.40.6, macOS 27.0.1, the model
`pipeline-qwen38:latest` (built over `qwen3.8:27b-mlx`, about 18 GB),
Python 3.14, requests 2.34, pandoc 3.11 (only for `toDocx.py`).

## 1. Python

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cd sourcecode
python3 -m unittest discover -s tests -t .       # expect: Ran 90 tests ... OK
```

## 2. Ollama and the model

1. Install Ollama from https://ollama.com and start it (`ollama serve`
   or the Mac app).
2. Pull the base model and build the pipeline model:
   ```bash
   ollama pull qwen3.8:27b-mlx
   ollama create pipeline-qwen38 -f pipeline-qwen38.modelfile
   ollama list | grep pipeline-qwen38
   ```
   If the pull can't find that tag, look up the current Qwen 27B tag on
   https://ollama.com and change the `FROM` line in the Modelfile.
   The Modelfile sets the sampling parameters and a short, mechanical
   system prompt. It deliberately has no persona: the lesson templates
   are the only instructions the model should see.
3. Memory: the model takes about 18 GB plus room for a 64k-token context.
   Don't run another large model during a lesson run.

`*-mlx` models need Apple Silicon and an Ollama version with MLX support.
On other hardware, use any model you like: change `FROM` in the Modelfile
and `MODEL_NAME` in `sourcecode/config.py`. Smaller models will follow
the template rules less well.

## 3. Machine settings (the placeholders)

The original pipeline ran on two Macs: a Studio that runs the model and a
laptop that sends its calls over the network. `sourcecode/config.py`
recognises each machine by its home folder and hostname:

```python
MACHINES = {
    "macbook": {"home": "/Users/laptop-user", "hostname_hints": ["macbook"], "mode": "client"},
    "studio":  {"home": "/Users/studio-user", "hostname_hints": ["studio"],  "mode": "local"},
}
DEFAULT_STUDIO_IP = "192.168.1.50"
```

The home folders and the address are placeholders. You have two options:

- **Pass `--machine` every time (easiest).** `--machine studio` means
  "Ollama runs on this computer" (it uses `localhost:11434`).
  `--machine macbook --studio-ip HOST:PORT` means "Ollama runs on another
  computer at that address".
- **Edit `MACHINES`** to match your own home folder (`echo $HOME`) and
  hostname (`hostname`), and set `DEFAULT_STUDIO_IP`. Then detection is
  automatic. Setting the environment variable `ESL_MACHINE` also works.

If Ollama runs on another computer, start it there with
`OLLAMA_HOST=0.0.0.0 ollama serve`, allow it through the firewall, and
check the path with `curl -s -m 5 http://HOST:11434/api/tags`.

## 4. First check

```bash
cd sourcecode
python3 pipeline.py 1 Week01StoryBoard.md MainLessonTemplate.txt Debug --machine studio --check
```

Expect `Ollama OK`, the model name, and `Check passed`. It takes seconds
and generates nothing. Then run without `--check` for a real lesson
(about 30 to 45 minutes), followed by the Five-Minute run shown in the
top-level `README.md`.

## Folders the code expects

| Folder | Used for |
|---|---|
| `sourcecode/` | the pipeline; run every command from here |
| `source_docs/WeeklyTemplates/` | inputs: `WeekNNStoryBoard.md`, the two templates, `promptFive.md`, `reference.docx` |
| `source_docs/WeeklyLessons/` | outputs: `NN_MAIN_PASSPIPELINE_<time>.md`, `NN_FIVEMIN_PASSPIPELINE_<time>.md`, `mainSupport.md` |
| `runs/` | created automatically: one folder per run with the log, checkpoints and debug files |

## If something fails

| Symptom | Fix |
|---|---|
| `ModuleNotFoundError: No module named 'requests'` | activate the venv, or `pip install -r requirements.txt` |
| `Can't tell which machine this is` | pass `--machine`, or edit `MACHINES` (step 3) |
| `Cannot reach Ollama at ...` | Ollama isn't running there, or the address is wrong |
| `Model 'pipeline-qwen38:latest' is not installed` | step 2: `ollama create pipeline-qwen38 -f pipeline-qwen38.modelfile` |
| a run stopped partway | run the same command again with `--resume` |
| `toDocx.py` fails | install pandoc |
