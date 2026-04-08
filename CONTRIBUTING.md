# Contributing to ARO Training

Thank you for your interest in improving this training material!

---

## Types of Contributions

- **Bug fixes** — incorrect commands, broken YAML, outdated API versions
- **Content improvements** — clearer explanations, better examples
- **New labs** — hands-on exercises for topics not yet covered
- **New modules** — entire topic areas (e.g., GitOps with ArgoCD, ACR integration)

---

## Pull Request Process

1. Fork this repository and create a feature branch:
   ```bash
   git checkout -b feature/my-improvement
   ```

2. Make your changes following the style guide below.

3. Test all commands — ensure every `oc`, `kubectl`, and `az` command runs successfully against an ARO 4.x cluster.

4. Test all YAML — validate manifests with:
   ```bash
   oc apply --dry-run=client -f path/to/manifest.yaml
   ```

5. Submit a PR with a clear title, description, and issue references.

---

## Markdown Style Guide

- Use `#` for document title (one per file)
- Use `##` for major sections
- Use `###` for subsections
- Always specify language in code blocks
- Use tables for comparisons
- Use blockquotes for notes and warnings

---

## Testing Checklist

- [ ] All YAML manifests pass: `oc apply --dry-run=client`
- [ ] All `az aro` commands are valid
- [ ] All `oc` commands are valid for OpenShift 4.x
- [ ] No hardcoded credentials or subscription IDs
- [ ] New lab has estimated time and prerequisites listed

---

## License

By contributing, you agree your contributions will be licensed under the MIT License.
