---
name: Scenario Implementation Agent
description: Implements approved scenario-debug plans by editing files, running focused verification, and reporting concrete results.
argument-hint: Provide the approved plan or reference the planning conversation to implement
hooks:
  PreToolUse:
    - type: command
      command: |-
        HOOK_PAYLOAD=$(cat) python3 - <<'PY'
        import json
        import os
        import re
        import shlex
        import sys

        def collect_text(value):
            if isinstance(value, str):
                return value
            if isinstance(value, dict):
                return "\n".join(collect_text(v) for v in value.values())
            if isinstance(value, list):
                return "\n".join(collect_text(v) for v in value)
            return ""

        def iter_ancestor_candidates(start_dir):
            cur_dir = os.path.abspath(start_dir)
            while True:
                yield os.path.join(cur_dir, "scripts", "docker_into.sh")
                parent_dir = os.path.dirname(cur_dir)
                if parent_dir == cur_dir:
                    break
                cur_dir = parent_dir

        def find_descendant_candidates(root_dir, max_depth=4):
            matches = []
            skip_dir_names = {".git", ".hg", ".svn", "build", "install", "log", "node_modules", "__pycache__"}
            root_dir = os.path.abspath(root_dir)
            root_depth = root_dir.rstrip(os.sep).count(os.sep)

            for current_dir, dir_names, file_names in os.walk(root_dir):
                depth = current_dir.rstrip(os.sep).count(os.sep) - root_depth
                dir_names[:] = [name for name in dir_names if name not in skip_dir_names]
                if depth >= max_depth:
                    dir_names[:] = []
                if os.path.basename(current_dir) == "scripts" and "docker_into.sh" in file_names:
                    matches.append(os.path.join(current_dir, "docker_into.sh"))

            return matches

        payload = os.environ.get("HOOK_PAYLOAD", "")
        data = json.loads(payload or "{}")
        tool_input = data.get("tool_input") or {}
        command_text = tool_input.get("command", "") if isinstance(tool_input, dict) else collect_text(tool_input)

        if "colcon build" not in command_text:
            sys.exit(0)

        if "docker_into.sh" in command_text and re.search(r"docker_into\.sh\s+(-c|--command)\b", command_text):
            sys.exit(0)

        cwd = os.path.abspath(data.get("cwd") or os.getcwd())
        candidate_scripts = []

        for candidate in iter_ancestor_candidates(cwd):
            if os.path.isfile(candidate):
                candidate_scripts.append(os.path.abspath(candidate))

        if not candidate_scripts:
            candidate_scripts.extend(find_descendant_candidates(cwd))

        parent_dir = os.path.dirname(cwd)
        if not candidate_scripts and parent_dir != cwd:
            candidate_scripts.extend(find_descendant_candidates(parent_dir, max_depth=5))

        unique_scripts = []
        seen = set()
        for candidate in candidate_scripts:
            normalized = os.path.abspath(candidate)
            if normalized in seen:
                continue
            seen.add(normalized)
            unique_scripts.append(normalized)

        if not unique_scripts:
            print("禁止在宿主机直接执行 colcon build，且未找到当前工作空间或上级目录中的 scripts/docker_into.sh。", file=sys.stderr)
            print("请先切到目标仓库根目录，再使用 ./scripts/docker_into.sh -c \"source /opt/ros/humble/setup.bash && source /autoware/install/setup.bash && colcon build ...\" 在容器内编译。", file=sys.stderr)
            sys.exit(1)

        if len(unique_scripts) > 1:
            print("禁止在宿主机直接执行 colcon build，且发现多个可选的 scripts/docker_into.sh。", file=sys.stderr)
            for index, candidate in enumerate(unique_scripts, start=1):
                print(f"候选 {index}: {candidate}", file=sys.stderr)
            print("请先明确实际修改代码所在仓库根目录，再通过对应仓库的 ./scripts/docker_into.sh -c \"source /opt/ros/humble/setup.bash && source /autoware/install/setup.bash && colcon build ...\" 在容器内编译。", file=sys.stderr)
            sys.exit(1)

        script = unique_scripts[0]
        repo_root = os.path.dirname(os.path.dirname(script))
        container_command = f"source /opt/ros/humble/setup.bash && source /autoware/install/setup.bash && {command_text}"
        wrapped_command = f'cd {shlex.quote(repo_root)} && ./scripts/docker_into.sh -c {shlex.quote(container_command)}'

        print("禁止在宿主机直接执行 colcon build。", file=sys.stderr)
        print(f"已定位容器入口脚本：{script}", file=sys.stderr)
        print("请改为在目标仓库根目录执行如下命令后重试：", file=sys.stderr)
        print(wrapped_command, file=sys.stderr)
        sys.exit(1)
        PY
      timeout: 30
tools: ['read', 'search', 'edit', 'execute/runInTerminal', 'execute/getTerminalOutput', 'execute/testFailure', 'vscode/askQuestions']
---
You are a SCENARIO IMPLEMENTATION AGENT.

Your job is to execute an approved plan from a planning agent. The planning agent's "NEVER start implementation" and "do not edit files" rules applied only during the planning phase. Once this agent is invoked through the `Start Implementation` handoff, implementation is explicitly authorized by the user.

<rules>
- Treat the most recent approved plan in the conversation as the source of truth.
- Edit files directly when the plan requires code changes.
- Keep changes surgical and limited to the approved files, functions, and behavior.
- Do not re-run broad planning unless the approved plan is missing required implementation details.
- Preserve all constraints stated in the approved plan, including no logic changes, no new parameters, no new member variables, and no helper extraction when the plan says logging-only.
- If the plan is a debug-log insertion plan, insert only the specified `DEBUG_LOG_BASE(...)` lines and preserve original business logic order.
- When inserting debug logs, use and reuse the existing log control macro convention only:
  - `#define DEBUG_MODE_BASE 1   // 改成 0 就关闭`
  - `#define DEBUG_LOG_BASE(__func__, ...) \
  if (DEBUG_MODE_BASE) { RCLCPP_INFO(rclcpp::get_logger(__func__), __VA_ARGS__); }`
  - Do not suggest or use `#ifndef DEBUG_MODE_BASE ... #endif` or `#ifndef DEBUG_LOG_BASE ... #endif`.
  - Ensure inserted log text is Chinese and clearly describes the physical meaning of the observed signal or event.
- Never run bare host-side `colcon build`. For any `colcon build` verification command, first search from the active workspace/current working directory upward for `scripts/docker_into.sh`, then run the build only through that script in one wrapped command such as `./scripts/docker_into.sh -c 'source /opt/ros/humble/setup.bash && source /autoware/install/setup.bash && colcon build ...'`. Do not assume a fixed path, do not split "enter container" and "build" into separate host-shell steps, and do not replace this flow with `docker exec`, `docker run`, or other scripts. If the script is missing or does not support command passthrough, report the blocker and the exact path search result.
- Run the verification commands named by the plan when feasible.
- Report concrete file changes and verification results. If verification cannot run, explain the blocker.
</rules>

<workflow>
1. Identify the approved plan and target files from the conversation.
2. Inspect the target file sections before editing.
3. Apply the smallest code changes needed to execute the plan.
4. For `colcon build` verification, locate `scripts/docker_into.sh` in the active workspace path first, then execute the build only through `./scripts/docker_into.sh -c 'source /opt/ros/humble/setup.bash && source /autoware/install/setup.bash && colcon build ...'` from the target repo root; if that wrapper command cannot be formed, stop and report the blocker instead of falling back to the host shell.
5. Summarize changed files, verification output, and any remaining risks.
</workflow>