# Pytest

## Install and confirm it's there

```bash
uv add --dev pytest pytest-cov
uv run pytest --version
```
You already have this — just the official first step for completeness.

## Your first test

```python
# tests/core/test_video_inspector.py
from clipper.core.video_inspector import get_video_metadata

def test_get_video_metadata_returns_none_on_missing_file():
    result = get_video_metadata("does_not_exist.mp4")
    assert result is None
```

Run it:
```bash
uv run pytest
```
No base class, no registration, no decorator required — pytest finds this because the file starts with `test_`, and the function inside starts with `test_`. That naming convention *is* the mechanism; there's nothing else wiring it up.

## How pytest finds your tests at all

pytest walks the current directory and subdirectories looking for files matching `test_*.py` or `*_test.py`, then inside those, functions matching `test_*` (or methods inside classes matching `Test*`). This is why `tests/core/test_video_inspector.py` gets picked up automatically the moment it exists — no config needed for the basic case. Your `justfile`'s `uv run pytest --cov=src` already relies entirely on this discovery working correctly.

## Assertions — just use `assert`

```python
def test_fps_math():
    assert round(29.970029970029969, 2) == 29.97
```
pytest rewrites plain `assert` statements to show you exactly what each side evaluated to on failure — no `assertEqual`/`assertTrue`-style method zoo required. If this fails, the output shows both the expression and the actual computed values, not just "assertion failed."

**Comparing floats — use `pytest.approx`, not `==`:**
```python
import pytest

def test_duration_seconds():
    assert 125.5 == pytest.approx(float("125.500001"))
```
Floating-point math almost never lands on an exact value — your own `fps` calculation from `Fraction` is a perfect example. `pytest.approx` handles the tiny rounding tolerance for you instead of you hand-writing `abs(a - b) < 0.001` everywhere.

**`pytest.approx` also works on whole nested structures, not just single numbers** — dicts inside lists inside a dict, compared in one assertion:
```python
def test_transcribe_matches_golden_output(sample_speech_path, sample_transcript):
    result = transcribe(str(sample_speech_path))
    assert result == pytest.approx(sample_transcript)
```
`pytest.approx` recursively walks the structure: numbers get compared with float tolerance, strings and everything else get compared with exact equality. This is what makes a "golden file" regression test practical — comparing a whole transcript (segments, nested word lists, timestamps) against a saved expected version without hand-writing a loop over every field, and without the test breaking on harmless floating-point drift between model versions.

## Testing that an error is actually raised

```python
import pytest

def test_extract_audio_raises_on_bad_codec():
    with pytest.raises(ffmpeg.Error):
        extract_audio("in.mp4", "out.wav", codec="not_a_real_codec")
```
`pytest.raises` is how you assert "this should fail, and specifically with this exception" — the test fails if the code *doesn't* raise, and fails if it raises the *wrong* exception type. Both are meaningful outcomes to catch.

**Only applies when the exception actually escapes to the caller.** A function that catches its own exception and returns `None` (like `get_video_metadata` and `extract_audio` both do) will never trigger `pytest.raises` — there's nothing propagating out to catch. Test those with the return value instead: `assert result is None`, plus `caplog` to confirm the right thing got logged. Rule of thumb: if the function has a `return None` sitting inside its `except` block, test the return value, not an exception.

## Parametrize — one test, many inputs

Running the same test logic against several inputs without copy-pasting the test body once per case:

```python
@pytest.mark.parametrize(
    "timestamp,expected_seconds",
    [
        ("00:00:01", 1.0),
        ("00:01:00", 60.0),
        ("01:00:00", 3600.0),
    ],
)
def test_validate_timestamp_parses_valid_formats(timestamp, expected_seconds):
    assert validate_timestamp(timestamp) == pytest.approx(expected_seconds)
```

Read it in three pieces: the first argument to `parametrize` is a string naming the parameter(s), comma-separated — these become real parameters on the test function below, same mechanism as a fixture. The second argument is a list of tuples, one tuple per case, values in the same order as the names. pytest then runs the function once per tuple, not once total — three tuples in, three independent tests out, each showing up separately in the output (`test_..._formats[00:00:01-1.0]`) and each able to pass or fail on its own.

That independence is the actual point, not just avoiding repetition. A hand-rolled `for` loop inside one test stops at the *first* failing case — you never find out whether the others would have failed too. `parametrize` runs every case regardless, so one run tells you exactly which inputs are broken, not just that "the test failed."

**The signal to reach for it:** any time you're about to write two or three near-identical test functions that only differ in their input values — a list of valid formats, a list of malformed ones — that's exactly what `parametrize` replaces.

## Grouping tests in a class (optional, use when it earns its keep)

```python
class TestGetVideoMetadata:
    def test_returns_none_for_missing_file(self):
        assert get_video_metadata("missing.mp4") is None

    def test_has_audio_true_when_audio_stream_present(self):
        ...
```
Worth it for shared setup, shared fixtures scoped to just that group, or applying a marker to every test in the class at once. Not worth it just to organize things visually — plain functions in a well-named file do that already. One sharp edge worth knowing: each test method gets a **fresh instance** of the class. Setting `self.value = 1` in one test does not carry over to the next — that isolation is deliberate, not a bug, and relying on shared mutable state between test methods is exactly the kind of coupling that makes tests fail in confusing, order-dependent ways.

## Fixtures

A fixture is a function that provides something a test needs, and pytest wires it in automatically when a test asks for it by name as a parameter.

```python
# tests/conftest.py
import pytest
from pathlib import Path

@pytest.fixture
def sample_video_path() -> Path:
    return Path(__file__).parent / "fixtures" / "sample.mp4"
```
```python
# tests/core/test_video_inspector.py
def test_metadata_against_real_file(sample_video_path):
    result = get_video_metadata(str(sample_video_path))
    assert result is not None
```
pytest sees `sample_video_path` in the test's signature, finds the matching fixture, runs it, and passes the return value in — no import needed if the fixture lives in `conftest.py`.

**Why `conftest.py` specifically:** it's a file pytest auto-discovers, and fixtures defined in it are automatically available to every test file in the same directory *and* every subdirectory below it — no import statement anywhere. A `conftest.py` at `tests/` root makes its fixtures available to `tests/core/` and `tests/api/` both. You can also have a narrower `conftest.py` inside `tests/core/` for fixtures only relevant to that subfolder.

**Fixture scope controls how often it re-runs:**
```python
@pytest.fixture(scope="function")  # default — fresh for every single test
@pytest.fixture(scope="module")    # once per test file
@pytest.fixture(scope="session")   # once for the entire test run
```
Use `function` (the default) unless setup is genuinely expensive and safe to reuse — e.g., generating your synthetic sample video once per session rather than once per test.

**Built-in fixtures worth knowing exist**, no setup required, just add as a parameter:
- `tmp_path` — a real, unique, auto-cleaned-up directory per test, for anything that needs to actually write a file.
- `caplog` — captures log output, so you can assert your `logger.error(...)` calls actually fired.
- `capsys` — captures stdout/stderr, useful if you're testing CLI output directly.
- `monkeypatch` — pytest's own tool for temporarily changing something (an attribute, an environment variable, a dict entry) for the duration of one test, with automatic cleanup after — covered more below.

Run `pytest --fixtures` any time to see every fixture available in your project, built-in and custom.

**`tmp_path` in practice:**
```python
def test_extract_audio_writes_real_file(sample_video_path, tmp_path):
    output_path = tmp_path / "output.wav"
    result = extract_audio(str(sample_video_path), str(output_path))

    assert result == str(output_path)
    assert output_path.exists()
```
This is a real integration-style check — it actually runs FFmpeg and checks a real file appeared — using a throwaway directory pytest creates and deletes for you, so nothing pollutes your actual project folder.

## Mocking: two tools, different jobs

**`unittest.mock.patch`** — the general-purpose Python mocking tool, not pytest-specific, best for replacing an entire function/object and asserting on how it was called:
```python
from unittest.mock import patch

def test_metadata_no_audio():
    fake_probe = {
        "format": {"filename": "silent.mp4", "duration": "10.0"},
        "streams": [{"codec_type": "video", "r_frame_rate": "25/1"}],
    }
    with patch("clipper.core.video_inspector.ffmpeg.probe", return_value=fake_probe):
        result = get_video_metadata("silent.mp4")

    assert result["has_audio"] is False
```

**`monkeypatch`** — pytest's own fixture, best for simpler swaps: an environment variable, a single attribute, a dict entry — anywhere you don't need call-count assertions, just a temporary substitution:
```python
def test_uses_default_bitrate_env_var(monkeypatch):
    monkeypatch.setenv("CLIPPER_DEFAULT_BITRATE", "256k")
    assert get_default_bitrate() == "256k"
```
No `with` block, no manual cleanup — `monkeypatch` automatically reverts everything it touched the moment the test finishes, even if the test fails partway through.

**The mental model worth keeping:** imagine your code is a toy robot, and you want to test what happens when one part breaks — without actually breaking the real part, just pretending for a little while.

`monkeypatch` is like borrowing a magic eraser and sticky note from pytest's own toy box. You stick a note over the real part saying "pretend this does X instead," and when you're done, pytest peels the note off by itself — automatically, every time, even if the test breaks mid-way.

`patch` is your own sticky note, brought from home (`unittest.mock`). It works the same way — cover the part, pretend, peel it off — but *you* own the peeling. In this project, that always happens via a `with` block, which peels the note off automatically the instant the block ends, success or failure — so in practice, both tools clean up after themselves equally reliably here. The only version where `patch` genuinely requires you to remember cleanup is the manual form (`patcher = patch(...); patcher.start()` … `patcher.stop()`), which we don't use.

**When to pick one over the other, in this project:**
- Just want a part to return something different ("pretend this says quack instead of moof")? Either sticky note works — `monkeypatch` is the house rule for that case, so it's the default.
- Want a part to break on purpose ("pretend this explodes when pressed")? `patch(..., side_effect=...)` is the shorter, more direct tool for that job — `side_effect` is a built-in option on `patch`/`Mock` itself, so reaching for `patch` skips a layer of indirection you'd otherwise add by routing the same exploding `Mock` through `monkeypatch.setattr(...)` instead.

**When to reach for which, directly:**

| Need | Use |
|---|---|
| Assert *how* something was called (call count, exact arguments) | `patch` — `mock_input.assert_called_once_with("in.mp4")` |
| Mock a chained/fluent call (`ffmpeg.input(...).output(...).run()`) | `patch` — `MagicMock`'s auto-chaining is what makes walking the chain possible at all |
| Swap an environment variable for one test | `monkeypatch.setenv(...)` |
| Temporarily change a single attribute or dict entry, no call inspection needed | `monkeypatch.setattr(...)` / `monkeypatch.setitem(...)` |
| Replace an entire external dependency (an API client, a whole module-level function) | `patch` |

The short version: if the test needs to check *how* the mock was used, reach for `patch`. If it just needs something to temporarily *be* a different value with zero cleanup ceremony, `monkeypatch` is the lighter tool for the job. Nothing stops you from using `patch` everywhere — `monkeypatch` is a convenience for the simpler half of what `patch` can already do, not a separate capability.

**The rule that matters regardless of which tool you use: mock the boundary, not your own logic.** Mock `ffmpeg.probe` — the point where your code hands off to something external. Don't mock the parsing logic you're actually trying to verify; a test that mocks everything, including the thing under test, just proves your mocks return what you told them to.

**The single most common mistake — patching the wrong path:** patch *where the name is looked up*, not where it's defined. Your module does `import ffmpeg` and calls `ffmpeg.probe(...)`, so `ffmpeg` lives in *that module's* namespace at call time — patch `"clipper.core.video_inspector.ffmpeg.probe"`, not `"ffmpeg.probe"`. Get this wrong and the mock silently never engages — your test quietly calls the real function instead.

**`return_value` vs `side_effect`:**
```python
patch(..., return_value=fake_probe)                                    # always returns this
patch(..., side_effect=ffmpeg.Error(cmd="ffprobe", stdout=b"", stderr=b"bad file"))  # raises this instead
```

## How to actually set up your test files

- **Mirror your `src/` structure inside `tests/`.** `src/clipper/core/video_inspector.py` → `tests/core/test_video_inspector.py`. Anyone can find the test for any file by pattern-matching the path.
- **`tests/` lives next to `src/`, never inside it** — you already know this one from the earlier structure discussion, and it still applies: nesting tests inside the package risks shipping them in the built wheel and risks accidentally importing local code instead of the installed package.
- **No `__init__.py` needed inside `tests/`** — pytest doesn't require your test directories to be Python packages. Leave them out unless you have a specific reason (e.g., two test files in different folders with the identical name).
- **One `conftest.py` at `tests/` root** for fixtures shared everywhere (like `sample_video_path`); add narrower `conftest.py` files inside subfolders only when a fixture is genuinely local to that area.
- **Fixture files live with the fixtures they hold**, e.g. `tests/fixtures/sample.mp4` — separate from test *code*, so it's obvious at a glance which files are data versus logic.
- **Mark slow/external tests distinctly**, e.g. `@pytest.mark.integration` for anything hitting real FFmpeg, and register the marker in `pyproject.toml`:
```toml
[tool.pytest.ini_options]
markers = ["integration: calls real ffmpeg/ffprobe against a fixture file"]
```
Run the fast suite day-to-day with `pytest -m "not integration"`, and everything (including integration) in CI.

## Mocking a chained call

Not every dependency is a single function call. `ffmpeg-python`'s fluent API (`ffmpeg.input(...).output(...).overwrite_output().run(...)`) is four chained calls, and mocking it needs walking the chain, not just patching one name:

```python
def test_extract_audio_sad_path(caplog):
    fake_error = ffmpeg.Error(cmd="ffmpeg", stdout=b"", stderr=b"encoding failed")

    with patch("clipper.core.video_inspector.ffmpeg.input") as mock_input:
        mock_input.return_value.output.return_value.overwrite_output.return_value.run.side_effect = fake_error
        result = extract_audio("in.mp4", "out.wav")

    assert result is None
    assert "error" in caplog.text.lower()
```
`MagicMock` auto-creates a new mock for every attribute you touch, so `mock_input.return_value` stands in for whatever `.input()` returned, `.output` on that is the next link, and so on — you set `side_effect` on the mock sitting at the very end of the chain, the one representing the real call that can actually fail.

## Debugging a test without adding print statements everywhere

**`pytest -s`** unsilences stdout — pytest swallows `print()` output on passing tests by default, which is the single most common source of "why isn't my print showing up":
```bash
uv run pytest -s
```

**`-k "pattern"`** runs only tests whose name matches, instead of the whole suite — the one you'll reach for constantly while actively debugging one thing:
```bash
uv run pytest -k "extract_audio"
```

**`-x`** stops at the first failure instead of dumping every failure in the suite at once:
```bash
uv run pytest -x
```

**`--pdb`** drops you into a live debugger automatically, at the exact point of failure, on any test that fails — no need to guess where to place a `breakpoint()` ahead of time:
```bash
uv run pytest --pdb
```
Once inside: type any variable name to inspect it, `c` continues, `q` quits.

**`-vv`** shows full, un-truncated values when an assertion on a dict or long string fails, instead of a shortened diff:
```bash
uv run pytest -vv
```

**`--lf`** ("last failed") reruns only what failed last time, skipping everything that already passed — useful once you're fixing a handful of failures one at a time:
```bash
uv run pytest --lf
```

Put together, this is the actual debugging loop, not five separate tricks: `pytest -k extract_audio -x --pdb` — run just the test in question, stop at the first failure, land directly in an interactive session at the line that broke, every local variable available to inspect live.

**No native line-number selection.** `pytest file.py:29` isn't a real pytest feature — selection is by name (`::test_name`) or by `-k` keyword match. Editors that let you click a line and "run this test" (VS Code, PyCharm) are translating that click to a name-based selection behind the scenes, not passing pytest a line number. A small plugin, `pytest-line-runner`, adds literal line-number selection if you want it — worth knowing it exists, not essential.

**Peeking at a return value without a throwaway script — the fastest option:** deliberately assert something wrong, and read the real value off pytest's own failure diff:
```python
def test_scratch(sample_video_path):
    result = get_video_metadata(str(sample_video_path))
    assert result == {}  # wrong on purpose — reveals the real dict in the failure output
```
Delete it once you've seen what you needed. `--showlocals` (`-l`) does something similar automatically — dumps every local variable's value on any failure, no deliberate wrong-assert needed.

## Dealing with slow tests

Measure before touching anything — don't guess at what's slow:
```bash
uv run pytest --durations=10
```
Shows the 10 slowest tests after a run, with real timings. Often it's one or two outliers, not the whole suite — optimizing the wrong thing wastes the time you're trying to save.

Mark anything that loads a real model or runs real inference, same pattern as `integration`:
```python
@pytest.mark.slow
def test_transcribe_real_audio_produces_valid_shape(sample_speech_path):
    ...
```
```toml
[tool.pytest.ini_options]
markers = [
    "integration: calls real ffmpeg/ffprobe against a fixture file",
    "slow: tests that load a real model or run real transcription",
]
```
Fast loop while actively coding, everything before a push:
```bash
pytest -m "not slow"    # quick feedback
pytest                  # everything
```

**Module-level caching often solves most of the actual slowness on its own** — a global loaded once (`_model` in `transcriber.py`, for instance) means only the *first* test in a run pays the load cost; everything after it in the same run reuses the cached instance. Confirm this with `--durations=10`: if only the first slow test is genuinely slow and the rest are fast, caching is already doing its job. If they're all slow, something's reloading when it shouldn't be.

If the whole suite is slow, not just a handful of tests, parallelize instead of chasing individual tests:
```bash
uv add --dev pytest-xdist
uv run pytest -n auto
```
Only worth reaching for once the *sum* of your fast tests is dragging — for most projects, `-m "not slow"` alone solves the day-to-day pain.

## Coverage is a flashlight, not a scoreboard

`pytest --cov` shows which lines never ran during your tests — good for spotting blind spots. It does not measure whether your tests are actually good; 100% coverage is trivially achievable with tests that assert nothing meaningful. Use it to find what you forgot to think about, not as a number to maximize.

## A short checklist for testing anything new

1. What's the happy path?
2. What's at least one edge case (empty input, missing field, boundary value)?
3. What's the failure mode — what should happen when the dependency fails or input is invalid?
4. Which parts need mocking (external, slow, side-effecting) and which are safe to run for real?
5. One test, one question — if the test name needs "and" in it, it's probably two tests.

Everything above — fixtures, `conftest.py`, markers, `monkeypatch` — exists to make following that checklist less repetitive. The checklist is the actual discipline; the syntax is just how you express it.
