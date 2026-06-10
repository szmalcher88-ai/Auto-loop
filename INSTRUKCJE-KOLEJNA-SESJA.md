# Instrukcje dla kolejnej sesji: pętla promptująca agenta kodującego

> Kontekst pochodzenia: „You shouldn't be prompting coding agents anymore. You should
> be designing loops that prompt your agents." (P. Steinberger). Ten pakiet to generyczny
> SZKIELET takiej pętli. Szkielet jest celowo uniwersalny; pętla zaczyna działać dopiero,
> gdy dostanie warstwę projektową — i zebranie tej warstwy jest TWOIM zadaniem w tej sesji.

## 1. Czym to jest (i czym nie jest)

Pętla zastępuje człowieka ręcznie promptującego agenta (np. Claude Code) w trybie:
weź zadanie → zbuduj prompt → uruchom świeżego agenta → zweryfikuj niezależnie →
commituj albo retry z feedbackiem → eskaluj do człowieka, gdy się kręci w miejscu.

Podział ról jest twardy i architektoniczny: **agent proponuje zmiany; pętla weryfikuje
i commituje**. Agent nigdy nie rozstrzyga sam, czy jego praca jest dobra, nie commituje
i nie dotyka planu zadań. To jest zasada „wykonawca ≠ decydent" przeniesiona do kodu.

To NIE jest narzędzie do zadań niejasnych. Pętla wykonuje plan złożony z małych,
jednoznacznych, maszynowo weryfikowalnych zadań. Dekompozycja mglistego celu na taki
plan pozostaje pracą człowieka (lub osobnej sesji), nie tej pętli.

## 2. Inwentarz plików

| Plik | Rola | Generyczny czy projektowy? |
|---|---|---|
| `loop.py` | szkielet pętli (Python, stdlib-only) | generyczny — nie edytuj pod projekt |
| `loop.config.example.json` | konfiguracja | **projektowy** — skopiuj jako `loop.config.json` i wypełnij |
| `PROMPT.template.md` | szablon promptu dla agenta | **projektowy** — sekcja „Kontekst projektu" do wypełnienia |
| `PLAN.example.md` | format planu zadań | **projektowy** — utwórz `PLAN.md` z realnymi zadaniami |
| ten dokument | instrukcja przekazania | — |

Uruchomienie: `python loop.py --config loop.config.json` (wcześniej `--dry-run`).

## 3. Cykl życia jednej iteracji (co dokładnie robi `loop.py`)

1. Bierze pierwsze otwarte zadanie `- [ ]` z `PLAN.md`.
2. Buduje prompt z szablonu: zadanie + kontekst projektu + lista weryfikatorów +
   feedback z poprzedniej próby (przy pierwszej — pusty).
3. Uruchamia **świeżego agenta** (nowy proces, czysty kontekst; prompt przez stdin
   albo plik tymczasowy poza repo, gdy w `agent_command` jest `{prompt_file}`).
4. `git add -A` i guardy na zastanym diffie: chronione ścieżki → eskalacja;
   diff > `max_diff_lines` → eskalacja.
5. Uruchamia `verify_commands` (stop na pierwszej porażce).
6. Zielono → oznacza zadanie `[x]` i commituje wszystko jednym commitem
   (`loop: <zadanie>`). Czerwono → ogon outputu weryfikatora wraca do agenta jako
   feedback w następnej próbie (praca częściowa ZOSTAJE w drzewie między próbami —
   agent naprawia własny stan, nie zaczyna od zera).
7. Identyczna sygnatura porażki drugi raz (hash: komenda+kod+ogon z zamaskowanymi
   cyframi) → eskalacja „brak postępu". Wyczerpane próby → eskalacja.
8. Eskalacja = praca agenta na `git stash` (nic nie ginie), zadanie `[!]` w planie
   (scommitowane), raport `ESKALACJA-NNN.md` w `state_dir` z instrukcją dla człowieka.
   Domyślnie pętla wtedy STAJE (`on_escalation: stop`).
9. Budżety twarde: `max_iterations` (każda próba się liczy), `max_wall_seconds`,
   timeouty na agenta i weryfikatory. Log JSONL crash-proof (flush+fsync po evencie).

Preflight ODMAWIA startu, gdy: brak weryfikatorów, brudne drzewo, czerwony baseline,
niewypełniony szablon (znacznik `WYPEŁNIJ`), `state_dir` poza `.gitignore`, brak repo git.

## 4. Inwarianty — z uzasadnieniem (stosuj z osądem, nie rytualnie)

**Pętla jest dokładnie tak dobra, jak jej weryfikatory.** Wszystko inne to hydraulika.
Verify_commands to sygnał prawdy: jeśli jest dziurawy (testy nie pokrywają zachowania),
pętla będzie sprawnie commitować śmieci; jeśli jest flaky, pętla będzie losowo eskalować
albo — gorzej — losowo przepuszczać. Dlatego pętla bez weryfikatorów odmawia startu:
to nie byłaby pętla, tylko bezobsługowe generowanie zmian.

**Baseline musi być zielony przed startem.** Inaczej pętla nie odróżnia szkód
wyrządzonych przez agenta od zastanych — każdy werdykt czerwony/zielony traci znaczenie.

**Stan żyje w plikach i w gicie, nie w kontekście agenta.** Świeży agent na iterację
bije długą sesję: kontekst nie gnije, a `PLAN.md` + `git log` + feedback niosą wszystko,
co potrzebne. Konsekwencja: każdy diff w drzewie jest jednoznacznie pracą agenta
(stąd wymóg czystego drzewa na starcie).

**Plan należy do pętli.** Agent, który odhacza własne zadania albo edytuje plan,
podmienia sygnał prawdy na autodeklarację — dlatego `plan_file` jest zawsze na liście
chronionych ścieżek i jego dotknięcie eskaluje.

**Guardy bronią przed optymalizacją proxy zamiast celu:** chronione ścieżki (agent
„naprawia" czerwone testy osłabiając testy), limit diffa (scope creep), sygnatura
porażki (palenie budżetu w miejscu). Wspólny mianownik wszystkich trzech: utwardzenie
sygnału prawdy, nie „lepsze promptowanie".

**Eskalacja niczego nie niszczy.** Praca agenta idzie na stash z etykietą, raport
mówi człowiekowi, co obejrzeć i jak przywrócić. Pętla nie robi `reset --hard` nigdy.

## 5. Kontekst projektowy do zebrania — to jest Twoja właściwa praca

Zanim pętla ruszy na realnym repo, zbierz od użytkownika / z repo poniższe.
Kolejność odpowiada ważności; pozycja pierwsza to ~80% wartości.

1. **Sygnał prawdy projektu** → `verify_commands`. Pytania do zadania: jaka komenda
   jednoznacznie mówi „kod jest poprawny"? (testy, typecheck, lint, build). Czy jest
   deterministyczna? Ile trwa? (porządkuj od najszybszej). Czy zostawia artefakty —
   jeśli tak, muszą być w `.gitignore`, inaczej preflight to wyłapie (celowo).
   Jeśli projekt NIE MA sensownego sygnału prawdy — zatrzymaj się: pierwszą inwestycją
   jest suite testowy, nie pętla. Pętla na dziurawym weryfikatorze skaluje błędy.
2. **Definicja „zrobione" + konwencje** → sekcja „Kontekst projektu" w
   `PROMPT.template.md`. 5–15 zdań: architektura w akapicie, styl, struktura katalogów,
   jak pisać testy, czego nie wolno (zależności, moduły zakazane). Zwięzłość ma wagę:
   ten tekst wchodzi do każdego promptu.
3. **Granulacja zadań** → `PLAN.md`. Jedno zadanie = jeden commit, wykonalne bez
   pytań, weryfikowalne, mniejsze niż limit diffa. Test jakości zadania: czy obcy
   kompetentny programista wykonałby je bez dopytywania? Jeśli nie — podziel albo
   doprecyzuj. Zadania zależne od siebie → `on_escalation: stop`; niezależne → `skip`.
4. **Chronione ścieżki** → `protected_paths`. Domyślnie `tests/` (anty-reward-hacking).
   UWAGA na konflikt: jeśli zadania mają legalnie dodawać testy, nie chroń `tests/` —
   zamiast tego recenzuj diffy testów w commitach po biegu. Wybierz świadomie jedno.
5. **Budżety** → realistycznie do projektu: czas jednej kompilacji+testów,
   spodziewana liczba zadań, koszt agenta. Lepiej zacząć ciasno i poluzować.
6. **Komenda agenta** → `agent_command`. Dla Claude Code: `claude -p` (+ flaga
   pomijania potwierdzeń dla biegu bezobsługowego — wtedy pętlę uruchamiaj wyłącznie
   w środowisku, gdzie agent nie ma nic do zepsucia poza tym repo: kontener/VM/worktree).
   Na Windows uruchamiaj pod Git Bash/WSL.
7. **Higiena repo**: `state_dir` (domyślnie `.loop/`) dodany do `.gitignore`;
   dedykowany branch dla biegu pętli (tania izolacja i tani odwrót: `git branch -D`).

## 6. Procedura pierwszego uruchomienia (waliduj instrument, zanim mu zaufasz)

Nie ufaj pętli, której ścieżek porażki nie widziałeś. Kolejno:

1. `--dry-run` — preflight + podgląd dokładnego promptu, który dostałby agent.
   Przeczytaj ten prompt oczami agenta: czy wykonałbyś to zadanie bez zgadywania?
2. **Zadanie kanarkowe pozytywne**: jedno trywialne zadanie o znanym wyniku
   (np. „dodaj funkcję X zwracającą Y" + istniejący test, który to sprawdza).
   Oczekiwane: zielono, jeden commit, `[x]` w planie. To testuje hydraulikę.
3. **Zadanie kanarkowe negatywne**: zadanie celowo niewykonalne albo z weryfikatorem,
   który nie może przejść. Oczekiwane: retry z feedbackiem, potem eskalacja
   „brak-postepu", praca na stashu, `[!]` w planie, czyste drzewo. To testuje, że
   porażka jest obsłużona, a nie zamieciona. Pętla, której ścieżki porażki nie
   przetestowano, ma niezweryfikowany sygnał prawdy o samej sobie.
4. Dopiero potem realny plan, na dedykowanym branchu, z ciasnymi budżetami.
5. Po pierwszym realnym biegu: przejrzyj `loop_log.jsonl` i commity. Szczególnie
   diffy dotykające czegokolwiek blisko testów/konfiguracji — guardy łapią wzorce,
   nie intencje.

## 7. Czego pętla świadomie NIE robi + znane ograniczenia

Nie dekomponuje mglistych celów (to praca człowieka przed pętlą). Nie ocenia
architektury ani jakości designu — tylko to, co mierzą weryfikatory. Nie robi
rollbacku merytorycznego (eskaluje zamiast zgadywać). Nie zarządza zależnościami
między zadaniami ponad to, że wykonuje je po kolei od góry.

Ograniczenia implementacyjne, o których warto wiedzieć: sygnatura „braku postępu"
to heurystyka (hash ogona z zamaskowanymi cyframi) — niedeterministyczne ścieżki
w outputach mogą ją osłabić, wtedy ratuje limit prób; `verify` zatrzymuje się na
pierwszej porażce (szybciej, ale feedback niepełny przy wielu problemach naraz);
zadania są liniami w pliku markdown — duże plany lepiej trzymać krótkie i doładowywać.

## 8. Status weryfikacji szkieletu (co zostało przetestowane, a co nie)

Przetestowane testem dymnym na mockowanych agentach (repo testowe + mock zamiast
prawdziwego agenta): ścieżka sukcesu (2 zadania → 2 commity, `[x]`), eskalacja
brak-postepu (stash + `[!]` + raport + czyste drzewo + exit 5), eskalacja
chroniona-ścieżka, dry-run, odmowy preflightu (czerwony baseline, brudne drzewo,
niewypełniony szablon), oba tryby podawania promptu (stdin i `{prompt_file}`).

NIEprzetestowane (zrób w ramach kroku 6 na realnym środowisku): integracja
z prawdziwym `claude -p` (timeouty, format wyjścia), zachowanie na Windows,
guard `max_diff_lines` (logika ta sama co chronione ścieżki, ale ścieżka nie była
osobno wykonana), `on_escalation: skip`, długie biegi pod budżetem czasu.
Traktuj tę listę jako jawny dług testowy, nie jako gwarancję.
