## Configuration Diff

{{ if .error_message }}
### ❌ Build Failed

{{ .error_message }}

<details>
<summary>Error Log</summary>

```
{{ .error_log }}
```
</details>

{{ else }}
**{{ .total }}** document(s) impacted:
```diff
+ {{ .added }} added
- {{ .removed }} removed
! {{ .modified }} modified
```

{{ end }}
{{ if .diff_content }}
<details>
<summary><strong>Diff</strong></summary>

```diff
{{ .diff_content }}
```

</details>

{{ end }}
---

<sub>📦 **Artifacts:** [base-output.yaml, head-output.yaml, dyff-output.txt](https://github.com/{{ .repo }}/actions/runs/{{ .run_id }})</sub>
