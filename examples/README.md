# Examples

- `config.intercom-only.yaml`: one agent, one API policy (shows `team_scope` and `enum_overrides` comments).
- `config.multi-agent.yaml`: two agents with different access, `response_validation` per policy.

To use an example:

```bash
cp examples/config.multi-agent.yaml config.yaml
./render
```
