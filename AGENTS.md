# Example repository rules

- Build against installed IBAMR with `find_package(IBAMR CONFIG REQUIRED)` and imported targets. Never require its source checkout or internal `IBAMR_ADD_EXAMPLE` macro.
- Setup instructions set `IBAMR_ROOT` directly to the installation prefix. Support installations made with autoibamr or built and installed separately; do not require an installer-specific environment script.
- Application source, inputs, and geometry live here. Compiled programs and all run files live under the example repository's build directory.
- No fallback code. Missing dependencies, unsupported required features, or failed runs must stop visibly.
- Add only examples used by written lessons. Keep candidates and preparation for future lessons outside this repository. Do not include completed solution applications or a solutions directory.
- Preserve upstream notices and COPYRIGHT. Record the IBAMR version, source link, and tutorial changes in UPSTREAM.json and the relevant example notes. Refer to the version in learner text instead of displaying commit or file hashes.
- Follow the guide's plain-language style. Verify exact targets, commands, data files, and run output before claiming a lesson works.
- Do not commit generated build files or simulation output. Do not publish or push without the user's instruction.

- Learner lessons live in `tutorial/`, with figures under `tutorial/figures/`. Keep the root README as the overview and entry point.
- Each “Before moving on” section first summarizes the lesson and what the reader should have, then introduces the next tutorial in a separate one- or two-sentence paragraph.
