# Contributing to Loki-LLM Analyst

Thank you for your interest in contributing! This project welcomes contributions in the form of new Fabric patterns, lab improvements, documentation fixes, and bug reports.

---

## Ways to Contribute

### 1. Add a New Fabric Pattern

The most impactful contribution you can make is a new security analysis pattern.

**Pattern structure:**
```
fabric-patterns/
└── your_pattern_name/
    └── system.md    ← The system prompt for the LLM
```

**Requirements for a good pattern:**
- Clear `# IDENTITY and PURPOSE` section explaining what the pattern does
- `# DETECTION CATEGORIES` with at least 3 specific, actionable detection criteria
- `# STEPS` section with a clear analysis workflow
- Structured `# OUTPUT FORMAT` with MITRE ATT&CK mapping
- Output must include: `SUMMARY | IOCS | MITRE | RECOMMENDATION`

**Test your pattern:**
```bash
# Copy to local Fabric
cp -r fabric-patterns/your_pattern_name ~/.config/fabric/patterns/

# Test with sample data
echo "your sample log data here" | fabric -p your_pattern_name
```

---

### 2. Improve the Lab

The `lab/` directory welcomes:
- More realistic attack patterns in `access.log`
- New LAB exercises in `LAB_INSTRUCTION.md`
- Grafana dashboard JSON for pre-built visualizations
- Additional Loki/Promtail configuration examples

---

### 3. Fix Documentation

- Typos in any `.md` file
- Outdated version numbers in `docker-compose.yml`
- Missing steps in guides

---

## Contribution Workflow

1. **Fork** this repository
2. **Create a branch**: `git checkout -b feature/new-pattern-name`
3. **Make your changes**
4. **Test**: verify your pattern works with `fabric -p your_pattern_name`
5. **Submit a Pull Request** with:
   - What the pattern/change does
   - Sample input you tested with
   - Sample output from the pattern

---

## Pattern Quality Checklist

Before submitting a new pattern, confirm:

- [ ] Pattern name is lowercase with underscores (e.g., `new_pattern_name`)
- [ ] `system.md` starts with `# IDENTITY and PURPOSE`
- [ ] Has at least 3 detection categories
- [ ] Output format includes SUMMARY, IOCS, MITRE ATT&CK, and RECOMMENDATION sections
- [ ] MITRE technique IDs are accurate (verify at [attack.mitre.org](https://attack.mitre.org))
- [ ] Tested locally with `fabric -p your_pattern_name`
- [ ] Added to the table in [`fabric-patterns/README.md`](fabric-patterns/README.md)

---

## Code of Conduct

Be respectful. This is a security education project — contributions should be oriented toward defense and detection, not offense.

---

## Questions?

Open a GitHub Issue with the `question` label.
