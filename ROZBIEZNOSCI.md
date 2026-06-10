# Rozbieżności: spec (INSTRUKCJE-KOLEJNA-SESJA.md §3) vs zachowanie loop.py

**Brak rozbieżności.**

Podczas pisania suite'u (tests/test_loop.py, 11 testów) faktyczne zachowanie
`loop.py` we wszystkich sprawdzanych punktach zgadzało się z opisem w sekcji 3
specyfikacji:

- happy path: commit `loop: <zadanie>` per zadanie, `[x]` w planie, exit 0;
- identyczna sygnatura porażki przy drugiej próbie → eskalacja „brak-postepu",
  exit 5, praca na stashu (`loop-eskalacja`), `[!]` w planie, czyste drzewo,
  raport `.loop/ESKALACJA-001.md`;
- guardy chronionej ścieżki i `max_diff_lines` działają na zastanym diffie,
  PRZED weryfikacją — eskalacja w pierwszej próbie, bez retry;
- preflight odmawia startu (exit 2) dla: czerwonego baseline'u, brudnego
  drzewa, pustych `verify_commands`, znacznika `WYPEŁNIJ` w szablonie,
  `state_dir` poza `.gitignore`;
- tryby podawania promptu (stdin i `{prompt_file}`) dają identyczny prompt
  i identyczny rezultat;
- `on_escalation: skip` oznacza zadanie `[!]` i kontynuuje od następnego.

Żaden test nie wymagał oznaczenia `xfail` ani zmiany `loop.py`.
