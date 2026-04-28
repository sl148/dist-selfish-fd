dist-selfish-fd
===============

MA-FD code, with the option of using selfish agents. Multi-agent A\* on
top of Fast Downward, where each agent runs its own search process and
the agents coordinate over UDP sockets.

This README covers building and running the planner on a modern 64-bit
Linux toolchain (Ubuntu 22.04, gcc 11, Python 3.10). The original code
targeted 32-bit gcc 4.x and Python 2; the changes required to bring it
up on a current system are documented at the bottom.


Requirements
------------

- gcc / g++ (11.x tested)
- GNU make
- Python 3 (3.10 tested)
- `awk` (mawk or gawk; the helper scripts work with either)
- `pthread` and `librt` development headers (already present in
  most Linux distributions)
- Optional: `flex` and `bison` if you want to build the `VAL` plan
  validator (skipped automatically if `src/VAL` is absent)


Quick start (rovers / p12, 4 agents, optimal A\*)
------------------------------------------------

From the repo root:

    cd src
    ./build_all
    python3 translate/translate.py ../benchmarks/rovers/domain.pddl \
                                   ../benchmarks/rovers/p12.pddl
    ./preprocess/preprocess < output.sas

This produces:

- `output.sas` — translator output (PDDL → SAS+)
- `output`     — preprocessor output, fed into the search

The `p12` rovers problem has 4 agents named `rover0..rover3`. Configure
the planner by writing two files in `src/`:

`agents` (count on the first line, one agent name per line after):

    4
    rover0
    rover1
    rover2
    rover3

The names are important: they determine how operators are distributed
across agents. An operator is owned by agent *i* if its name contains
the *i*-th line's string.

`comm` (one `host:port` per agent, in the same order as `agents`):

    127.0.0.1:3010
    127.0.0.1:3011
    127.0.0.1:3012
    127.0.0.1:3013

Then run the `4_agents` script, which launches one search process per
agent and parses the per-agent logs:

    bash 4_agents

The script is just:

    ./search/downward --search "astar(lmcut())" --agents 0 < output > 0 &
    ./search/downward --search "astar(lmcut())" --agents 1 < output > 1 &
    ./search/downward --search "astar(lmcut())" --agents 2 < output > 2 &
    ./search/downward --search "astar(lmcut())" --agents 3 < output > 3
    python3 parse_results.py 4

Each agent writes its log to a file named after its agent index
(`0`, `1`, `2`, `3`). On completion the solving agent prints
`SOLVED!! Solution cost is: <cost>` and `parse_results.py` aggregates
expansions, messages, search times, and the plan cost across all
agents.

Expected output (rovers/p12, lm-cut, 4 agents) looks like:

    Expanded Generated messages max_time sum_time cost init_h
    1182221 5908376 698375 17.92 70.71 19 0


Running other configurations
----------------------------

- **Different agent counts.** The repo includes pre-baked launcher
  scripts `2_agents`, `3_agents`, ..., `40_agents`, plus
  `10_agents_opt`. Edit `agents` and `comm` to match the agent count
  you choose, then run the corresponding script. Remember that the
  agent count, the lines in `agents`, and the lines in `comm` must
  agree.
- **Suboptimal / different heuristics.** See `script` for the full
  set of flags the search binary accepts (e.g. `--parallel`,
  `--marginal 2`, `--multiple_goal`, alternative heuristics like
  `astar(ff())`, `lazy_greedy([ff()])`, etc.).
- **Selfish agents.** Pass the appropriate flags to `downward`
  (see `script` for the menu) — selfish behavior changes how an
  agent values shared vs. private goals during search.

To switch to a different problem (e.g. `rovers/p15` with 4 agents),
re-run translate + preprocess on the new PDDL pair, leave `agents` /
`comm` alone if the agent set is unchanged, and re-run the launcher
script. If the agent set changes, update both files first.


Layout
------

    benchmarks/         IPC-style PDDL benchmarks grouped by domain
    scaling_results/    historical experiment outputs
    src/
      translate/        PDDL → SAS+ translator (Python)
      preprocess/       SAS+ preprocessor (C++)
      search/           Fast Downward search engine + MA additions
      VAL/              optional plan validator (built only if present)
      agents, comm      runtime config for an MA run
      N_agents          launcher scripts for N ∈ {2..40}
      parse_results.py  aggregates per-agent logs after a run


Files produced at runtime
-------------------------

- `output.sas` — translator output (in `src/`)
- `output`     — preprocessor output (in `src/`)
- `0`, `1`, ... `N-1` — per-agent search logs
- `elapsed.time` — wall-clock metadata from the `downward` wrapper
- `all.groups`, `test.groups`, `output.sas` are translator artifacts


Modernization notes
-------------------

The original code targeted 32-bit gcc 4.x and Python 2. The following
changes were made so it builds and runs on a modern 64-bit toolchain:

- `src/preprocess/Makefile` and `src/search/Makefile`
  - Removed `-m32` from `CCOPT` and `LINKOPT` (build 64-bit native).
  - Disabled `-static -static-libgcc` in the release link step
    (static `libstdc++` / `libc` archives are typically not present
    on modern distros).
  - Replaced `-ansi -pedantic -Werror` with `-Wno-error` so warnings
    in third-party headers don't fail the build.
  - Added `-std=gnu++03` (and `-Wno-write-strings`) to the search
    Makefile so dynamic exception specifications in
    `practical_socket.h` (`throw(SocketException)`) compile under
    g++ 11. With `-std=gnu++03` the local `tuple` typedef in
    `hm_heuristic.h` no longer collides with `std::tuple` either,
    but the typedef has been renamed to `htuple` for safety.
- `src/search/hm_heuristic.{h,cc}` — `typedef ... tuple` renamed to
  `htuple` (avoids ambiguity with `std::tuple`).
- `src/translate/invariant_finder.py` — `time.clock()` →
  `time.process_time()` (Python 3 removed `time.clock`).
- `src/parse_results.py` — Python 3 `print()` syntax; shebang
  switched to `/usr/bin/env python3`.
- `src/4_agents` — invokes `python3 parse_results.py 4` instead of
  `python parse_results.py 4`.
- `src/search/dispatch` and `src/search/unitcost` — call `awk`
  rather than `gawk` (the embedded scripts use no gawk-specific
  features, and many systems ship only mawk by default).

These are minimal changes; nothing about the planner's logic or
output format was touched.
