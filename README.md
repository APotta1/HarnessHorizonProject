# HarnessHorizonProject

## 1. Why am I doing this?

- **Who it's for**
  - A learning project for me
  - Anyone who builds with an LLM agent and wants their first working code to stay the way it is when they ask for more

- **The problem**
  - The first prompt gives working code with the expected outcomes
  - The next prompt (for example, "now add due dates") often makes the LLM rewrite that first code too, even though nobody asked for it
  - Because regeneration is random, the first code's outputs can come out different, so the expected outcomes no longer happen
  - Example: `list` printed `1. Buy milk`, and after adding due dates it prints `[1] Buy milk (no date)`

- **What the harness does**
  - Freezes the first implementation once it is done
  - Locks the first code: any edit to it is rejected (checked with file hashes)
  - Locks the expected outcomes: saves what the first code produces as tests and reruns them after every new prompt
  - Lets the agent **add** new code, but not rewrite existing code
  - If a new feature really needs old code to change, asks the user to approve the change first

- **What I want to show at the end**
  - Same app, 10 add-a-feature prompts, with and without the harness
  - Without the harness: the first code and its outputs drift more with each prompt
  - With the harness: no changes to the first code, and the original outcomes stay the same

- **What I want to learn**
  - How to build an agent loop with tools
  - How to make a harness enforce rules instead of trusting the prompt
  - How to measure agent reliability with evals

## 2. Is the problem real?

> These are predictions, not results yet. A small test run will confirm or disprove them.

- **Does a plain LLM change the first code when asked to add something?**
  - Probably yes, but it depends on how the LLM edits code
  - Chat style (it rewrites the whole file and you paste it in): changes are very common, since every reply regenerates everything
  - Tool style (Claude Code, Cursor, edits applied as small diffs): changes are less common, but still happen when a new feature touches shared code
  - Comparing the two styles may be the most interesting finding

- **After how many prompts does it start?**
  - Chat style: likely within the first 1 to 3 prompts
  - Tool style: later, mostly when a feature touches the same code as an earlier one (like storage or output formatting)
  - Drift should grow with each new prompt

- **What kind of changes does it make?** (least to most harmful)
  - Formatting and naming changes (renamed variables, reordered functions)
  - "Improvements" nobody asked for (extra error handling, refactors)
  - Output format changes (`1. Buy milk` becomes `[1] Buy milk`)
  - Data structure changes (new fields, a different storage format) that can break earlier features
  - Accidentally removing or simplifying earlier behavior

- **Do the changes break expected outcomes, or is the code just written differently?**
  - Both happen; most changes are probably harmless rewrites with the same outputs
  - Output format and data structure changes are the ones that break things
  - Track both, but count **broken outcomes** as the main failure and **code changed, same outputs** separately

- **Why does it happen?**
  - The LLM has no concept of "approved" or "finished" code; all code in its context looks editable
  - Whole-file regeneration is random, so the same request gives slightly different code each time
  - Models are trained to be helpful, so they "improve" things nearby
  - In a new session, the LLM has forgotten why the first code was written that way
  - Some features genuinely need old code to change (for example, due dates need a new field)

- **How I'll check**
  - Build the to-do app with a plain LLM, then ask for 5 to 10 additions one at a time
  - After each prompt, check whether the first code changed and whether the original outputs still match
  - Do this once in chat style and once in tool style

## 3. What counts as a requirement?

- **What counts as "the first code"?**
  - The code produced by the first prompt

- **When does code become frozen?**
  - The first part of the code is frozen
  - This matters more as prompts pile up, because more prompts cause more changes

- **Where do the expected outcomes come from?**
  - Throughout the code: what each function in the first code returns and outputs
  - The harness records these automatically after the first prompt, instead of me writing them by hand

- **Is every expected outcome written down as a test?**
  - Yes: each recorded outcome becomes a saved test
  - The saved tests rerun after every new prompt, and any different result counts as a change

- **What counts as "changing" the first code?**
  - A function changes (its body, its name, or its inputs)
  - A return statement changes
  - What the code does changes (its outputs or behavior)
  - A saved test is deleted or weakened
  - Does NOT count: whitespace, formatting, or comment changes

- **Can new code call or wrap the old code?**
  - Yes: new code can call old functions or build on top of them
  - No: new code can't change what an old function returns or does

- **What happens when a new feature truly needs old code to change?**
  - The harness pauses and shows me the proposed change
  - The change is applied only if I approve it
  - After approval, the saved tests are updated to the new expected outcome

## 4. How will I know if the harness works?

- **What am I comparing against?**
  - A plain LLM making the same additions without the harness
  - Same model, same prompts, same starting app; the only difference is the harness

- **Is the comparison fair?**
  - Both get the same prompts, the same tools, and the same number of tries
  - The plain LLM gets a good prompt too (for example, "don't change existing code"), so the harness is compared against a reasonable baseline, not one set up to fail

- **What do I measure after each prompt?**
  - **First code changed:** how many first-code functions changed
  - **Outcomes broken:** how many saved expected outcomes no longer match
  - **Feature works:** whether the new feature actually works (its own test passes)
  - **Blocked edits:** how many times the harness stopped the agent from changing first code

- **What result would prove me wrong?**
  - The plain LLM changes the first code just as rarely as the harness does
  - Or the harness keeps the first code safe, but new features stop working

- **How many times do I run it?**
  - LLM results vary from run to run, so one run can be luck
  - Start with 3 runs of each setup (plain LLM vs. harness) and report the average and the spread
  - Add more runs if the results are close

- **Does the harness make the agent worse at adding features?**
  - Freezing everything would score perfectly on "no changes" but be useless
  - So success means both: the first code stays the same **and** new features still get built
  - Track features completed alongside changes blocked

- **What does it cost?**
  - Record time and API cost per run for both setups
  - The harness will cost more (rerunning tests, retries), so report whether that extra cost is worth it

- **What counts as success?**
  - With the harness: zero broken expected outcomes, and nearly as many working features as the plain LLM
  - Without the harness: broken outcomes go up as more prompts are added

## 5. Scope and constraints

- **Cost: the harness is free; the AI tool may not be**
  - The harness is an **MCP server** (a connector), so it doesn't need its own LLM or API key
  - The user's own AI tool (Claude Code, Claude Desktop, Cursor, etc.) supplies the model; the harness only checks and protects the code
  - **Free:**
    - Building it: Python, the MCP SDK, git, and pytest are all free and open source
    - Running it: it runs locally on the user's machine, with no hosting and no API calls of its own
    - Sharing it: anyone can use it from GitHub
  - **Possibly not free:**
    - The AI tool it connects to: most need a paid plan or API credits, but users who already have one pay nothing extra for the harness
  - **Testing it for free:**
    - A subscription I already have: no extra cost, but runs count against usage limits
    - A local model through Ollama: completely free, but needs a decent computer and the models are weaker (which may make drift easier to show)
    - A provider's free tier: rate-limited, but fine for a small test

- **How people use it**
  - Add it as an MCP connector to their AI tool
  - Freeze the first code with one tool call, then keep prompting as usual
  - The harness checks every change against the frozen code and saved outcomes

- **MCP tools it provides** (first draft)
  - `freeze`: lock the current code and record its expected outcomes
  - `check`: compare the code now against the frozen version and rerun the saved outcomes
  - `request_change`: ask the user to approve a change to frozen code
  - `report`: show what's frozen, what changed, and what was blocked

- **Seeing that nothing changed**
  - A visual report (for example, an HTML page) that lists every frozen function
  - Each function shows ✅ unchanged or ❌ changed, plus whether its expected outcome still matches
  - A list of the changes the agent tried to make to frozen code, and which ones were blocked or approved

- **Known limit**
  - An MCP server can't stop the AI tool from editing files with its own built-in tools
  - So blocking needs a second piece: a hook (for example, a Claude Code hook) or a git pre-commit check that calls the harness before a change is kept
  - Without that, the harness can still **detect** and **report** changes, just not block them

- **Smallest version that proves the idea**
  - Python only
  - One to-do app, 5 to 10 add-a-feature prompts
  - Plain AI tool vs. the same tool with the harness, 3 runs each

- **Time:** to decide

- **Not doing (for now)**
  - Languages other than Python
  - Huge codebases
  - Training or fine-tuning a model
  - Paid hosting; it runs locally on the user's machine

## 6. Design decisions

- **Detect or block?**
  - Detect: after each prompt, the harness checks what changed and produces a report
  - Blocking (with a hook or git pre-commit check) can be added later; detection comes first

- **What "freeze" records**
  - **Per function, not per file:** a hash of each function's code, so the report can say exactly *which* function changed, and new code added to the same file doesn't count as a change
  - **Expected outcomes:** for each frozen function, the harness runs it with sample inputs and saves the results; if the project already has tests, it runs those too and saves which ones pass
  - **A snapshot of the code:** a copy of the frozen code, so the report can show the exact before-and-after difference
  - **When and what:** the time of the freeze and the list of frozen files and functions

- **Where it's stored**
  - A `.harness/` folder in the user's project, holding the hashes, saved outcomes, code snapshot, and a change log
  - Committed to git, so the frozen state has its own history

- **Approving a change**
  - Always asked to the user, never decided by the agent
  - The harness shows the change (before vs. after) and its effect on the expected outcomes
  - If approved: the frozen record updates to the new version
  - If rejected: the change is flagged so the user can undo it (for example, by restoring the snapshot)

- **The agent reports changes**
  - After each prompt, the agent calls `check` and tells the user plainly whether any frozen code changed
  - The agent can't hide a change, because the harness checks the files itself rather than trusting what the agent says

- **Unfreezing**
  - Available as an option: frozen code can be unfrozen, either one function or everything
  - The user confirms every unfreeze; the agent can suggest it but can't do it alone
  - Every unfreeze is recorded in the change log and shown in the report

- **What the agent can do alone vs. what needs the user**

  | Action | Agent can do it alone? | Why |
  |---|---|---|
  | Generate a report (`check` / `report`) | ✅ Yes | It only reads and shows information, so the agent should run it after every prompt |
  | Unfreeze code | ❌ No, it can ask, but the user says yes or no | Unfreezing removes the protection; if the agent could do it alone, it could unfreeze anything in its way and the harness would protect nothing |

- **Example: adding due dates**
  1. I prompt: "Add due dates."
  2. The agent realizes it needs to change the frozen `save_task` function
  3. The agent asks: "`save_task` is frozen. Adding due dates requires changing it. Can I unfreeze it?"
  4. I say yes or no
  5. Either way, the report shows what happened

- **The visual report**
  - A static HTML page, regenerated each time `check` or `report` runs
  - Shows every frozen function with ✅ unchanged or ❌ changed, whether its outcomes still match, the before-and-after code for anything changed, and the history of approvals and unfreezes

- **New prompts in new sessions**
  - At the start of each session, the agent can ask the harness what's frozen, so it knows which code to leave alone
  - Even if the agent forgets, the check after the prompt still catches any change

## 7. What if it doesn't work?

- **If the plain AI tool rarely changes the first code (plan B)**
  - Make the task harder: features that touch shared code (like storage or output format), more prompts (15 to 20), and new sessions between prompts so the AI forgets earlier decisions
  - Try a weaker or local model, which tends to drift more
  - Try chat style (whole-file rewrites), where drift is more likely
  - Even if drift is rare, the harness is still useful as a safety check and a visible report that nothing changed; the write-up just says the risk is smaller than expected

- **If the harness misses real changes**
  - Collect each missed case as a test for the harness itself
  - Common causes to check: changes outside functions (global variables, imports, config files), or behavior that the saved sample inputs don't cover
  - Fix by tracking more than functions, or by saving more sample inputs

- **If the harness flags harmless changes (false alarms)**
  - Check whether the "change" is really whitespace, formatting, or comments, which shouldn't count
  - Compare code structure instead of raw text, so formatting differences are ignored
  - Track false alarms as their own number in the results

- **If the harness gets in the way (too many approval requests)**
  - Count how often the agent has to ask to unfreeze
  - If it's constant, the freeze may be too strict (for example, freezing helper functions that are meant to grow)
  - Option: let the user freeze only the functions that matter most

- **Is a "no effect" result still worth writing up?**
  - Yes: "a plain AI tool already keeps the first code stable in these conditions" is a real finding
  - The harness, the report, and the test setup still show the skills I wanted to learn

- **When do I stop?**
  - Once the smallest version works end to end: freeze, prompt, check, and report on one to-do app
  - And the before/after test has run 3 times each, with or without a clear result
  - Then write up what I have; extra features go on a "next steps" list instead of delaying the write-up

## 8. How will I present it?

- **Who it's for**
  - **Users:** developers who want to plug the harness into their AI tool (the main audience for the final README)
  - **Learners and reviewers:** people who want to see how it works and what I learned (served by a separate write-up)

- **The one picture that tells the story**
  - The visual report, with every frozen function showing ✅ unchanged after 10 prompts
  - Plus a chart of changes per prompt, with and without the harness

- **Easy to try**
  - One command to add it as an MCP connector
  - A short example to follow: freeze the to-do app, ask for due dates, and see the report

- **Showing it working**
  - A screenshot of the report at the top of the README
  - A short GIF or 1–2 minute demo video of the due-dates example (the agent asks to unfreeze `save_task`, I approve or reject, and the report updates)

- **Writing about it**
  - A blog or LinkedIn post about the results, including what didn't work
  - Honest numbers from the before/after test, even if the effect is small

- **The final README**
  - These planning notes move to `docs/planning.md`
  - The main README becomes: what it is, the report screenshot, how to install it, how to use it, and what the test showed
