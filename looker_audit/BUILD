load("//learning/gemini/agents/skill_rules:skills.bzl", "agent_skill")
load("//learning/gemini/agents/skills/skill_creator/scripts:skill_defs.bzl", "validate_skill_test")

agent_skill(
    name = "looker-audit",
    srcs = ["SKILL.md"],
)

validate_skill_test(
    name = "validate_looker_audit_test",
)
