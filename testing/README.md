# Osysharp.Agents.Testing

Test an agent without a model: write what the model answers, turn by turn, and assert what the agent then did.

```osy
use Osysharp.Agents@1;
tests use Osysharp.Agents.Testing@1;

[Test]
[runas(Ada)]
void a_short_crate_is_weighed_flagged_and_graded() {
  var script = AgentScript.Of(
    Says.Tool(Actions.Of(Weigh), "{\"crate\":\"" + crate.Id + "\"}"),
    Says.Tool(Actions.Of(FlagCrate), "{\"crate\":\"" + crate.Id + "\",\"flag\":\"short\"}"),
    Says.HandBack(new Grade { Score = 4, Short = true }));
  ScriptedAnswer.Stub((model, request) => script.Answer(model, request));

  var outcome = await Agents.Of<Inspector>().Do(new GradeCrate(), crate);

  Assert.True(outcome.Done);
  Assert.Equal(["Weigh", "FlagCrate"], Trajectory.Of(outcome.Task));     // what it did, in order
  Assert.Equal("short", crate.Flag);                                      // …really did
  Assert.Equal(600, Trajectory.Cost(outcome.Task));                       // and what it cost
}
```

## Use case

An agent is hard to test the usual way. A test against a real model costs money every run, needs a key in CI, takes
seconds, and gives a slightly different answer each time — so the suite is either skipped or red for no reason.

Here the model is the one thing that is scripted. The loop, the agent's rights, the tools, the budget, the meter and
the record are the real ones, so a test proves what matters and what a model cannot be asked to promise:

- the tools the agent ran, and in what order;
- that a tool it was not given never ran, and one it was not allowed was refused;
- what the work cost, and that it stopped where its limit says;
- what it handed back, and what happened when that did not fit.

It costs nothing, needs no key, and is the same on every machine.

## What it does

| | |
|---|---|
| **`Says`** | One answer of the model: `Says.Tool(Actions.Of(F), argumentsJson)` — it asks for a tool; `Says.HandBack(value)` — it hands its typed result back; `Says.HandBackJson(json)` — a result that is not the shape; `Says.Words(text)` — it answers in words; `Says.Consult(question, context)` — it asks the model its agent consults; `Says.AskPerson(question, context)` — it asks a person, and the work waits; `Says.Tool("why", Actions.Of(F), argumentsJson)` — a tool call with the model's reason before it, as a proposal records it. Each takes the tokens the answer is said to have used, which is what the meter prices. |
| **`AgentScript`** | The answers in order: `AgentScript.Of(a, b, c)`, `.Then(d)`. `.Advises("Strong", "…")` is what a consulted model says; `.IsDown("Small")` makes a model fail as a provider that cannot be reached. `script.Answer(model, request)` is what a test hands `ScriptedAnswer.Stub`. Which turn it is is read from the request, so a script answers the same however calls are interleaved. |
| **`Trajectory`** | What a piece of work did, read from its record: `Of(task)` — the tools that ran, in order; `Called(task, Actions.Of(F))`; `Models(task)` — the model that served each call; `Kinds(task)` and `Said(task)` — every line of the record; `Helps(task)`; `Cost(task)`. |
| **`AgentCast`** | Several agents scripted at once, for work one agent hands on to others: `AgentCast.Of().Plays<Coordinator>(script).Plays<Checker>(script)`; `.PlaysAbout<T>("Crate A-2", script)` picks among several pieces of one agent's work by a few words of its job or its brief. `Says.HandTo<T>(skill, inputJson)` hands a piece on (`Says.Row(id)` for a skill that takes a row); `Says.Together(a, b)` puts several in one turn, which is what makes them run side by side. `Trajectory.Tree(task)` reads the tree back a line a piece; `HandedOn`, `TreeCost`, `StillGoing`. |

A script that runs out of answers fails the call the way a provider does, and says which turn had none.

`ProcessTrail` reads back what the agents did in one run of a process — `Skills(item)`, `States(item)`,
`Ended(item)`, `Proposed(item)`, `Cost(item)` — for a test that walks a whole workflow with scripted models.

## Its tests

The kit's own test app is a lighthouse log: a keeper's assistant reads the day's entries and files a report — and a
bosun hands the report and the watch on to two others at once, each with a script of its own.

```
osy test kits/agents-testing/tests
```

## Source and package

- Package: `Osysharp.Agents.Testing` — `tests use Osysharp.Agents.Testing@1;`
- Source: [`kits/agents-testing`](.) — `model/Script.osy`
- Uses [`Osysharp.Agents`](../agents) and the scripted model of [`Osysharp.Llm`](../llm).

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.Llm;
using Osysharp.Agents;
using Osysharp.Agents.Testing;
using Lighthouse;

[Test(Rock)]
[runas(Kit)]
void a_script_answers_turn_by_turn_and_the_trajectory_is_what_the_agent_did() {
  var script = AgentScript.Of(
    Says.Tool(Actions.Of(CountLog), "{}"),
    Says.Tool(Actions.Of(FileLog), "{}"),
    Says.HandBack(new Report { Ships = 1, Weather = "Fog from noon." }));
  ScriptedAnswer.Stub((model, request) => script.Answer(model, request));

  var outcome = FileToday();

  Assert.True(outcome.Done);
  Assert.Equal(1, outcome.Result.Ships);
  Assert.Equal("Fog from noon.", outcome.Result.Weather);
  Assert.Equal(["CountLog", "FileLog"], Trajectory.Of(outcome.Task));
  Assert.True(Trajectory.Called(outcome.Task, Actions.Of(FileLog)));
  Assert.False(Trajectory.Called(outcome.Task, Actions.Of(DouseLamp)));
  // The tools really ran.
  Assert.Equal(2, LogEntry.Count(e => e.Filed));
}
```

From [`tests/script.test.osy`](tests/script.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Llm;
using Osysharp.Agents;
using Osysharp.Agents.Testing;
using Lighthouse;

[Test(Rock)]
[runas(Kit)]
void the_tokens_a_script_says_were_used_are_what_the_work_cost() {
  var script = AgentScript.Of(Says.Tool(Actions.Of(CountLog), "{}", 1000, 100), Says.HandBack(new Report { Ships = 1 }, 2000, 200));
  ScriptedAnswer.Stub((model, request) => script.Answer(model, request));

  var outcome = FileToday();

  // 1000 + 5×100 and 2000 + 5×200 millionths of a dollar, at the lamp's price.
  Assert.Equal(4500, Trajectory.Cost(outcome.Task));
  Assert.Equal(4500, outcome.CostMicros);
  Assert.Equal(["Lamp", "Lamp"], Trajectory.Models(outcome.Task));
}
```

From [`tests/script.test.osy`](tests/script.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Llm;
using Osysharp.Agents;
using Osysharp.Agents.Testing;
using Lighthouse;

[Test(Rock)]
[runas(Kit)]
void a_result_of_the_wrong_shape_and_then_a_good_one() {
  var script = AgentScript.Of(Says.HandBackJson("{\"Ships\":\"several\"}"), Says.HandBack(new Report { Ships = 3 }));
  ScriptedAnswer.Stub((model, request) => script.Answer(model, request));

  var outcome = FileToday();

  Assert.True(outcome.Done);
  Assert.Equal(3, outcome.Result.Ships);
  Assert.Equal(["Started", "ModelCall", "SentBack", "ModelCall", "Ended"], Trajectory.Kinds(outcome.Task));
}
```

From [`tests/script.test.osy`](tests/script.test.osy).

## Learn more

- [The Osysharp.Agents.Testing reference](https://osysharp.com/reference/agent/kit-agents-testing/)
- [Osysharp.Agents.Testing in the package catalogue](https://osysharp.com/packages/#osysharp-agents-testing)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
