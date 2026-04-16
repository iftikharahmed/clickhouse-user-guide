# 🤝 Contributing to ClickHouse User Guide

First off — thank you for considering a contribution! Every improvement, no matter how small, helps the community.

---

## 📋 Ways to Contribute

- 🐛 **Fix a mistake** — typo, wrong code, outdated info
- ➕ **Add an example** — real-world query or use case
- 🌍 **Translate** — help make this guide available in other languages
- 💡 **Suggest a topic** — open an issue with your idea
- ⭐ **Star the repo** — helps more people discover this guide

---

## 🚀 How to Submit a PR

1. **Fork** this repository
2. **Create a branch** — `git checkout -b improve/section-name`
3. **Make your changes** — follow the style guide below
4. **Commit** — `git commit -m "docs: improve explanation of MergeTree"`
5. **Push** — `git push origin improve/section-name`
6. **Open a Pull Request** — describe what you changed and why

---

## ✍️ Style Guide

- Use **simple, plain English** — assume the reader is smart but new to ClickHouse
- Every concept should have a **code example**
- Prefer **short paragraphs** over walls of text
- Use **tables** for comparisons
- Use **callout boxes** (blockquotes) for tips, warnings, and notes:

```markdown
> 💡 **Tip:** Always use LowCardinality for columns with fewer than 10,000 unique values.

> ⚠️ **Warning:** Avoid JOINs on large tables without proper indexing.
```

---

## 📁 File Structure

Each section lives in its own folder with a `README.md`:

```
01-introduction/
└── README.md
```

If a section needs supplementary files (e.g., sample datasets, diagrams), add them in the same folder.

---

## 💬 Questions?

Open a [GitHub Issue](https://github.com/iftikharahmed/clickhouse-user-guide/issues) — happy to help!

---

<div align="center">

**Thank you for making this guide better for everyone! 🙏**

</div>
