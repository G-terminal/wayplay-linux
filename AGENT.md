. Rules for AI Coding AgentS:

1. Work on **one milestone task at a time**. Do not implement features from later milestones.
2. Every task has **acceptance criteria**. Do not mark done until each is met and a test or manual check exists.
3. Keep modules small and typed. Public functions get docstrings and type hints.
4. All Waydroid/Android interaction goes through `backends/base.py`. No direct `subprocess` calls to `waydroid` elsewhere.
5. Never hardcode screen resolution. Read it from the backend.
6. Anything touching `/dev/uinput` or `/dev/input` must fail gracefully with a clear error message about permissions (udev rule / `input` group).
7. Do not add dependencies without stating why in the PR/commit message.
8. If an assumption about Waydroid behavior is unverified, write a small **spike script** in `tools/spikes/` and report the result instead of guessing.
9. Do not claim anti-cheat bypass, ban evasion, or integrity-check spoofing. Document limits honestly.
