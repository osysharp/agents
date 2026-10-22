# Osysharp.Agents

Give an AI agent a real job inside your app — and trust it with that job.

An agent here is a class: a purpose, the skills it holds, what it may spend. A skill is a class too: a typed input, a
typed result that is checked before it reaches you, and the tools it may use — which are your app's own functions. The
agent runs as an account of its own with the roles you gave it, and when it works for a person it can never see or do
more than that person could.

```osy
use Osysharp.Agents@1;

/// Grades the fruit that arrives at the packhouse.
[Actor(Roles = [PackhouseRole.Grader])]
public class Inspector : Agent {
  public static Person Principal => new Person { Email = "inspector@packhouse.example", DisplayName = "Inspector" };
  public override string Purpose => "You grade the fruit that arrives at the packhouse, one crate at a time.";
  public override List<SkillBase> Skills => [new GradeCrate()];
}

/// Judges one crate.
public class GradeCrate : Skill<Crate, Grade> {
  public override string Description => "Grade one crate of fruit.";
  public override string Instructions => "Weigh the crate first. A crate under 10 kg is short: flag it and say so.";
  public override List<ActionInfo> Tools => [Actions.Of(Weigh), Actions.Of(FlagCrate)];
  public override string? Check(Grade g) => g.Score < 0 || g.Score > 10 ? "A score is from 0 to 10." : null;
}

app.Agents = new AgentsSetup { Agents = [new Inspector()] };

var outcome = await Agents.Of<Inspector>().Do(new GradeCrate(), crate);          // a Crate in, a checked Grade out
if (outcome.Done && outcome.Result.Short) { TellTheGrower(crate); }
```

## Use case

An app where a model does part of the work: it reviews what was submitted against a written policy, drafts the reply
to a customer, files the documents that arrive, sorts what comes in. The first version is a prompt and a loop. Then
the questions start. What exactly did it do to this record, and why? Who was it acting as when it did? Why did last
month cost four times what this one did? What stops it sending the same thing twice, or reading what the person who
asked could not?

This kit is the answer to those, as the default rather than as work you do later:

- **Typed work.** A skill takes a type and hands back a type. The model is held to the result's own schema, and
  your `Check` sees the value before your code does. A result that does not fit is sent back with the reason.
- **Your functions are its tools.** A tool is an `[Action]` of your app. The compiler has already proved what each one
  reads, changes and sends, and its arguments are described by its own signature — nothing is written twice.
- **Its rights are declared, and they are a ceiling.** The agent is an account with the roles its class names. For
  a person, it reads and changes only what both of them may. A tool its skill does not list never runs, whatever the
  model calls it.
- **A hard limit on every piece of work.** Room is reserved before each call leaves; work whose money runs out is
  stopped, and keeps what it did.
- **A receipt and a record.** Every piece of work leaves one receipt — who asked, as whom it ran, what it cost, how
  it ended — and a record of each step in plain words: each model call with the model that served it and the rule
  that chose it, each tool, each time it asked for help. The record is chained, so a line changed afterwards is found.
- **Help, when it is unsure.** An agent can name a more capable model to consult. The model doing the work may ask it
  one focused question; work that is stuck asks once before it gives up; a decision it is not sure of is asked again.
- **Tests that cost nothing.** [`Osysharp.Agents.Testing`](../agents-testing) scripts what the model answers, turn by
  turn. Everything around it is real.

It has no screens. It is the layer your pages, your processes and other packages build on.

## What it does

| | |
|---|---|
| **Agents** | `[Actor(Roles = […])] public class X : Agent` — a `Purpose`, standing `Instructions`, the `Skills` it holds, its `Limits`, and whom it `Consults`. Listed in `app.Agents`; `Agents.Of<X>()` answers one, ready for work. |
| **Skills** | `Skill<TIn, TOut>` — `Description`, `Instructions`, `Tools`, `Brief(input)` (what the model is shown), `Check(result)` (what a schema cannot say), `Needs` (what the router decides from), `Stop` (turns, circles, how much conversation is kept), `Budget`, `Version`. `TOut` is a class, or `string` for prose. |
| **Giving it work** | `Do(skill, input)` answers an `AgentOutcome<TOut>`: the typed `Result`, the `Status`, the receipt's id, the cost. `Ask(question)` answers in words with the tools of every skill it holds; `Stream(question)` hands the answer on as it arrives. `For(person)` does the work for somebody other than whoever is signed in. |
| **In a process** | A step of a `workflow` is the same call. Nobody is signed in while a process runs, so the agent works with its own roles alone — or `For(person)`. |
| **Tools** | Run as the agent, on behalf of whoever asked: a read finds what both may read, a change needs both to be allowed. Every change is recorded with the agent, the person and the piece of work. |
| **Money** | Each piece of work has a limit: the skill's `Budget`, else the agent's `Limits.PerTask`, else the app's `TaskBudget`. An agent's `Limits.Daily` binds everything it does. Budgets a manager sets with `SetBudget("agent", …)` or `("skill", …)` bind the same calls. All through [`Osysharp.Llm`](../llm). |
| **The receipt** | `AgentTask`: agent, skill and its version, who asked, the account it ran as, status, what it spent and its limit, model calls, tool calls, turns, tokens, how it ended, what it handed back. Extensible: add your own fields with `partial entity AgentTask { … }`. |
| **The record** | `TraceEvent`, per piece of work: `Started`, `ModelCall` (model, router rule, attempts, cost, the meter's id for the call), `ToolCall`, `ToolResult`, `AskedForHelp`, `SentBack`, `Ended` — each with a sentence, its content, and its place in a SHA-256 chain. `VerifyTrace(task)` checks the chain and names the first line that does not fit. |
| **Help** | `Consults = "Strong"` names one of the app's models. `consult` is then one of the model's tools; work that fails `StuckAfter` times in a row consults once; `Decide<T>` re-asks a decision under `ConfidenceFloor`. The consulted model gets the question and the context given with it — no conversation, no tools. Each request is an `AgentHelp` row and a line of the record. |
| **Erasing** | What a record said is personal data, and can go: `EraseAgentWorkContent(task, why)` empties every line's content, the help asked and the result; `AnonymiseAgentWork(task, why)` takes the person off the work and turns every sentence into `Erased.`; `ForgetAgentWork(task)` deletes work whose time is up. `ForgetWhoOversawAgents(person)` takes a person off the standing rules, changes to an agent's place, questions and proposals that name them; `EraseStandingRulesFor(person, why)` takes back the rules for their work. The chain holds hashes of the words, not the words, so `VerifyTrace` still answers intact afterwards, and still finds a line changed later. With [`Osysharp.Privacy`](../privacy) in the app, [`Osysharp.Privacy.Agents`](../privacy-agents) does all of it for a person who asks. |
| **What it does alone** | `May => new Permits { … }`: what it runs by itself, what it PROPOSES to a person first, what it never does — by what a tool does (changes data, sends, spends, reaches outside) and tool by tool. Said nothing, it looks things up alone and proposes the rest. A skill's `May` can only be stricter. |
| **Proposals** | `AgentProposal`: the exact call — tool, arguments, what it does, why, what the agent had looked at. The work waits (`Status == Waiting`). `ApproveAgentProposal` runs it as whoever approves — and as the agent: only what both may — for the version of things that was shown (`VersionOf`); `RejectAgentProposal` gives the agent the reason. `AllowAlone` / `RevokeStandingRule`: "from now on, alone", for whoever `WidensAgentAutonomy` admits. |
| **Asking a person** | `People => new AsksPeople { MayChoose, WhenStuck, About }`: the model may ask one focused question, and stuck work asks once before it gives up. The work stops where it stands and goes on with the answer (`AnswerAgentQuestion`), a minute or a month later. In a workflow, `await agent.Do(…)` waits with it. |
| **Handing work on** | `Delegates => new Delegation { To = [Specialist.Of<Checker>(new CheckPapers()), Specialist.Helper(new Summarise())], MaxDepth }`: the model is offered each as a tool, typed in and out, and pieces asked for in one turn are done side by side. A piece has its own receipt under the one that handed it on (`Parent`, `Root`, `Depth`, and `Sequence` — its place in the order it was handed on); it is done for the same person, proposes what any piece of work above it would have to propose, never does what one above it never does, and spends out of what is left above it. `WorkTree.Of(task)` reads the tree; one record, one chain. |
| **Guards** | `AgentsSetup.Guards = [new MyGuard()]` — a class of the app's asked before and after every model call and every tool, which may stop the step with a reason. The kit ships none. |
| **Trying again** | `Retry => new RetryRule { Attempts = 2 }` on a skill: failed work is tried again from the start, each attempt with its own receipt. `AgentWorkGaveUp` once the last has failed. |
| **Corrections** | `CorrectAgentWork(task, new Correction { Wrong, Right, Because })` — on the result, on one step of the record (`Step`) or on a proposal — by whoever the work was for, gave it, or is in charge. A record on that work: it changes nothing the agent does. |
| **Instructions** | Standing guidance for an agent, a skill, one person's work or what work is charged to. Proposed by a person (`ProposeAgentInstruction`), a model (`DraftAgentInstruction`) or the agent (`SuggestsInstructions`) — and sent to no model until a signed-in person `TeachesAgents` admits makes it active (`ActivateAgentInstruction`). Never an agent's own account; never code running as the app. |
| **Instruction sets** | Every activation, retirement (`RetireAgentInstruction`) and rollback (`RollBackAgentInstructions(agent, toSet, why)`) is the agent's next numbered set: who, when, why, what is active after it, and a hash. A receipt records the set its work started under and the hash of what it was sent; a proposal's fingerprint covers the set. `InstructionsOf(task)` explains a past result; `EffectOf(instruction)` counts whether it helped. |
| **In charge** | An agent runs only once somebody answers for it (`SponsorAgent`). `PauseAgent` / `ResumeAgent` stop it taking new work; `StopAgentTask` calls one piece of work off at its next step. |
| **The catalogue** | `Skills.Catalog()` — every skill of every agent: what it takes and hands back (with schemas), its version, and its tools with what each one does. What a picker or a canvas lists. |
| **On demand** | `RunSkill(agent, skill, inputJson)` — a person runs one skill, as any work they give the agent; `InputFieldsOf(skill)` is the form, built from what the skill takes, and `InputJsonOf(fields, values)` its answers as JSON. The agent's page in [`Osysharp.Agents.Management`](../agents-management) has a **Run** button built from them. |
| **Events** | `AgentTaskStarted`, `AgentTaskFinished`, `AgentTaskFailed`, `AgentTaskStopped`, `AgentTaskWaiting`, `AgentWorkGaveUp`, `AgentProposed`, `AgentProposalDecided`, `AgentAskedForHelp`, `AgentQuestionClosed`, `AgentWorkErased`. Hear them with `[On]`. |

## Who is in charge

Twelve policies. Your app answers `UsesAgents` and `SupervisesAgents`; everything else is nobody's until it says so,
except the four that start out as *whoever supervises*:

```osy
policy UsesAgents             => IsStaff;     // gives agents work, and reads the runs they asked for
policy SupervisesAgents       => IsLead;      // stops any work, answers any question, takes back any rule, reads every run
                                              // …and with it, until you hand one to somebody else:
policy SponsorsAgents         => SupervisesAgents;                  // names the person who answers for an agent
policy PausesAgents           => SupervisesAgents || IsOnCall;      // pauses an agent, and lets it work again
policy ReadsAllAgentWork      => SupervisesAgents || AuditsAgents || IsFinance;   // every receipt — its figures, not its words
policy DecidesAgentProposals  => SupervisesAgents;                  // decides a proposal put to somebody else

policy WidensAgentAutonomy    => IsLead;      // may say "from now on, do this alone" for the work an agent does for them
policy TeachesAgents          => IsLead;      // makes an instruction active, rejects, retires and rolls back — never an agent's own account — and reads every run with its content, to judge and correct it
policy DirectsAgentsForOthers => IsLead;      // may give an agent work FOR somebody else — it then acts with what THEY may do
policy AuditsAgents           => IsAuditor;   // reads every run with its content
policy ErasesAgentRecords     => IsRecords;   // erases what a run said, takes a person off it, deletes it when its time is up
policy AgentsMayConsult       => true;        // whether an agent may ask the model it consults (default: yes)
```

A person who asked for a piece of work reads its receipt and its record, content included. A supervisor reads that
every piece of work happened, in words, but not what was said. An auditor reads everything. Somebody admitted by
`ReadsAllAgentWork` alone — finance — reads what work was done and what it cost, and not a word of it.

`PauseAgent(agent, why)`, `ResumeAgent(agent, why)` and `SponsorAgent(agent, person)` each leave a line that says
who did it, when and why (`SeatChangesOf(agent)`). `RetryAgentWork(task)` starts work that failed, was stopped or was
called off again from what it was given — for whoever may read that. The screens for all of it are
[`Osysharp.Agents.Management`](../agents-management).

## Giving an agent work

```osy
// A typed job. Every way out is a status — a model that fails, money that runs out, a result that never fits.
AgentOutcome<Grade> outcome = await Agents.Of<Inspector>().Do(new GradeCrate(), crate, about: "Crate " + crate.Code);
outcome.Done;  outcome.Result;  outcome.Status;  outcome.Problem;  outcome.CostMicros;  outcome.Task;

// In words.
AgentReply reply = await Agents.Of<Inspector>().Ask("What does A-2 weigh?");

// As it arrives — for an answer somebody watches.
stream<AgentSignal> Watch(string question) {
  foreach (var s in Agents.Of<Inspector>().Stream(question)) { yield return s; }
}

// For a person who is not the one signed in.
await Agents.Of<Inspector>().For(order.Customer).Do(new GradeCrate(), crate);

// A typed decision, with a second opinion when the first is not sure enough.
Decision<Line> d = await Agents.Of<Sorter>().Decide<Line>(whatWasFound, "Which line does this crate go to?");
```

`Do`, `Ask` and `Decide<T>` each answer a task, as any call that waits on a model does. `await` it where you call
it for the plain call; keep the tasks — `var a = inspector.Do(…); var b = inspector.Do(…);` — and both pieces of work
are in flight at once, each with its own receipt, limit and record. (Work started that way may ask a model and hand
back; a tool it runs there is refused today. For several agents at work at once with their tools, see *Handing work
on*.)

`Status` is `Done`, `Failed` (the model could not be reached, the result never fitted, it went in circles), `Stopped`
(its turns or its money ran out) or `Cancelled` (somebody called it off). Work is refused before it starts — an
exception, and no receipt — when the class is not in `app.Agents` (`AgentNotListed`), nobody answers for the agent
(`AgentNotSponsored`), it is paused (`AgentPaused`), or it does not hold the skill (`SkillNotHeld`).

## Asking for help

```osy
[Actor(Roles = [PackhouseRole.Grader])]
public class Sorter : Agent {
  public override string Consults => "Strong";            // one of the app's models, by name
  public override decimal ConfidenceFloor => 0.8m;        // a decision under this is asked again of Strong
  public override int StuckAfter => 2;                    // two failed attempts in a row: consult once, then fail
}
```

A consult is an ordinary model call: charged to the same piece of work, held to the same limits, on the meter and on
the record. It is never stepped down to a cheaper model — a consult that does not fit the budget is refused, and the
work is told. With `Consults` unset, nothing of an agent's work is ever sent to another model.

## What it does alone, and what it proposes

```osy
[Actor(Roles = [PackhouseRole.Grader])]
public class Porter : Agent {
  public override Permits May => new Permits {
    Changes = Leave.Alone,                             // it flags and moves crates by itself
    ProposeEveryTime = [Actions.Of(MoveToLine)],       // a change of line is put to a person, every time
    Never = [Actions.Of(CondemnCrate)],                // and it never condemns a crate
  };
  public override AsksPeople People => new AsksPeople { MayChoose = true, WhenStuck = true };
}

var outcome = await Agents.Of<Porter>().Do(new SettleCrate(), crate);
if (outcome.Waiting) {                                 // it stopped on a person: a proposal, or a question
  var proposal = ProposalsForMe().Single();            // the exact call, why, and what it had looked at
  ApproveAgentProposal(proposal);                      // runs it now, as whoever approves
  // RejectAgentProposal(proposal, "It is the variety's colour, not a bruise.");
}
```

A yes from somebody who may not do the thing does not do it. A yes to a crate that has since been weighed again is
not a yes to it as it is now. What an agent never does is not offered to the model, not proposed, and not something
anybody can approve. With [`Osysharp.Agents.HumanRequests`](../agents-human-requests) in the app, proposals and
questions are requests in the person's inbox.

## Handing work on

```osy
/// Plans a rebinding: has the leaves counted and the cloth priced — at the same time.
[Actor(Roles = [BinderyRole.Helper])]
public class Planner : Agent {
  public override List<SkillBase> Skills => [new PlanRebinding()];
  public override Delegation Delegates => new Delegation {
    To = [Specialist.Of<Collator>(new CountLeaves()),                  // another agent's skill
          Specialist.Of<Buyer>(new PriceCloth(), budget: 0.006m),      // …held to a share of its own
          Specialist.Helper(new Summarise())],                         // one of its own, done apart
    MaxDepth = 2,                                                      // how deep the tree under it may go
  };
}

var outcome = await Agents.Of<Planner>().Do(new PlanRebinding(), volume);
foreach (var node in WorkTree.Of(AgentTask.Single(t => t.Id == outcome.Task))) {
  // node.Depth, node.Agent, node.Skill, node.Status, node.WaitingOn, node.OwnMicros, node.TreeMicros
}
```

An agent that says nothing hands nothing on. Each entry is offered to its model as a tool — `hand_to_Collator_CountLeaves`
— whose input is what the skill takes and whose answer is what it hands back. Pieces the model asks for in one turn
go out together and their models answer side by side; the turn's answers go back together.

- **Never more than the work above it.** A piece is done by the specialist's own account, for the person the work at
  the top is for — read from the receipt above it, never from whoever is signed in. Its gate is the strictest of its
  own and of every piece above it: what any of them proposes first is proposed, what any of them never does is not
  offered, not run and not approvable. It keeps its own roles — that is what makes it a specialist.
- **Money flows down and counts up.** A piece is opened with the least of its share, its own limit and what the work
  above it has left; each of its calls is charged to — and held to the limit of — every piece above it, and whatever
  the top was charged to with `On`. A piece that runs out is stopped; when the limit that ran out is a piece's above
  it, everything under that piece is called off. The model above is told, and carries on.
- **People, from any depth.** A proposal or a question goes to the person the work at the top is for — its sponsor
  when it is for nobody — and says through whom it comes (`AskedThrough`: "Collator, working for Planner"). The whole
  chain waits, each receipt saying what it waits for, and goes on from the same point when the person has decided.
- **Stopping and failing.** `StopAgentTask` on any piece calls off everything under it. A piece that fails — after
  the retries its skill names — is told to the model above with how it ended, why, and what its tools had found.
  Nothing of a tree is left going once its top has ended.
- **A row is named by its id.** A skill that takes one of the app's rows says how it is found when the work is handed
  on: `HandedSchema() => HandOn.ById("The volume to count.")` and `Handed(json)` — run as the specialist, for the
  person, so a row they may not read is not found.

## As a step of a process

In a workflow's body, `Do` is the same call — and it waits. Work that stops on a person parks the run with it, for
as long as the person takes, and the step answers the finished work; a step that failed, was stopped or gave up
answers that, and the process goes on from there like from any branch.

```osy
state Vetting {
  enter {
    var vetted = await Agents.Of<Vetter>().For(this.Item.RequestedBy).Do(new VetSupplier(), this.Item);
    if (!vetted.Done) { goto NeedsAPerson; }
    goto Deciding;
  }
}
```

The receipt says which run gave it the work (`AgentTask.Run`, `RunWorkflow`, `RunState`, `RunItem`, `RunStarter`,
`RunStep`), every model call of the work is charged to that workflow and that run — so a budget for each run holds
it — and a run that ends while a step still waits calls the work off. With
[`Osysharp.Processes`](../processes) in the app, each step is told on its run and opens there as the
[inspector](../agents-inspector) ([`Osysharp.Agents.Processes`](../agents-processes)).

## What only the kit writes

Receipts, records, sponsors and requests for help are rows in your app's database that only this package's functions
write. Each of these is a test in the kit's test app:

| | |
|---|---|
| A receipt is made, counted and closed only by the kit | `a_receipt_is_written_only_by_the_kit` |
| A line of the record is written only by the kit, and never changed | `the_record_is_written_only_by_the_kit_and_never_changed` |
| A line changed, removed or slipped in is found | `a_line_that_was_changed_removed_or_slipped_in_is_found` |
| The chain checks after what the work said is erased, and after the person is taken off it | `the_chain_verifies_after_what_a_piece_of_work_said_is_erased`, `the_chain_verifies_after_the_person_is_taken_off_the_work` |
| A line changed after an erasure is still found, and nothing can be written in an erased line's place | `a_changed_line_is_still_found_after_an_erasure`, `a_line_that_was_not_erased_cannot_be_passed_off_as_erased` |
| Only whoever the app names erases, and only through the kit | `only_whoever_the_app_names_erases_and_only_through_the_kit` |
| A tool reads only what the agent and whoever asked may both read | `a_tool_reads_only_what_the_agent_and_whoever_asked_may_both_read` |
| A tool changes only what both may change | `a_tool_changes_only_what_the_agent_and_whoever_asked_may_both_change` |
| A tool the skill does not list never runs | `a_tool_the_skill_does_not_list_never_runs_whatever_the_model_calls_it` |
| An agent without a sponsor does not run | `an_agent_without_a_sponsor_does_not_run` |
| What an agent never does is not proposed, and nobody can allow it | `out_of_envelope_is_never_approvable` |
| An approved proposal runs as whoever approved, and only if they may | `an_approved_proposal_runs_as_the_approver` |
| A yes does not cover what changed after it was proposed | `a_yes_does_not_cover_what_changed_after_it_was_proposed` |
| A proposal is decided once, through the kit, by who it is put to | `a_proposal_is_decided_only_through_the_kit_by_who_it_is_put_to` |
| Nobody gives an agent work for a colleague to borrow what they may do | `nobody_borrows_a_colleagues_rights_through_an_agent` |
| An instruction is sent to no model until a person who teaches makes it active | `an_instruction_is_active_only_after_a_person` |
| Nobody writes an instruction active, or a set, past the kit | `nobody_writes_an_instruction_active_past_the_kit` |
| An agent's own account cannot activate one, even working for a teacher | `an_agent_cannot_activate_an_instruction_even_for_a_teacher` |
| Code running as the app cannot activate one | `code_running_as_the_app_cannot_activate_one` |
| What an agent suggests for itself is only proposed | `what_an_agent_suggests_for_itself_is_only_proposed` |
| A piece of work records the skill's version and the instruction set it ran under | `a_task_records_the_version_it_ran` |
| A yes does not carry from one instruction set to another | `a_yes_does_not_carry_from_one_instruction_set_to_another` |
| Work under way finishes under the set it started with | `work_in_flight_finishes_under_the_set_it_started_with` |
| A rollback restores an earlier set, and is itself a record | `a_rollback_restores_an_earlier_set_and_is_itself_a_record` |
| A correction is made and changed only through the kit | `a_correction_is_made_and_changed_only_through_the_kit` |
| A question is answered once, by who was asked | `a_question_is_answered_only_by_who_was_asked` |
| Handed-on work is for whoever the work above it is for — nobody else — and its gate is no wider than any piece above it | `a_child_never_holds_more_than_its_parent` |
| What any piece of work above never does is not offered, not run and not approvable below | `what_any_work_above_never_does_is_never_done_below`, `a_yes_does_not_make_a_piece_do_what_the_work_above_it_now_never_does` |
| Work is handed on only as the agent's class names, and no deeper than the work above allows | `work_is_handed_on_only_as_the_agent_declares`, `work_is_handed_on_no_deeper_than_the_work_above_allows` |
| A piece spends against every limit above it, and is given no more than is left there | `a_childs_spend_counts_against_its_parent_and_the_top`, `a_child_is_given_no_more_than_its_parent_has_left` |
| A piece that runs out stops everything under it and nothing beside it | `a_spent_child_cancels_its_subtree` |
| A proposal from any depth goes to the person the top is for, and runs as them and as the piece that proposed it | `an_approval_from_depth_two_reaches_the_root_requester` |
| Stopping the top calls off everything under it, and nothing is left going | `stopping_the_work_at_the_top_calls_off_everything_under_it` |
| Work is stopped when its money runs out, before the call that would overrun it | `a_piece_of_work_is_stopped_when_its_money_runs_out_and_keeps_what_it_did` |
| What a receipt says was spent is what the meter says | `what_a_receipt_says_was_spent_is_what_the_meter_says` |
| The record's content is read by whoever asked and by the auditors, and by nobody else | `the_record_is_read_by_whoever_asked_the_supervisor_and_the_auditor_and_its_content_by_fewer` |
| The consulted model is given no tools, and only the question | `a_model_may_ask_the_one_its_agent_consults_and_is_given_the_advice` |
| With nobody to consult, nothing goes to another model | `an_agent_that_names_nobody_to_consult_never_sends_anything_to_another_model` |

## Its tests

`kits/agents/tests` is a packhouse: growers bring fruit by the crate, an inspector grades it, a sorter decides which
line it goes to. Two scripted models, no router, and suites for giving work (`tasks`), the record (`trace`), the
catalogue (`skills`), money (`budgets`), asking for help (`help`), erasing (`erasure`), what an agent does alone
and proposes (`autonomy`), asking a person, guards and trying again (`people`), corrections and who makes an
instruction active (`teaching`), instruction sets, rollback and what work is sent (`instruction-sets`), and handing
work on to other agents (`delegation`: 19 tests over a crew of four that hand deliveries and crates on, three levels
deep), a step of a process (`processes`), who is in charge (`roles`) and trying work again (`again`). 135 tests.

```
osy test kits/agents/tests
```

## Who decides a proposal, and where it goes

A proposal goes to the person the work is for (the agent's sponsor when it is for nobody). A skill may send what
it proposes for a tool somewhere else — "refunds go to Finance":

```osy
public override ProposalRoute? DecidedBy(string action) =>
  action == Actions.Of(RefundOrder).Name ? ProposalRoute.ToQueue("Finance") : null;   // or ProposalRoute.ToPerson(account)
```

Then the person the work is for reads the proposal and does not decide it; whoever works that queue does (with
[`Osysharp.Agents.HumanRequests`](../agents-human-requests)), and so may whoever `DecidesAgentProposals` admits. A
tool the agent never runs is still never proposed. Tests: `tests/routing.test.osy`.

## When the process doing the work dies

A receipt that says `Working` with nothing written on its record for `AgentsSetup.AbandonedAfter` (30 minutes) was
abandoned. The kit closes it by itself: its own clock looks every `AgentsSetup.AbandonedCheckEvery` (ten minutes;
zero: never) and closes such work with everything it had handed on — nothing for the app to schedule.
`CloseAbandonedAgentWork()` is the same look at once, for whoever supervises; `StopAgentTask` on abandoned work closes
it at once; and a process that comes back after its work was closed does nothing more in its name. Tests:
`tests/delegation.test.osy` ("a process that dies", "closed by the kit's own clock").

## Source and package

- Package: `Osysharp.Agents` — `use Osysharp.Agents@1;`
- Source: [`kits/agents`](.) — `model/Setup.osy` (policies, the app's setting), `Agent.osy` (agents, `Do` / `Ask` /
  `Stream` / `Decide`), `Skill.osy` (skills, the catalogue), `Loop.osy` (the loop, tools), `Task.osy` (the receipt,
  sponsors), `Trace.osy` (the record and its chain), `Delegation.osy` (handing work on: specialists, the tree, money down
  and up, calling a subtree off), `Help.osy` (asking for help), `Erasure.osy` (erasing what a record
  said), `Corrections.osy`, `Instructions.osy` (corrections, instructions, instruction sets, rollback), `OnDemand.osy` (running a skill from a form), `Abandoned.osy`, `Events.osy`
- Uses [`Osysharp.Llm`](../llm) (every model call: routed, budgeted, metered), `Osysharp.Identity` (who an agent is
  and who it works for) and `Osysharp.Observability` (an agent's changes are always on the change trail).
- Testing: [`Osysharp.Agents.Testing`](../agents-testing).
- Screens for teaching: [`Osysharp.Agents.Teaching`](../agents-teaching) — "Correct this" and an agent's Instructions page.
