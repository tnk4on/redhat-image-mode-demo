# AI Agent Guidelines

This project is a Japanese translation of the Red Hat Enterprise Linux Image Mode demo documentation.

## Translation Rules

### Terminology Translation Table

Use the following Japanese translations for these terms:

| English | Japanese |
|---------|----------|
| Image Mode | イメージモード |
| bootc | bootc (keep as-is) |
| bootc-image-builder | bootc-image-builder (keep as-is) |
| container | コンテナ |
| Containerfile | Containerfile (keep as-is) |
| Golden Image | ゴールデンイメージ |
| Standard Operating Environment (SOE) | 標準オペレーティング環境（SOE） |
| upgrade | アップグレード |
| update | アップデート/更新 |
| rollback | ロールバック |
| deploy | デプロイ |
| use case | ユースケース |
| quickstart | クイックスタート |
| repository | リポジトリ |
| registry | レジストリ |
| VM / Virtual Machine | VM / 仮想マシン |
| layer | レイヤー |
| atomic | アトミック（原子的） |
| immutable | イミュータブル（不変） |
| workflow | ワークフロー |
| pipeline | パイプライン |
| CI/CD | CI/CD (keep as-is) |
| GitOps | GitOps (keep as-is) |

### Terms to Keep in English

The following terms should remain in English:

- Command names (podman, git, ssh, curl, etc.)
- File names and paths
- Package names (httpd, mariadb, nginx, etc.)
- Configuration file contents
- Code inside code blocks
- URLs
- Example usernames/passwords (bootc-user, redhat, etc.)

### Style Guidelines

- Use polite form (です・ます style)
- Translate technical terms accurately, use katakana when appropriate
- Keep instructions concise
- Translate warning/tip headings to Japanese

### Markdown Conventions

- Keep admonition types (warning, tip, note) unchanged
- Do not modify code blocks
- Translate link text to Japanese
- Image alt text does not need translation

## File Structure

```
docs/
├── index.md                    # Homepage
├── getting-started/            # Getting Started section
│   ├── introduction.md         # Image Mode introduction
│   ├── quickstart.md           # Quickstart guide
│   ├── contributing.md         # Contribution guidelines
│   └── review-and-test-changes.md
└── use-cases/                  # Use Cases
    └── [each-use-case-folder]/README.md
```

## Translation Guidelines

1. Maintain consistency with existing translations
2. Prioritize technical accuracy
3. Choose appropriate translations based on context
4. For unclear terms, keep the original and add Japanese explanation in parentheses
