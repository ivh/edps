With this `pyproject.toml` you should be able to run ESO's `pyesorex` and `edps`
directly with [uv](https://docs.astral.sh/uv/), like so:

```bash
uv sync          # install dependencies into .venv
uv run pyesorex  # see if pyesorex runs, prints help info
uv run edps -lw  # start EDPS and list workflows
```

Note that it uses the re-packaged `pycpl` from [here](https://github.com/ivh/pycpl).

Apart from that, there are a few random markdown files, resulting from conversations
with ClaudeCode.
