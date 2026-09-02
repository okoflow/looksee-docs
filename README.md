# LookSee documentation

Documentation for [LookSee](https://github.com/okoflow/looksee), the
self-hosted video analytics system with a visual workflow editor. The same
pages are maintained in four languages:

| Language | Start page |
| --- | --- |
| English | [en/index.md](en/index.md) |
| Русский | [ru/index.md](ru/index.md) |
| עברית | [he/index.md](he/index.md) |
| 한국어 | [ko/index.md](ko/index.md) |

Every language directory holds the same set of files under the same names, so
a page in one language links to its sibling in another by directory only.
Screenshots are shared from `images/`.

## Writing conventions

- One page per topic, one topic per page. The first paragraph says what the
  page covers; headings are sentence case.
- Timeless, terse prose in the present tense. Full words: "configuration",
  "repository", "documentation".
- Product terms as they appear in Studio: workflow, node, Camera, Detect,
  Zone, Schedule, Alert, Snapshot, Monitor, credential. Interface labels are
  bold (**Run**), code and configuration are in backticks (`WEBRTC_HOST_IP`).
- Commands, configuration, and payloads go in fenced code blocks. Reference
  material goes in tables with the field, its default, and its meaning.
- Warnings and tips use GitHub alerts (`> [!WARNING]`, `> [!TIP]`), at most
  one or two per page.
- Screenshots come from a 1920 by 1080 viewport at 2x scale, light theme, with
  realistic but fictional data. Never include real credentials, addresses, or
  people.
- A translation mirrors the English page section by section. Commands, code,
  file paths, environment variables, node kinds, and API paths stay in
  English. Interface labels stay in English because Studio ships in English.

## Contributing

Open a pull request against `main`. Keep a change to one page or one
translation so it can be reviewed in one sitting. To report a problem with the
product itself, use the issue forms in the
[main repository](https://github.com/okoflow/looksee/issues/new/choose).

## License

The documentation is licensed under
[CC BY 4.0](LICENSE). Sample footage in the screenshots comes from the
[Intel IoT DevKit sample videos](https://github.com/intel-iot-devkit/sample-videos),
also CC BY 4.0.
