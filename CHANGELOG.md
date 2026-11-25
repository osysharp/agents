# Osysharp.Agents — changes

## 2.13.1
- A refusal that comes from the app's own rules says so, naming the rules and the account they were asked about:
  proposing an instruction outright, from somebody else's correction or for one other person ("Whoever the app's
  `TeachesAgents` or `SupervisesAgents` admits may propose any instruction outright, and both answer no for Claude
  now."), correcting a piece of work and taking a correction back. They used to say only "Whoever teaches the agents
  may propose one outright", so an account the app admits only some of the time — a person's delegate whose stated
  reason had run out — read it as "you do not teach" and went looking for a bug in the kit (PLATFORM-725). Who may is
  unchanged: the kit asks the app's rules about the account that is signed in, exactly as before.

## 2.10.0
- Its organization-policy settings name their area from the platform's list (`Area = OrgPolicyArea.Agents`), not as text.
  Needs platform 0.23.0 (`minPlatform "0.23.0"`): an older platform refuses this version by its floor.
