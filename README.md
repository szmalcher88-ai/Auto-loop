# Auto-loop

Generyczny szkielet **pętli promptującej agenta kodującego** — narzędzie do bezobsługowego wykonywania listy zadań przez agenta AI (np. Claude Code), z niezależną weryfikacją, twardymi budżetami i eskalacją do człowieka.

Cała logika mieści się w jednym pliku [loop.py](loop.py) (stdlib-only, Python ≥ 3.9, cross-platform).

## Filozofia (inwarianty)

- **Agent proponuje, pętla weryfikuje i commituje.** Agent nigdy sam nie rozstrzyga, czy jego praca jest dobra (wykonawca ≠ decydent).
- **Każda iteracja = świeży agent, czysty kontekst.** Stan żyje w plikach (`PLAN.md`) i w gicie, nie w oknie kontekstowym agenta.
- **Pętla jest dokładnie tak dobra, jak jej `verify_commands`** (sygnał prawdy). Pętla bez weryfikatorów odmawia startu.
- **Baseline musi być zielony przed startem** — inaczej pętla nie odróżnia własnych szkód od zastanych.
- **`PLAN.md` należy do pętli** — agent, który go dotyka, jest automatycznie eskalowany.

## Jak to działa

1. **Preflight** — sprawdza, że repo jest czyste, weryfikatory przechodzą na nietkniętym kodzie, plan i szablon promptu istnieją, a katalog stanu jest w `.gitignore`.
2. Pętla bierze **pierwsze otwarte zadanie** (`- [ ]`) z `PLAN.md`.
3. Buduje prompt z szablonu i odpala **świeżego agenta** (domyślnie `claude -p`).
4. Po agencie uruchamia **niezależne weryfikatory** (`verify_commands`).
   - Zielono → zadanie oznaczone `[x]`, zmiany **scommitowane przez pętlę**.
   - Czerwono → ogon outputu wraca do agenta jako feedback w kolejnej próbie.
5. **Guardy bezpieczeństwa** (każdy kończy się eskalacją do człowieka):
   - zmiana chronionych ścieżek (`protected_paths`, anty-reward-hacking),
   - scope creep — diff większy niż `max_diff_lines`,
   - brak postępu — identyczna sygnatura porażki w kolejnych próbach,
   - wyczerpane próby (`max_retries_per_task`).
6. **Eskalacja**: praca agenta ląduje na `git stash` (nic nie ginie), zadanie dostaje `[!]`, powstaje raport `ESKALACJA-NNN.md` z instrukcją dla człowieka.
7. Twarde budżety: `max_iterations`, `max_wall_seconds`, timeouty agenta i weryfikatorów.

Każde zdarzenie trafia natychmiast do crash-proof logu JSONL (`.loop/loop_log.jsonl`).

## Szybki start

```bash
# 1. Skopiuj i wypełnij konfigurację pod swój projekt
cp loop.config.example.json loop.config.json

# 2. Przygotuj plan zadań (jedna linia = jedno zadanie = jeden commit)
cp PLAN.example.md PLAN.md

# 3. Uzupełnij sekcję "Kontekst projektu" w PROMPT.template.md
#    (preflight nie wystartuje, dopóki jest tam znacznik WYPEŁNIJ)

# 4. Dodaj katalog stanu do .gitignore
echo ".loop/" >> .gitignore

# 5. Podgląd bez uruchamiania agenta
python loop.py --config loop.config.json --dry-run

# 6. Normalny bieg
python loop.py --config loop.config.json
```

## Pliki

| Plik | Rola |
|---|---|
| [loop.py](loop.py) | Generyczny szkielet pętli — nie wymaga modyfikacji per projekt |
| [loop.config.example.json](loop.config.example.json) | Warstwa projektowa: agent, weryfikatory, budżety, guardy |
| [PLAN.example.md](PLAN.example.md) | Format planu zadań (`[ ]` otwarte, `[x]` zrobione, `[!]` eskalowane) |
| [PROMPT.template.md](PROMPT.template.md) | Szablon promptu z placeholderami `{{TASK}}`, `{{VERIFY_COMMANDS}}`, `{{FEEDBACK}}` |
| [INSTRUKCJE-KOLEJNA-SESJA.md](INSTRUKCJE-KOLEJNA-SESJA.md) | Notatki robocze projektu |

## Najważniejsze pole konfiguracji: `verify_commands`

To **sygnał prawdy projektu**. Przykład:

```json
"verify_commands": [
  ["python", "-m", "compileall", "-q", "."],
  ["ruff", "check", "."],
  ["pytest", "-q", "-x"]
]
```

Kolejność od najszybszego do najwolniejszego — pętla zatrzymuje się na pierwszej porażce. Weryfikatory muszą być deterministyczne: flaky test = szum w sygnale prawdy i losowe eskalacje.

## Bezpieczeństwo

Bieg bezobsługowy wymaga w praktyce flagi omijającej potwierdzenia uprawnień agenta (np. `--dangerously-skip-permissions`). **Uruchamiaj pętlę wyłącznie w środowisku, w którym agent nie ma nic cennego do zepsucia poza tym repo** — kontener, sandbox lub VM.
