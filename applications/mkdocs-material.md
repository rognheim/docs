---
title: MkDocs Material
status: working
last_reviewed: 2026-04-16
owner: Rognheim
---

# :material-file-document-multiple:


## **MkDocs Material**

---

### **Role summary**

MkDocs Material is the documentation stack behind this repository. It provides navigation, rendering, search, and theme behavior for the published site.

---

### **Deployment model**

- Documentation source lives in Git under `docs/` and `zensical.toml`
- Local build or preview runs from the project container workflow
- Published site behavior depends on both Markdown content and navigation structure

---

### **Required dependencies**

- [Git](../infrastructure-tools/git.md)
- [Gitea Actions](../infrastructure-tools/gitea-actions.md)
- [Markdown](../programming/markdown.md)

---

### **Post-install checklist**

- Build locally before publishing structural doc changes.
- Verify navigation, section indexes, and internal links after edits.
- Check custom CSS, images, and external assets for regressions.
- Keep repo structure and published structure aligned.

---

### **Related canonical pages**

- [Guides](../guides/index.md)
- [Programming](../programming/index.md)
- [Gitea Actions](../infrastructure-tools/gitea-actions.md)
