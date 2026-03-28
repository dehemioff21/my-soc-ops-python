# Copilot Workspace Instructions — Soc Ops

> **Project:** Social Bingo game for in-person mixers built with Python, FastAPI, Jinja2, and HTMX.  
> **Tech Stack:** Python 3.13+, FastAPI, Uvicorn, Pydantic, Jinja2, HTMX, pytest, ruff

---

## Quick Reference

### Setup & Dev Commands

```bash
# First time setup
uv sync                                               # Install all dependencies

# Development
uv run uvicorn app.main:app --reload --port 8000    # Start dev server
uv run pytest                                         # Run all tests
uv run ruff check .                                  # Lint code

# Before commit
uv run pytest && uv run ruff check .                 # Full validation
```

### Key Files & Directories

| Path | Purpose |
|------|---------|
| `app/main.py` | FastAPI routes and HTMX endpoints |
| `app/game_service.py` | `GameSession` class for state management |
| `app/game_logic.py` | Pure game logic (board generation, bingo detection) |
| `app/models.py` | Pydantic models for data validation |
| `app/data.py` | Question bank for bingo squares |
| `app/templates/` | Jinja2 HTML templates |
| `app/static/css/app.css` | Custom CSS utility classes |
| `tests/` | pytest test files |

---

## Architecture & Design Patterns

### State Management

**GameSession** (in `app/game_service.py`) is the single source of truth for game state:

```python
@dataclass
class GameSession:
    game_state: GameState           # START, PLAYING, or BINGO
    board: list[BingoSquareData]    # 25 squares (5x5 grid)
    winning_line: BingoLine | None  # Winning row/column/diagonal
    show_bingo_modal: bool          # UI state for modal
```

Sessions are persisted server-side in an in-memory dict keyed by `session_id` (stored in browser cookies). Immutability is enforced via Pydantic's `frozen=True` for models.

**Key principle:** Routes never modify session state directly. Instead, they call methods like `session.start_game()`, `session.handle_square_click()`, etc. This keeps game logic centralized.

### Game Flow

1. **START** → User clicks "Play" button
2. **PLAYING** → User marks squares; check for bingo after each click
3. **BINGO** → 5-in-a-row detected; show modal; user dismisses to go back to START

### Pure Game Logic

All board generation and bingo detection lives in `app/game_logic.py`. These are pure, testable functions:

- `generate_board()` → Creates a new shuffled 5x5 board with center as free space
- `toggle_square(board, id)` → Toggles a square's marked state, returns new board
- `check_bingo(board)` → Returns winning `BingoLine` if bingo detected, else None
- `_get_winning_lines()` → Cached tuple of all 12 possible winning lines (5 rows + 5 cols + 2 diagonals)

Benefit: These functions are fast, cacheable, and testable without HTTP requests.

### Templates & HTMX

Templates use Jinja2 and HTMX for dynamic updates without page reloads:

- `templates/base.html` — Layout wrapper
- `templates/home.html` — Entry point
- `templates/components/` — Reusable partial templates (game_screen, bingo_board, etc.)

HTMX attributes send POST requests to endpoints; responses return HTML snippets that replace page regions.

---

## Code Conventions

### Naming & Style

- **Functions:** snake_case, verb-forward (`generate_board`, `check_bingo`)
- **Classes:** PascalCase (`GameSession`, `BingoSquareData`)
- **Type hints:** Required on all function signatures (enforced by linter)
- **Imports:** `from module import name` (explicit over wildcard)
- **Line length:** 88 characters (ruff/black default)

### Linting & Testing

All code must pass before commit:

```bash
uv run ruff check .    # Check E (errors), F (undefined), I (imports), N (naming), W (warnings)
uv run pytest          # All tests must pass (testpaths: tests/)
```

### Test Structure

Tests live in `tests/` and follow two patterns:

1. **API tests** (`test_api.py`) — httpx TestClient against FastAPI routes
2. **Logic tests** (`test_game_logic.py`) — Direct function calls to game_logic module

Example:

```python
# tests/test_game_logic.py
def test_generate_board_has_25_squares():
    board = generate_board()
    assert len(board) == 25
    assert sum(1 for sq in board if sq.is_free_space) == 1  # Center only

# tests/test_api.py
def test_start_game_returns_html():
    response = client.post("/start")
    assert response.status_code == 200
    assert "bingo" in response.text.lower()
```

---

## Common Workflows

### Adding a New Route

1. Add the route handler to `app/main.py`
2. If it modifies game state, call `session.` method (e.g., `session.start_game()`)
3. Return `TemplateResponse` with the appropriate component
4. Add test to `tests/test_api.py`

Example:

```python
@app.post("/my-endpoint", response_class=HTMLResponse)
async def my_endpoint(request: Request) -> Response:
    session = _get_game_session(request)
    # Modify session state via method call
    session.my_game_action()
    # Return rendered component
    return templates.TemplateResponse(
        request,
        "components/my_template.html",
        {"session": session}
    )
```

### Adding a New Game Feature

1. Add logic function to `app/game_logic.py` (pure, testable)
2. Call the function from `GameSession` method in `app/game_service.py`
3. Add endpoint in `app/main.py` that calls the session method
4. Add Jinja2 template in `app/templates/components/`
5. Add tests: logic tests in `test_game_logic.py`, integration tests in `test_api.py`

### Updating CSS

CSS utility classes live in `app/static/css/app.css`. Use existing utilities for consistency:

- **Flexbox:** `.flex`, `.flex-col`, `.gap-2`, `.items-center`, `.justify-between`
- **Grid:** `.grid`, `.grid-cols-5`, `.gap-1`
- **Spacing:** `.p-4`, `.m-2`, `.mb-4`, `.mx-auto`
- **Colors:** `.bg-primary`, `.text-gray-700`, `.bg-accent`

See [css-utilities.instructions.md](instructions/css-utilities.instructions.md) for the full utility reference.

---

## Development Checklist

Before committing changes:

- [ ] Code follows Python conventions (snake_case, type hints)
- [ ] No unused variables, imports, or dead code
- [ ] `uv run ruff check .` passes with no errors
- [ ] `uv run pytest` passes all tests (aim for >80% coverage)
- [ ] Routes use `_get_game_session()` to access session state
- [ ] New features include tests
- [ ] Templates use existing CSS utilities or are marked for `frontend-design` review

---

## Frontend Development

For design-first frontend work (new themes, redesigns, component polishing), use the **frontend-design** instruction. This guides creative, distinctive UI work that avoids generic "AI slop" aesthetics.

See [frontend-design.instructions.md](instructions/frontend-design.instructions.md) for full guidance.

---

## Workshop Context

This project is part of a GitHub Copilot Agent Lab where learners practice:

- **Part 01:** Context Engineering (setting up instructions like this!)
- **Part 02:** Design-First Frontend (using Copilot to redesign the UI)
- **Part 03:** Custom Quiz Master (creating quiz themes with custom agents)
- **Part 04:** Multi-Agent Development (building features with TDD agents)

See [workshop/GUIDE.md](../workshop/GUIDE.md) for full lab details.

---

## Running the App Locally

```bash
# Terminal 1: Start the dev server (with hot reload)
uv run uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Terminal 2: Run tests in watch mode (if available)
uv run pytest --watch

# Browser: Open http://localhost:8000
```

Server runs on port 8000 and watches for code changes. HTMX requires a full browser (not VS Code Simple Browser).

---

## Questions or Issues?

- **Linting issues?** Run `uv run ruff check .` for full lint report
- **Test failures?** Run `uv run pytest -v` for detailed output
- **State issues?** Check `GameSession` logic in `app/game_service.py`
- **Template issues?** Verify Jinja2 syntax and HTMX attributes
