# Fix: Inline-Image (CID) Support for HTML Email Bodies

**Branch**: `fix/1485-inline-image-cid-attachments`
**Status**: Internal fork fix, pending review (not upstreamed)

## Problem

The work package that prompted this change (OP#1485) reported that `send_email`
could not send HTML emails containing embedded base64 images: a ~36KB HTML
body (image data inlined as a base64 `data:` URI) was rejected or truncated,
with the symptom described as "body is too large for the MCP parameter
transport."

### Root cause — corrected from the original report

The original report assumed the size limit was enforced somewhere in this
codebase and asked for it to be raised. That assumption does not hold: reading
the actual `send_email` tool definition in `mcp_email_server/app.py` and the
full outbound compose chain in `mcp_email_server/emails/classic.py`
(`compose_message` → `_create_message_with_attachments` →
`EmailClient.send_email` → `ClassicEmailHandler.send_email`) at commit
`5f9efbf` shows no `max_length` constraint on the `body` parameter and no
size check anywhere on the send path. The only size-related constant in the
repository, `MAX_BODY_LENGTH = 20000` in `classic.py`, truncates **inbound**
email bodies when reading received mail (`_parse_email_message` /
`get_emails_content`) — it has nothing to do with sending.

The actual constraint that produced the symptom lives outside this
repository, in the MCP tool-call parameter transport used by the calling
client. That layer is not owned by this fork and cannot be fixed by editing
this codebase — there is no limit here to raise.

What **is** a real, actionable gap: the attachment mechanism
(`_create_message_with_attachments` / `_create_attachment_part`) only ever
produced a flat `MIMEMultipart()` with `Content-Disposition: attachment`
parts. There was no `multipart/related`, `Content-ID`, or inline-disposition
support anywhere in the compose/send path — so there was no way to reference
an image from the HTML body without inlining its bytes as base64 directly in
`body`, which is exactly what runs into the transport-layer limit above.

### Why this is a genuine gap, not something already fixed

Checked both the local fork (`main`, `5f9efbf`) and `upstream/main`
(`cab9a6b`, 56 commits ahead at the time of this fix). The only
size/attachment-adjacent commits in that range are receiving-side:
`f2c2492` ("add max_body_length to get_emails_content for configurable body
truncation") and `de2bb87` ("add semantic IMAP tags and embedded attachment
content", an opt-in path for *reading back* attachment bytes for clients
without a shared filesystem). Neither adds CID/inline-image support to the
send path.

## Fix Applied

Added an optional `inline_images` parameter to `send_email` and
`save_to_mailbox`. Each item is `{"path": "/absolute/path/to/file", "cid":
"name"}`; the HTML body references it as `<img src="cid:name">`. Only the
local file path travels through the MCP tool call — the image bytes never
pass through the `body` text parameter, sidestepping the transport-layer
constraint entirely.

### File: `mcp_email_server/app.py`

Added the `inline_images` parameter to the `send_email` and
`save_to_mailbox` tool signatures (matching the existing
`Annotated[..., Field(...)]` idiom) and threaded it through to the handler
calls.

### File: `mcp_email_server/emails/__init__.py`

Added `inline_images` to the `EmailHandler.send_email` and
`EmailHandler.save_to_mailbox` abstract method signatures.

### File: `mcp_email_server/emails/classic.py`

- Added `EmailClient._create_inline_image_part`: builds a `Content-ID`
  referenced MIME part (`MIMEImage` for `image/*` types, `MIMEApplication`
  otherwise) with `Content-Disposition: inline`, reusing the existing
  `_validate_attachment` file-existence check.
- Added `EmailClient._create_message_with_inline_images`: builds a
  `multipart/related` message (HTML body + inline image parts). When regular
  `attachments` are also supplied, the `multipart/related` part is nested as
  the first child of an outer `multipart/mixed`, with the regular attachment
  parts appended after it — so clients that ignore the related structure
  still see the attachments as attachments.
- Extracted the existing body/attachment branching out of `compose_message`
  into `EmailClient._build_message_body` (kept `compose_message` under the
  project's mccabe complexity limit) and added the `inline_images` branch
  there: `ValueError` if `inline_images` is supplied with `html=False` (CID
  references only make sense inside an HTML body), otherwise builds the
  related/mixed structure.
- Threaded `inline_images` through `EmailClient.send_email`,
  `ClassicEmailHandler.send_email`, and `ClassicEmailHandler.save_to_mailbox`
  down to `compose_message`.
- The existing `attachments`-only and plain-text/HTML code paths are
  unchanged — `inline_images` is purely additive.

## Verification

| Metric                                 | Before    | After                              |
| --------------------------------------- | --------- | ----------------------------------- |
| Test suite                              | 313 passed | 318 passed (+5 new tests)          |
| `ruff check .`                          | clean     | clean                               |
| `ruff format --check .`                 | clean     | clean                               |
| `uv run pre-commit run -a`              | clean     | clean                               |
| MIME-structure inspection (via tests, see below) | N/A | confirmed `cid:` reference + `Content-ID` / `Content-Disposition: inline` |

No live SMTP send was attempted against a real or test mail server — this
repo's test suite at this commit has no GreenMail/SMTP-server end-to-end
harness (that arrived later, upstream). Verification is via the test suite's
message-structure assertions (`message.walk()` over the composed
`MIMEMultipart`, checked against real `aiosmtplib.SMTP` call arguments with
SMTP itself mocked) rather than an actual delivered email.

### Regression Tests Added (`tests/test_email_attachments.py::TestInlineImages`)

- `test_inline_image_alone_produces_related_structure`: `inline_images`
  alone produces a `multipart/related` message; the HTML part contains the
  `cid:` reference and the image part carries matching `Content-ID` and
  `Content-Disposition: inline` headers.
- `test_inline_image_with_attachments_nests_related_inside_mixed`:
  `inline_images` + `attachments` together produce
  `multipart/mixed` > `multipart/related`, with the regular attachment
  still present as a `Content-Disposition: attachment` sibling.
- `test_inline_images_with_html_false_raises_value_error`: `inline_images`
  with `html=False` raises `ValueError`.
- `test_inline_image_missing_file_raises_file_not_found`: a missing
  inline-image path raises `FileNotFoundError` via the reused
  `_validate_attachment`.
- `test_send_email_without_inline_images_or_attachments_is_unchanged`:
  regression — a plain send with neither `attachments` nor `inline_images`
  still produces a bare `MIMEText`.

Four existing tests (`test_classic_handler.py::test_send_email`,
`test_classic_handler.py::test_send_email_with_attachments`,
`test_mcp_tools.py::test_send_email`,
`test_save_to_mailbox.py::test_save_to_mailbox_tool_custom_folder`) had
their mock call-assertions updated to include the new
`inline_images=None` keyword argument that now flows through every layer
of the call chain; the underlying behaviour they assert is unchanged.

## Scope note

`save_to_mailbox` parity was implemented alongside `send_email` (not
deferred) since both share `compose_message`/`_build_message_body` — adding
the parameter to `save_to_mailbox`'s signature and threading it through was
low-marginal-cost once the shared composition logic supported it.

This fix was deliberately **not** proposed upstream to
`ai-zerolab/mcp-email-server`. Per the current internal-first policy, this
branch stays on NeatNerds' own fork (`github`/`origin` remotes) pending
Hugo's review; upstreaming is a separate, later decision.
