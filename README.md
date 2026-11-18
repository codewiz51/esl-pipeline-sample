# ESL lesson pipeline: sample

A sanitized sample of the pipeline described in the blog series
"Rebuilding My ESL Lesson Pipeline". It turns a weekly storyboard into a
set of seven daily lessons (Main) and a five-minute review (Five-Minute),
one small model call at a time, with checkpoints so a failed run can
resume.

Everything in `source_docs/` is invented: the coffee shop, the
characters, the storyboard. The code, the pass structure and the 90 unit
tests are the real ones. License: MIT (see `LICENSE`).

## Try it in five minutes, with no model

You need Python 3.7 or later and `requests`:

```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cd sourcecode
python3 -m unittest discover -s tests -t .        # Ran 90 tests ... OK
```

Then run the whole pipeline against a fake Ollama server. The mock writes
well-formed but meaningless lessons, so this tests the plumbing (passes,
checkpoints, resume, exit codes), not lesson quality.

```bash
# terminal 1
python3 tools/mock_ollama.py --port 11500

# terminal 2 (from sourcecode/, venv active)
python3 pipeline.py 1 Week01StoryBoard.md MainLessonTemplate.txt Debug \
    --machine macbook --studio-ip 127.0.0.1:11500
```

Add `--check` to validate the inputs and the connection without
generating anything. To see a failure and a resume, start the mock with
`--mode failat:4`, run the command above (it stops with exit code 2), then
restart the mock with `--mode ok` and run it again with `--resume`.

## Generate a real lesson

You need Ollama and a model. `docs/SETUP.md` has the steps. In short:

```bash
ollama pull qwen3.8:27b-mlx
ollama create pipeline-qwen38 -f pipeline-qwen38.modelfile
cd sourcecode
python3 pipeline.py 1 Week01StoryBoard.md MainLessonTemplate.txt Debug --machine studio
python3 supportPipeline.py 1 mainSupport.md Week01StoryBoard.md FiveMinuteTemplate.txt promptFive.md Debug --machine studio
```

On the reference setup (a 27B model on an M4 Max with 36 GB) the Main
lesson takes about 30 to 45 minutes and the Five-Minute lesson about 10.
Lessons are written to `source_docs/WeeklyLessons/`. Machine-specific
settings are placeholders in `sourcecode/config.py`; `docs/SETUP.md`
explains them.

## What is in the box

| Path | What it is |
|---|---|
| `sourcecode/` | the pipeline, the tests (`tests/`) and the mock server (`tools/mock_ollama.py`) |
| `sourcecode/README.md` | how the passes work |
| `source_docs/WeeklyTemplates/` | the two templates, the Five-Minute prompt, the invented `Week01StoryBoard.md` and `reference.docx` (Word styles) |
| `source_docs/WeeklyLessons/` | where lessons are written; includes a generated example |
| `pipeline-qwen38.modelfile` | the Ollama model recipe |
| `docs/SETUP.md` | setup and the settings you must change |

## Notes

- The generated lessons are meant to be corrected by a teacher before
  students see them. The example in `source_docs/WeeklyLessons/` is
  uncorrected model output.
- The MIT license covers the code, the templates, the invented storyboard
  and the example lessons. It does not cover the model weights, which have
  their own license from their publisher.
